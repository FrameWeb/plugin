---
name: frameweb-state
description: >
  폼 디자이너 "상태 매트릭스" 탭(State Engine) 전용 스킬. 상태별로 화면의 각 칸을
  잠그거나 숨기는 규칙을 메타데이터로 선언하고 디버깅한다. 명령형 form_script
  (setReadonly/hide)를 대체하는 선언 방식. 저장은 form_property_config.state_matrix.
  Trigger: 상태 매트릭스, state-matrix, State Engine, 상태 규칙, 칸 잠금, 칸 숨김,
  form_property_config, state_matrix, 상태별 readonly, frameweb-state,
  StateMatrixTab, applyStateRules, 멀티 상태코드, 가장 제한적 우선, POPCOD 상태,
  발송 가드 제거, 그리드 컬럼 잠금.
user-invocable: true
---

# Form State -- 상태 매트릭스 탭 (State Engine)

폼의 상태(state)별로 화면의 각 칸을 잠그거나 숨기는 규칙을 메타데이터로 선언한다.
폼 디자이너 "상태 매트릭스" 탭이 편집하는 바로 그 대상이다.

## 위치 — 폼 디자이너 탭 스킬 6종 중 하나

이 스킬은 폼 디자이너 탭과 1:1로 맞는 작업 스킬 6종(`frameweb-canvas` / `frameweb-bindchain` /
`frameweb-binding` / `frameweb-script` / `frameweb-sandbox` / `frameweb-state`) 중 하나다.
6종은 `frameweb-` prefix로 통일되어 있어 포지(Forge, Cloud Dev AI 영역)로 이식할 때
폴더 이동만으로 옮길 수 있다. 설계 청사진은 `docs/specs/form-skill-tab-restructure.md`.

`frameweb-state`는 신규 스킬이다. 상태 매트릭스 탭은 그동안 담당 스킬이 없었다.

## 본질

- **메타데이터 선언 방식.** 디자이너가 "어떤 상태일 때 어떤 칸을 잠그거나 숨길지"를
  격자에서 선언하면, 런타임이 문서 상태값을 읽어 자동 적용한다.
- **명령형 form_script를 대체한다.** 기존에는 폼마다 `if (status) { field.setReadonly(true) }`
  식으로 화면에 직접 분기를 작성했다. State Engine은 같은 동작을 메타데이터 + 공통 엔진으로 옮긴다.
- **UI 표시 제어 전용이다 (ido 확정).** 화면에서 칸을 잠그거나 숨길 뿐, 서버 측 권한·검증은 별개다.
  잠금된 칸을 우회한 직접 API 호출까지 막지는 않는다. 서버 강제가 필요하면 별도 게이트를 둔다.
- 도메인 문서: `docs/architecture/domains/forms.md` "State Engine — 메타데이터 상태 제어" 절,
  `components.md` columnsKey 절. 마스터 설계는 `docs/specs/form-state-engine.md`.

## 저장 구조 — form_property_config.state_matrix (codes / byCode)

폼 메타 `form_property_config` JSONB 컬럼의 `state_matrix` 키에 저장한다.
멀티 상태코드 구조다.

```
state_matrix = {
  codes: ["SMSDOC", "SMSSTAT", ...],          // 등록 순서
  byCode: {
    SMSDOC:  { state_field?, new_state?, role_groups, rules },
    SMSSTAT: { state_field?, new_state?, role_groups, rules },
  }
}
```

- **codes** — 등록된 상태코드 리스트. 한 폼에 성격이 다른 여러 상태축을 동시에 둘 수 있다.
  예: SMSDOC(내부 품의 문서 흐름) + SMSSTAT(발송 결과 상태 흐름)이 한 화면에서 공존.
- **byCode[코드]** — 그 코드의 독립 규칙 세트.
  - `role_groups`: 상태별 지정 역할 묶음. key = stateId, 값 = `{ id, roles[], users[] }[]`.
  - `rules`: 칸 × columnKey 규칙. columnKey = `${stateId}::${groupId}` 또는 `${stateId}::__others__`.
  - `state_field`(선택): 현재 상태값을 읽을 칸. `new_state`(선택): NEW 시 시작 상태.
- **저장 호출**: `formsApi.update(frm_id, { form_property_config })`.
  기존 키(theme / component_overrides / chain_action_button 등)는 spread로 보존하고
  `state_matrix`만 갈아끼운다. 통째 교체하면 다른 폼 설정이 사라진다.
- 구버전 단일 구조(`{ state_field, role_groups, rules }`)는 `byCode`의 한 코드로 자동 마이그레이션된다
  (코드명 = `doc_stat || 'default'`). 단일 코드 폼은 그대로 동작한다.

UI 코드: `frontend/src/formDesigner/ui/StateMatrixTab.tsx`.
엔진: `frontend/src/formDesigner/stateEngine/` (types.ts, conditionEval.ts,
evaluateStateRules.ts, loadStatesFromPopcod.ts).
런타임 평가: `shared/packages/renderers/src/stateEngine/evaluateStateMatrix.ts`.

## 상태 코드 로드 — POPCOD

상태 코드는 공통코드(POPCOD)에서 로드한다 (`loadStatesFromPopcod`).

- 입력한 상태 코드(pop_id)로 현재 폼의 `bis_id` 회사 공통코드를 조회한다.
  호출: `GET /api/popcod/{pop_id}` + `X-Bis-Id: {현재 폼 bis_id}` 헤더.
- `use_yn=true`인 행만 `sort_order` 순으로 상태 목록(StateDef[])으로 변환한다.
  각 상태는 `{ id: fcd, label: nm || fcd }`.
- 여러 폼이 같은 상태 코드를 입력하면 같은 상태 셋을 공유한다.
- 코드에 등록된 상태가 0건이면 "공통코드에 먼저 등록하세요" 안내가 뜬다.
  이때는 상태 매트릭스를 만들 수 없으므로, POPCOD에 상태 코드부터 등록한다.

## 셀 4상태 순환

규칙표는 칸(행) × (상태, 역할 묶음)(열) 격자다. 셀을 클릭하면 다음 순서로 바뀐다.

| 표시 | 의미 | 저장값 |
|------|------|--------|
| 원래대로 (none) | 상태 처리를 하지 않고 폼 원래 값을 유지. 런타임 평가 제외 | 키 삭제(저장 안 함) |
| 입력가능 (editable) | visible: true, readonly: false | `editable` |
| 읽기전용 (readonly) | visible: true, readonly: true (잠금) | `readonly` |
| 숨기기 (hidden) | visible: false | `hidden` |

- 순환: 원래대로 → 입력가능 → 읽기전용 → 숨기기 → 다시 원래대로.
- "원래대로"는 저장 데이터에 넣지 않는다. 규칙을 건 칸만 `rules`에 담고, 규칙 없는 칸은
  정적 폼값을 그대로 유지한다 (회귀 0).
- 규칙표 행 = 사용자가 실제로 보고·누르고·입력하는 요소. 컨테이너(레이아웃 그릇)는 제외하고
  입력칸·버튼, 그리드 컬럼, 체인 액션 버튼 6종(열기/신규/저장/삭제/인쇄/도움말)을 포함한다.

## 합성 규칙 — 가장 제한적 우선

한 칸이 여러 상태코드에서 규칙을 받으면 `evaluateMultiStateMatrix`가 칸별로 합성한다.
충돌 시 더 제한적인 규칙이 이긴다.

- **visible = AND** — 모든 코드가 보여야 보인다. 한 코드라도 숨기면 숨겨진다.
- **readonly = OR** — 한 코드라도 잠그면 잠긴다.
- 제한 강도: 숨김 > 잠금 > 편집.
- 코드별로 단일 평가(`evaluateStateMatrix`)를 돌린 뒤 칸별로 누적 합성한다.
  코드 단위·칸 단위 try/catch로 격리되어 한 칸/한 코드 오류가 전체를 깨지 않는다.
- 규칙이 걸린 칸만 결과에 담는다. 규칙 없는 칸은 기본값(editable)으로 결과에 넣지 않는다.

같은 칸에 여러 코드의 규칙이 겹칠 때, 합성 결과가 의도와 맞는지 폼별로 확인한다.

## 그리드 컬럼 제어

- 칸 키 형식: `gridcol::{grid}::{col}`. 규칙표 행은 그리드 자체가 아니라 그 그리드의
  BindingSet 컬럼들을 펼쳐 만든다. 컬럼 라벨은 `라벨 (그리드.컬럼)` 형식으로 표시한다
  (두 그리드의 같은 컬럼명을 구분하기 위함).
- 그리드는 행마다 별개 레코드지만 편집은 커서가 있는 한 행에서 일어난다.
  **현재 커서 행 = 문서 1건**으로 보고, 행 선택 / 커서 이동 / 행 추가 시 그 행의 상태값을 읽어
  그 상태 규칙으로 컬럼을 제어한다 (FieldSet과 동일 개념).
- 잠금의 단일 소스 경로: BFR `mergedGridColumnMappings` → DFR `transformedColumns`(readOnly)
  → HandsontableRenderer `hotColumns`.
- 그리드 컬럼 조회 호출: `getGridColumns(frmId, gridFcmpId, bisId)`. 컬럼 0건/실패여도 화면은
  안 깨지고 그리드 컬럼 행만 비어 있다. "BindingSet 미연결일 수 있습니다" 안내가 뜨면
  그 그리드의 바인딩을 먼저 연결한다.

## 평가 시점

| 시점 | 평가 근거 |
|------|-----------|
| before open (OPEN/NEW 전) | 문서가 없어 상태값 없음. 컴포넌트 초기 속성(정적)이 담당. State Engine 미적용 |
| NEW (신규 문서) | 코드별 `new_state` (선택적 지정). 미설정 축은 NEW 시 적용 안 함 |
| OPEN / 그리드 행 선택 | 현재 문서·현재 커서 행의 상태값 |

평가 함수는 상태값을 인자로 받기만 한다. "어느 행/대상이 문서인가"는 값 공급자(폼스크립트)가 정한다.

## form_script 통로 — applyStateRules

런타임에서 코드별로 `form.applyStateRules(code, stateValue)`를 호출해 State Engine을 발동한다.

- 시그니처: `applyStateRules(code: string, stateValue: string | null): void`.
  상태코드와 현재 상태값을 넘기면 `state_matrix`를 평가해 화면에 적용한다.
- 호출 시점: NEW(`afterNew`) / OPEN(`afterOpen`) / 그리드 행 선택(`afterSelection`).
- 그리드는 `afterSelection`에서 커서 행의 상태값으로 `applyStateRules`를 재평가한다.
- 통로 파일 6개: `formScriptWorker` / `formScriptBridge` / `buildApiHandlers` /
  `ScriptContext` / `FormEngine` / `ScriptTab`.
- **명령형 가드를 규칙으로 옮긴다.** `if (status) { ... }` 식 발송 가드는 State Engine 규칙
  ("발송완료(SMSSTAT)면 발송 버튼 잠금")으로 대체한다. SMS400 발송 가드 제거가 첫 사례다.

## FRMMST 재배포 의무

상태 매트릭스를 `bisfrm`(`form_property_config`)에 저장해도 ERP·런타임 화면은
FRMMST 배포 스냅샷을 읽는다. **저장만으로는 ERP에 반영되지 않는다.**

- 규칙을 바꾼 뒤 `POST /api/forms/deploy/`로 FRMMST를 재배포한다 (`frameweb-deploy` 스킬 연계).
- 검증 전에 배포본이 최신인지 확인한다. 메모리 `form-deploy-required-after-bisfrm-edit`.

## Gotcha — columnsKey에 동적 속성 포함 의무

그리드 컬럼 잠금이 작동하지 않는 가장 흔한 원인이다.

- HandsontableRenderer는 `visibleColumns`를 `hotColumns` memo로 변환한다.
  이 memo의 변경 감지 key인 `columnsKey`에 **동적 속성(readOnly 등)을 반드시 포함**해야 한다.
- 누락하면 잠금 계산값이 맞아도 셀이 재렌더되지 않아 화면에 반영되지 않는다.
  State Engine 그리드 컬럼 잠금이 R0에서 미작동한 근본 원인이 이것이었다.
  로직 전 구간은 정상이었으나 `columnsKey`에 `readOnly`가 빠져 셀이 재렌더되지 않았다.
  `columnsKey`에 `readOnly` 추가 1줄로 해결했다.
- 일반화: 컬럼의 동적 속성(readOnly / visible 등)을 바꾸면 그 속성을 변경 감지 key에 반드시 넣는다.
  계산이 맞아도 memo key에서 빠지면 화면에 안 나타난다. 재발 가능성이 큰 gotcha다.
  상세는 `components.md` "동적 컬럼 속성은 변경 감지 key(columnsKey)에 반드시 포함" 절.

추가 주의:
- **SMS400은 BindingSet 모드(frmwrk 0건, wrk_id = bnd_id)**. 디자이너·런타임 둘 다 bndcmp 컬럼을 쓴다.
  WRKCMP는 0건이라 State Engine 칸 매핑과 무관하다.
- 묶음 id는 `g{N}` 카운터(useRef)로 생성한다. `Date.now()`를 쓰지 않는다
  (메모리 `temp-id-counter-vs-date-now`).

## 작업 절차

1. **상태 코드 확보** — 제어하려는 상태축의 POPCOD 코드를 정한다 (예: SMSDOC). 코드에 등록된
   상태가 없으면 공통코드에 먼저 등록한다.
2. **탭 진입** — 폼 디자이너에서 대상 폼을 열고 "상태 매트릭스" 탭으로 간다. 폼에 칸(컴포넌트)이
   있어야 규칙을 편집할 수 있다.
3. **코드 등록** — 좌측 "상태 코드 추가"에 코드를 입력해 불러온다. POPCOD에서 상태 목록이 채워진다.
   여러 축이 필요하면 코드를 추가로 등록한다.
4. **역할 묶음 정의** (필요 시) — 상태별로 지정 역할 묶음([ADMIN] 등)과 사용자 태그를 만든다.
   "외 모두" 열은 항상 자동 표시된다. 역할 무관 규칙이면 묶음 없이 "외 모두"만 쓴다.
5. **규칙 격자 편집** — 칸 × (상태, 묶음) 격자에서 셀을 클릭해 입력가능 / 읽기전용 / 숨기기를 지정한다.
   그리드 컬럼은 별도 행으로 펼쳐진다.
6. **상태 칸 / 신규 상태 지정** (선택) — `state_field`로 상태값을 읽을 칸을, `new_state`로 NEW 시
   시작 상태를 지정한다.
7. **저장** — "규칙 저장"으로 `form_property_config.state_matrix`에 멀티코드 전체를 한 번에 저장한다.
8. **재배포** — `POST /api/forms/deploy/`로 FRMMST를 재배포한다 (`frameweb-deploy` 연계).
9. **런타임 검증** — `frameweb-verify` 연계 또는 모리 UI 검증으로 실제 화면에서 잠금/숨김을 확인한다.

## 검증 방법

저장 직후 디버깅은 다음 순서로 한다 (코드/DB 조회 도구가 있을 때).

1. **저장 구조 확인** — `bisfrm.form_property_config`의 `state_matrix.codes`와 `byCode`가
   의도대로 들어갔는지 확인.
   ```sql
   SELECT form_property_config->'state_matrix'
   FROM bisfrm WHERE frm_id = '{frm_id}' AND bis_id = '{bis_id}';
   ```
2. **POPCOD 상태 확인** — 상태 코드에 `use_yn=true` 행이 있는지 확인.
   ```
   GET /api/popcod/{pop_id}   (헤더 X-Bis-Id: {bis_id})
   ```
3. **배포 반영 확인** — FRMMST 스냅샷이 최신 `state_matrix`를 담고 있는지 확인. 저장만 하고
   재배포를 빠뜨리면 ERP 화면은 옛 규칙을 쓴다.
4. **런타임 동작 확인 (모리 / `frameweb-verify`)** — ERP에서 폼을 열고:
   - 지정한 상태값일 때 해당 칸이 읽기전용/숨김으로 바뀌는지.
   - 그리드 행 선택 시 그 행 상태에 맞춰 컬럼 잠금이 바뀌는지 (커서 행 = 문서).
   - 여러 코드가 겹친 칸이 가장 제한적 규칙(숨김 > 잠금 > 편집)으로 합성되는지.
   - 규칙을 안 건 칸은 정적 폼값 그대로인지 (회귀 0).
   - 그리드 컬럼 잠금이 계산은 맞는데 화면에 안 나타나면 columnsKey Gotcha를 의심한다.

## 참조

- `docs/architecture/domains/forms.md` — "State Engine — 메타데이터 상태 제어" 절
- `docs/architecture/domains/components.md` — columnsKey 동적 속성 절
- `docs/specs/form-state-engine.md` — 마스터 설계
- `frameweb-deploy` 스킬 — FRMMST 재배포
- `frameweb-verify` 스킬 — 런타임 검증 연계
