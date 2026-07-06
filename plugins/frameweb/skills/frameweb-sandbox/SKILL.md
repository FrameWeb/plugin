---
name: frameweb-sandbox
description: >
  폼 라이프사이클 보조: 폼 전용 AI 도우미(Sandbox) 작성/수정. 특정 폼에 붙어
  사용자 채팅에 응답하는 AI 에이전트의 작동 규칙(system_prompt + 도구 권한)을 설계하고
  비즈 DB의 agent 테이블에 적재한다. 폼 스크립트(form_script)와는 다른 영역.
  Trigger: frameweb-sandbox, 폼 sandbox, 샌드박스 작성, 폼 AI 도우미, 폼 에이전트,
  폼 전용 AI, agent.config.sandbox, form_assistant, 폼 챗 도우미, SandboxTab,
  이 폼에 AI 붙여줘, 번역 도우미, 폼 어시스턴트.
user-invocable: true
---

# Form Sandbox -- 폼 전용 AI 도우미 작성

특정 폼에 붙는 AI 도우미를 작성한다. 사용자가 ERP 화면에서 그 폼을 열고 채팅으로 말을 걸면,
이 도우미가 주어진 도구(DB 조회/저장 등)로 응답한다. 폼 디자이너의 "Sandbox" 탭이 편집하는
바로 그 대상이다.

## 도메인 지식

- **폼 sandbox = 폼 전용 AI 에이전트.** 폼 스크립트(`bisfrm.form_script`, Web Worker 자동화 코드)와
  절대 혼동하지 않는다. sandbox는 자연어 규칙 + 도구 권한이고, form_script는 자바스크립트 코드다.
  사용자가 "샌드박스"라 하면 거의 항상 이 도우미. "스크립트"라 하면 form_script.
- **저장 위치: 비즈 DB의 `public.agent` 테이블** (시스템 DB의 `forge.agent`가 아니다).
  bisdb로 라우팅된 비즈 DB 안의 row가 단일 진실 소스다. 예: bis_id=bnk → 비즈 DB `bnk`의 `agent`.
- **agent_id 규칙: `form:{bis_id}:{frm_id}`** (예: `form:winform7e7d:I18N100`).
- **실행 경로**: ERP에서 폼 열고 AI 챗 → `CoworkOrchestrator(agent_id=form:...)` →
  `_load_agent_sandbox()`가 `SELECT config FROM agent WHERE agent_id=:id AND enabled=TRUE`로 로드 →
  `config.sandbox.system_prompt`를 최상단에 주입 + `filter_tools_by_sandbox`로 도구 제한.
  (코드: `backend/services/cowork_orchestrator.py`, `backend/services/cowork_tools.py`)
- **UI**: `frontend/src/formDesigner/ui/SandboxTab.tsx` (저장 EP: `PUT /api/agent/{id}`, root-level config merge).
- 참고 메모리: `form-designer-sandbox-tab`.

## 사용 가능한 도구 (sandbox에서 켜고 끄는 것)

| 도구 | 동작 | 비고 |
|------|------|------|
| `db_query` | 비즈 DB SELECT 전용 | `allowed_tables` 화이트리스트로 테이블 제한 |
| `db_save`  | 비즈 DB INSERT/UPDATE/DELETE | DDL 불가. `allowed_tables` 화이트리스트 |
| `fill_form`| 화면 폼 필드에 값 입력 (프론트가 SSE로 처리) | 단일 레코드/현재 행 채우기. **다중 행 그리드 일괄에는 부적합** |
| `run_skill`| 등록된 스킬(파이프라인) 실행 | |

**도구 선택 기준** (이번 사례에서 확인):
- 단일 레코드 폼(입력 필드 위주, 예: SMS 템플릿) → `fill_form`로 화면 즉시 반영.
- 그리드 매트릭스 / 대량 행 처리(예: 다국어 번역표) → `db_query` + `db_save`로 DB 직접 작업.
  이 경우 화면 그리드는 자동 갱신되지 않으니, 사용자에게 "재조회(키 가져오기)" 안내를 규칙에 넣는다.

## 작업 절차

### 1. 대상 폼 + 비즈 확인

```sql
-- 폼이 어느 비즈 소속인지 (시스템 DB)
SELECT bis_id, frm_id, frm_nm FROM bisfrm WHERE frm_id='{frm_id}';
```

폼의 컴포넌트/BindingSet 구조와 도우미가 다룰 데이터 테이블의 컬럼을 `fn_db_info()`로 파악한다.

### 2. agent 테이블 존재 확인 (없으면 생성)

비즈마다 `agent` 테이블이 다 깔려 있지는 않다. 없으면 도우미를 저장할 수 없다(로드 시 ValueError).

```sql
-- 대상 비즈 DB에서
SELECT to_regclass('public.agent');
```

없으면, agent 테이블이 있는 다른 비즈 DB에서 구조를 그대로 가져와 생성한다:

```bash
PGPASSWORD=... pg_dump -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {참고비즈} \
  -t public.agent --schema-only --no-owner --no-privileges
```

추출한 DDL(`\restrict`/`\unrestrict` 라인 제거)을 대상 비즈 DB에 한 트랜잭션으로 적용한다.
agent 테이블은 self-FK(`reports_to → agent`)만 있어 단독 생성된다. 19 컬럼 + PK + 인덱스 3개.

### 3. config 구조

`config`는 `{model, bis_id, frm_id, sandbox}`. 기본값은 `frontend/src/types/agent.types.ts`의
`DEFAULT_AGENT_CONFIG` / `DEFAULT_AGENT_SANDBOX`를 따른다.

```jsonc
{
  "model": "claude-sonnet-4-6",
  "bis_id": "{bis_id}",
  "frm_id": "{frm_id}",
  "sandbox": {
    "system_prompt": "...(도우미 작동 규칙 본문)...",
    "description": "한 줄 설명",
    "context_bindings": null,            // BindingSet ID 배열. DB 직접 조회형이면 null
    "tools": {
      "db_query":  {"enabled": true,  "allowed_tables": ["대상테이블"]},
      "db_save":   {"enabled": true,  "allowed_tables": ["대상테이블"]},
      "fill_form": {"enabled": false},
      "run_skill": {"enabled": false}
    },
    "max_turns": 20
  }
}
```

agent row 컬럼: `agent_id`, `name`, `role='form_assistant'`, `bis_id`, `kind='form'`,
`frm_id`, `enabled=true`, `config(jsonb)`, `status='idle'`.

### 4. system_prompt 설계

도우미 품질은 system_prompt가 좌우한다. 데이터를 직접 수정하는 도우미(db_save)는 잘못된 규칙이
원본 데이터를 훼손할 수 있으니 충실히 설계한다.

- **기존 검증된 도우미를 참고 모델로 삼는다.** 같은 플랫폼의 다른 폼 도우미를 조회:
  `SELECT agent_id, jsonb_pretty(config->'sandbox') FROM agent WHERE kind='form' LIMIT 5;`
  (예: `form:bnk:SMS200` — 4언어 번역 폼 도우미.)
- **안전 규칙을 명시적으로 박는다** (DB 제약이 아니라 프롬프트로만 막히는 것):
  읽기 전용 데이터 보호 / PK·키 변경 금지 / 플레이스홀더·변수 토큰 보존 /
  허용 테이블·코드값 화이트리스트 / 저장 전 자가 점검 절차 / 대량 처리 분할.
- **위험이 큰 도우미는 다관점 설계를 권장**한다 (안전성 / 사용자 경험 / 도메인 정확성 렌즈로
  각각 초안 → 적대적 검증 → 합성). ultracode/대형 작업일 때 Workflow로 fan-out.
- 응답은 사용자 언어(한국어). SQL 본문을 사용자에게 노출하지 않도록 규칙에 넣는다.

### 5. 적재 (base64 주입 패턴)

system_prompt에 따옴표·줄바꿈·특수문자가 많으면 psql 직접 INSERT는 깨지기 쉽다.
config를 JSON으로 만들어 base64로 인코딩한 뒤 주입한다 (메모리 `form-designer-sandbox-tab`):

```python
import json, base64, html
sp = html.unescape(raw_system_prompt)   # JSON 경유 시 &lt; &gt; 디코딩
config = { ... }                        # 위 3번 구조
b64 = base64.b64encode(json.dumps(config, ensure_ascii=False).encode('utf-8')).decode()
```

```sql
INSERT INTO public.agent (agent_id, name, role, bis_id, kind, frm_id, enabled, config, status)
VALUES ('form:{bis}:{frm}', '{name}', 'form_assistant', '{bis}', 'form', '{frm}', true,
        convert_from(decode('<b64>','base64'),'UTF8')::jsonb, 'idle')
ON CONFLICT (agent_id) DO UPDATE SET config=EXCLUDED.config, name=EXCLUDED.name,
        enabled=EXCLUDED.enabled, updated_at=now();
```

> `decode(b64)::text::jsonb`는 bytea escape로 깨진다. 반드시 `convert_from(..., 'UTF8')::jsonb`.

### 6. 자체 검증

```sql
-- 백엔드 로드 쿼리 그대로 — enabled=TRUE 로드 성공?
SELECT agent_id, name, enabled FROM agent WHERE agent_id='form:{bis}:{frm}' AND enabled=TRUE;
-- 도구 권한 + 핵심 규칙 grep
SELECT config->'sandbox'->'tools', config->'sandbox'->>'max_turns' FROM agent WHERE agent_id='...';
SELECT (config->'sandbox'->>'system_prompt') LIKE '%핵심규칙키워드%' FROM agent WHERE agent_id='...';
```

### 7. UI 검증 위임 (모리)

도우미 정의 적재는 자체 검증으로 끝나지만, 실제 작동(ERP에서 폼 열고 챗 → 도구 호출 → 데이터 반영)은
모리에게 의뢰한다. 데이터를 수정하는 도우미면 **원본 데이터 보존 확인 항목을 반드시 포함**한다.
`admin-test-request` 또는 `frameweb-verify` 스킬 사용.

## Gotcha

- **agent 테이블 미설치 비즈**: 도우미 저장 전 `to_regclass('public.agent')` 확인. 없으면 DDL 먼저.
- **fill_form은 다중 행 그리드 일괄에 부적합**: 화면의 단일 필드/현재 행 key=value 채우기다.
  그리드 매트릭스/대량 처리는 db_query+db_save로 DB 직접 작업.
- **db_save 도우미의 원본 훼손 위험**: `INSERT ... ON CONFLICT DO UPDATE`는 충돌 시 UPDATE처럼
  동작하므로, 보호 대상 행(예: 원문)의 키가 VALUES에 섞이면 그 행이 덮어쓰기된다.
  "INSERT라 안전" 가정은 틀리다. system_prompt에 명시적으로 차단 규칙을 넣는다.
- **접두사·구조 추측 금지**: 데이터 키 패턴(접두사 등)은 도우미 규칙에 하드코딩하지 말고,
  db_query로 실제 값을 조사해 범위를 정하도록 설계한다 (잘못된 가정이 침묵 누락을 만든다).
- **"비어있음" 정의**: 행 부재 + NULL + 빈 문자열 3가지. `NOT EXISTS` 단독은 NULL/빈값을 놓친다.
