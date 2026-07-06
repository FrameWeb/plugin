---
name: frameweb-binding
description: >
  폼 디자이너 Binding 탭(Model/Pull/Push/Query/Save) 담당. bndquery(조회·입력 SQL),
  bndpull(조회 파라미터), bndsave(저장 매핑), bndcmp(컬럼 매핑)의 생성과 디버깅을 모두 다룬다.
  Query 하위(SELECT/INSERT/UPDATE/DELETE/GET/SEND) 작성, GET 파라미터 매핑, SET 저장 매핑, 입력보조(bispop) 설정.
  Trigger: 폼 바인딩, Binding 탭, bndquery, bndpull, bndsave, bndcmp, bispop,
  조회 SQL, CRUD SQL, GET 파라미터, SET 매핑, 입력보조, 코드 드롭다운, 팝업 검색,
  Open 에러, Save 에러, 500 에러, 검색 안 됨, 0행 저장, frameweb-binding.
user-invocable: true
---

# Form Binding — 폼 디자이너 Binding 탭 (Model/Pull/Push/Query/Save)

폼 디자이너의 Binding 탭을 담당한다. bndquery·bndpull·bndsave·bndcmp 네 테이블로 폼의 데이터 조회·입력·저장을 정의하고, 동작하지 않는 폼을 디버깅한다. 생성(새 바인딩 정의)과 디버깅(작동 안 하는 바인딩 보정)을 한 스킬 안에서 다룬다.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

이 스킬이 손대는 대상은 그중 네 테이블이다.

| 테이블 | Binding 탭 하위 | 역할 |
|--------|----------------|------|
| bndquery | Query | 조회(R)·입력(C/U/D)·외부 API(G/S) SQL/요청 |
| bndpull | Pull | R SQL의 `:파라미터`에 들어갈 입력 소스 매핑 |
| bndsave | Push(Save) | C/U/D SQL의 `:파라미터`에 들어갈 저장 소스 매핑 |
| bndcmp | Model | 그리드/필드 컬럼과 DB 컬럼(data_field) 매핑 + 입력보조 |

frmbnd(바인딩 체인 마스터, bound_fcmp_ids·open_trigger·open_sq)는 BindChain 탭 소관으로 `frameweb-bindchain` 스킬이 담당한다. 이 스킬은 frmbnd 행이 존재한다는 전제 위에서 그 안의 SQL·파라미터·컬럼을 채운다.

## 폼 디자이너 탭 스킬 묶음에서의 위치

이 스킬은 폼 디자이너 탭과 1:1로 맞춘 탭 작업 스킬 6종(frameweb-canvas / frameweb-bindchain / frameweb-binding / frameweb-script / frameweb-sandbox / frameweb-state) 중 하나다. 6종은 포지(Forge, Cloud Dev AI 영역) 이식 대상으로, `form-` prefix를 유지해 폴더 단위로 옮길 수 있게 묶여 있다. 설계 청사진은 `docs/specs/form-skill-tab-restructure.md`.

탭 간 책임 경계:

- 컴포넌트 배치·레이아웃(FRMCMP) → `frameweb-canvas`
- 바인딩 체인 연결(frmbnd, bound_fcmp_ids, Observer Chain) → `frameweb-bindchain`
- **조회·입력 SQL·파라미터·컬럼 매핑(bndquery/bndpull/bndsave/bndcmp) → 이 스킬**
- 이벤트 핸들러(form_script) → `frameweb-script`
- 폼 전용 AI 도우미 규칙 → `frameweb-sandbox`
- 상태 매트릭스(state_matrix) → `frameweb-state`

## 라이프사이클 위치

```
frameweb-require → frameweb-canvas → frameweb-bindchain → frameweb-binding → frameweb-load → frameweb-deploy → frameweb-verify
```

런타임 동작 검증(폼 열기·CRUD 테스트)은 탭 스킬로 쪼개지 않고 `frameweb-verify`(모리 런타임 검증)로 흡수한다. 이 스킬의 디버깅을 마치면 `frameweb-verify`로 연계한다.

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 비즈니스 DB | bisdb 테이블에서 host/port/database 조회 (`SELECT database FROM bisdb WHERE bis_id='{bis_id}' AND is_active='Y'`) |
| ERP Preview | ${FRAMEWEB_APP_URL}/erp/form/{폼ID}?bis_id={bis_id} |
| 모리 공유 폴더 | /mnt/z/Obsidian/browser-test/request/ |
| 접속 정보 단일 소스 | ~/.claude/memory/db-connections.md |

> bndquery/bndpull/bndsave/bndcmp는 모두 시스템 DB(${FRAMEWEB_DB_NAME})에 저장된다. R/C/U/D SQL이 실제로 실행되는 곳은 bis_id에 매핑된 비즈니스 DB다. SQL을 직접 테스트할 때는 비즈니스 DB로 접속한다.

---

## 호출

```
/frameweb-binding {폼ID} {bis_id}
```

## 작업 자동 생성 (TaskCreate)

`/frameweb-binding {폼ID} {bis_id}` 호출 시 이 스킬 영역의 작업 3개를 TaskCreate로 자동 생성한다. Binding 탭 영역(SQL·파라미터·매핑)에 해당하는 작업만 가져온다. frmbnd 보정은 `frameweb-bindchain`, Open/CRUD 런타임 테스트는 `frameweb-verify` 소관이므로 이 스킬에서 생성하지 않는다.

각 Task는 `subject`(제목) + `description`(절차/쿼리/완료조건)을 포함한다. 절차 본문은 이 스킬의 Query/Pull/Save/Model 절을 단일 소스로 인용한다.

### 자동 생성하는 3개 Task

```
TaskCreate:
  subject: "[{폼ID}] SQL 확인/작성 (bndquery)"
  description: |
    Query 절(BNDQUERY)의 "디버깅 — SQL 확인/작성" 절차를 따른다.
    1. 현재 R/C/U/D SQL 전문 조회 (Python — psql 80자 컷 회피)
    2. PostgreSQL 문법 검증 (선택적 조건 {{ }} / 타입 캐스팅 / _old / 세션변수)
    3. 없는 SQL 작성 — new_yn='Y' 또는 save_sq > 0 인 바인딩만
    4. {{AND}} 제거 후 비즈니스 DB에서 직접 실행 테스트
    완료 조건: 모든 바인딩 R SQL 존재 + 문법 통과, CUD 대상 C/U/D SQL 존재 + 실행 검증.

TaskCreate:
  subject: "[{폼ID}] GET 파라미터 (bndpull)"
  description: |
    Pull 절(BNDPULL)의 "디버깅 — GET 파라미터 점검" 절차를 따른다.
    선행: SQL 확인 Task 완료 (R SQL :파라미터가 확정돼야 매핑 가능).
    1. R SQL에서 :파라미터 추출
    2. 소스 결정 (source_type 6종 표 — 검색입력=COMPONENT, 다른 그리드=GRID, 세션=SESSION)
    3. bndpull INSERT (ON CONFLICT DO NOTHING)
    완료 조건: 모든 R SQL :파라미터에 대응하는 bndpull 존재, GRID source_column 전부 채워짐.

TaskCreate:
  subject: "[{폼ID}] SET 매핑 + 입력보조 (bndsave + bispop)"
  description: |
    Push/Save 절(BNDSAVE)의 "디버깅 — SET 매핑 점검" + 입력보조(bispop) 절차를 따른다.
    선행: SQL 확인 Task 완료 (C/U/D SQL :파라미터가 확정돼야 매핑 가능).
    1. CUD SQL에서 :파라미터 추출 (typecast/세션변수 제외)
    2. 매핑 규칙 표 적용 (일반=GRID 자기 bnd_id, _old, SESSION=꺾쇠 형식, 하위 FK=마스터 참조)
    3. bndsave INSERT + 하위 Grid PK/FK 자체참조 확인
    4. CODE/LOOKUP/POPUP 컬럼이 있으면 입력보조(bispop) 설정 — bis_id 단위, 없으면 EPIC_SELECT 403
    완료 조건: CUD 바인딩 전체 파라미터 매핑 완료, 미매핑 필드 없음, 하위 FK 마스터 참조, 팝업 설정 완료.
```

> Model(bndcmp) 보정은 Query/Pull/Save 작업 안에서 함께 점검한다(R 별칭 정합, 입력보조 연결). 별도 Task로 분리하지 않는다 — 컬럼 매핑은 SQL·파라미터 작업과 분리해서 검증할 수 없다.

### 작업 간 관계

```
SQL 확인/작성(bndquery) → GET 파라미터(bndpull)
                       → SET 매핑(bndsave + bispop)
                              |
                         frameweb-verify (Open/CRUD 런타임 테스트)
```

bndpull(GET 파라미터)과 bndsave(SET 매핑)는 둘 다 SQL 확인 작업의 산출물(:파라미터)에 의존하므로 SQL 작업 뒤에 둔다. 둘 사이에는 의존이 없어 순서는 무관하다. 세 작업을 마치면 `frameweb-verify`로 런타임 동작을 검증한다.

### 다오 직접 vs 모리 위임 (이 스킬 해당분)

| 작업 | 실행자 | 이유 |
|------|--------|------|
| SQL 확인/작성 (bndquery) | **다오** | psql 직접 접근 + 비즈니스 DB 실행 테스트 가능 |
| GET 파라미터 (bndpull) | **다오** | DB INSERT/조회 |
| SET 매핑 + 입력보조 (bndsave + bispop) | **다오** | DB INSERT/조회 |
| Open/CRUD 런타임 테스트 | **모리** | Chrome/UI 필요 → `frameweb-verify`가 의뢰서 생성 |

이 스킬의 3개 작업은 전부 다오 직접 영역(DB 작업)이다. 모리 위임은 디버깅 완료 후 `frameweb-verify`로 넘어간 뒤에 일어난다.

### 실행 예시

`/frameweb-binding COD100 PACKAGE` 입력 시:
1. 3개 TaskCreate 실행 (subject에 `[COD100]` 포함)
2. SQL 확인/작성(bndquery)부터 진행 — 현재 SQL 조회 → 문법 검증 → 작성 → 실행 테스트
3. GET 파라미터(bndpull) + SET 매핑(bndsave + bispop) 진행 — DB 작업 전부 다오 직접
4. 세 작업 완료 후 `/frameweb-verify COD100 {version} PACKAGE`로 Open/CRUD 런타임 검증 (version 미지정 시 최신 배포본, 모리 위임)
5. 검증 실패 시 실패 항목에 따라 해당 작업으로 복귀 (SQL 에러→bndquery, 파라미터 에러→bndpull, Save 에러→bndsave)

## 책임 경계 — Binding 탭 하위 5종

폼 디자이너 Binding 탭은 Model/Pull/Push/Query/Save 하위로 구성된다. 이 스킬이 다루는 대상과 테이블 대응은 다음과 같다.

| 하위 | 대상 | 작업 |
|------|------|------|
| Model | bndcmp | 그리드/필드 컬럼 ↔ DB 컬럼(data_field) 매핑, 입력보조 연결 |
| Query | bndquery | R(조회) / C·U·D(입력) SQL, G(GET)·S(SEND) 외부 API |
| Pull | bndpull | R SQL `:파라미터`의 입력 소스(COMPONENT/GRID/SESSION/CONST) |
| Push / Save | bndsave | C/U/D SQL `:파라미터`의 저장 소스 매핑 |

Query 하위는 SELECT/INSERT/UPDATE/DELETE/GET/SEND 단계로 더 나뉜다. 1차 버전(v0)에서는 이 단계들을 한 스킬 안에서 함께 다룬다. 더 잘게 나눌지는 9절 "후속 과제"로 둔다.

---

## Query — BNDQUERY (SQL/API 쿼리)

> **source_type 은 'SQL' 또는 'API' 둘 다 표준 분기.** 외부/내부 API 호출 위젯은 `bnd-design` 스킬 §6.5 절 참조. backend 분기는 `binding_executor.py` L397~418 (`source_type == 'API'` → `ApiExecutor.execute_get` → bnd_type 별 응답 정규화). **crudm 값은 SQL(R/C/U/D) 과 API(G/S) 가 분리** — API source_type 이면 crudm='G'(GET) 또는 'S'(SEND) 포함할 것. 'R' 박으면 폼 디자이너 UI 가 API 설정 인식 못 함.

### CRUD 패턴 (SQL)

```sql
-- R (조회) -- source_type='SQL' 일 때
INSERT INTO bndquery (..., crudm, execution_order, source_type, query, description, ...)
VALUES (..., 'R', 1, 'SQL', 'SELECT ... FROM ... WHERE ... ORDER BY ...', '조회', ...);

-- C (등록)
VALUES (..., 'C', 1, 'SQL', 'INSERT INTO table (col1, col2, ...) VALUES (:col1, :col2, ...)', '등록', ...);

-- U (수정) -- WHERE에 _old 파라미터
VALUES (..., 'U', 1, 'SQL', 'UPDATE table SET col2 = :col2 WHERE col1 = :col1_old', '수정', ...);

-- D (삭제) -- _old 없이 원본 필드명
VALUES (..., 'D', 1, 'SQL', 'DELETE FROM table WHERE col1 = :col1', '삭제', ...);
```

### _old 파라미터 규칙

| SQL | WHERE 절 | 이유 |
|-----|---------|------|
| UPDATE | `:pk_old` 사용 | SET에 새 값 + WHERE에 원본값, 구분 필요 |
| DELETE | `:pk` 사용 (_old 없음) | V5가 원본값을 원본 필드명으로 전송 |
| 공통 | `_old`는 식별 전용 | PK 값 자체를 바꾸는 용도가 아니다. PK를 변경하려면 아래 'PK 변경 + CTE cascade' 절 참조 |

> `_old` 생성 주체와 폼 유형 차이:
> - FieldSet(단건): `is_pk=true` 컬럼은 프론트(`worksetTrackingStore.buildFieldSetParams`)가 원본값을 추적해 `{pk}_old`를 자동 첨부한다. BNDSAVE에 미등록이어도 backend resolver가 통과시키지만, 명시 등록이 더 안전하다.
> - GridSet(그리드): 체인 SAVE 경로(SaveRegistry getData())는 `is_pk`와 무관하게 `_old`를 자동 생성하지 않는다. 원본 키는 R 쿼리 별칭 컬럼으로만 전달된다.
> - 어느 경우든 UPDATE 쿼리의 `WHERE ... = :pk_old`는 손으로 작성해야 한다(`:pk`만 쓰면 새 값으로 잡혀 0행).

### 삭제 연쇄 (FK 순서)

```sql
-- D execution_order 1: 자식 먼저 삭제
VALUES (..., 'D', 1, 'SQL', 'DELETE FROM child WHERE fk = :pk', ...);
-- D execution_order 2: 본체 삭제
VALUES (..., 'D', 2, 'SQL', 'DELETE FROM parent WHERE pk = :pk', ...);
```

### PK 변경 + CTE cascade

사용자가 PK 컬럼(코드값·메뉴코드 등) 자체를 바꿀 수 있어야 하는 폼의 표준이다. PK 변경 미지원 폼이면 PK 컬럼을 `bndcmp` readOnly로 두어 편집을 막는 것이 안전한 기본값이다.

메커니즘:
- backend는 `_old`를 같은 행의 현재값에서만 채운다(`binding_executor.py` `_ensure_bind_params`). 따라서 그리드에서 PK를 새 값으로 바꾸면 `_old`도 새 값이 되어 `WHERE pk = :pk_old`가 0행 매칭 → commit + success:True + affected_rows=0(조용한 실패).
- 원본 키를 보존하려면 R 쿼리가 PK를 별칭 컬럼으로 한 번 더 내보낸다(예: `menu_cd AS origin, menu_cd`). 이 별칭 컬럼은 `bndcmp`에 `visible=false`로 숨겨도 행 데이터에 실려 저장 payload에 포함된다. BNDSAVE 매핑이 없어도 backend resolver가 행의 별칭 값을 그대로 통과시킨다(`bnd_param_resolver.py`).

cascade(자기참조 부모코드·권한 등)는 PostgreSQL CTE 한 statement로 묶는다. C/U/D 조회는 `_get_bndquery`가 `.first()`를 써서 execution_order 첫 행만 실행하므로(`binding_executor.py`), cascade를 별도 execution_order 행으로 나누면 첫 행만 실행되고 나머지는 조용히 건너뛴다(silent skip). PostgreSQL 제약: 같은 statement 안에서 같은 테이블의 같은 행을 두 번 수정할 수 없다(다른 행은 가능).

```sql
-- MNU100: menu_cd 편집 + 권한(mnupms) + 자식 부모코드(up_cd) cascade
-- 본 행(부모) UPDATE를 메인(마지막) 문장에 둔다 — affected_rows가 부모 기준(1)이 되게.
WITH cascade_perms AS (
  UPDATE mnupms SET menu_cd = :menu_cd
   WHERE system_cd = :system_cd AND menu_cd = :origin
  RETURNING 1
),
cascade_child AS (
  UPDATE mnumst SET up_cd = :menu_cd
   WHERE system_cd = :system_cd AND up_cd = :origin    -- 자식 메뉴
  RETURNING 1
)
UPDATE mnumst
   SET menu_cd = :menu_cd, up_cd = :up_cd, title = :title,
       use_yn = :use_yn, hid_yn = :hid_yn
 WHERE system_cd = :system_cd AND menu_cd = :origin     -- 본 행(부모), 원본 키 보존
```

> **저장 건수(affected_rows) 함정**: 응답 건수는 CTE의 마지막 문장(메인 쿼리) 영향 행 수만 센다. 본 행(부모) UPDATE를 마지막 메인 문장에 두라. cascade(자식·권한)를 마지막에 두면, 하위가 없는 행을 변경할 때 마지막 문장이 0행이 되어 `affected_rows=0`으로 응답한다 — 본 행은 실제로 바뀌었는데도 화면에 "저장 안 됨(0건)"으로 떠 오해를 부른다(MNU100 leaf 메뉴 사례).

설계 절차의 단일 소스는 본 절이며, 도메인 문서 `docs/architecture/domains/forms.md`의 `:origin` 절도 같은 내용을 다룬다.

### 쿼리 작성 규칙

| 규칙 | 설명 |
|------|------|
| 바인드 변수 | `:column_name` 형식 |
| 선택적 조건 | `{{AND col = :param}}` (null이면 자동 스킵) |
| 타입 캐스팅 | `:param::timestamp`, `:param::integer` |
| ORDER BY | R 쿼리에 반드시 포함 |
| NOW() | 타임스탬프 컬럼 (cdt, mdt, created_at) |
| ON CONFLICT | UPSERT: `INSERT ... ON CONFLICT (pk) DO UPDATE SET ...` |
| MSSQL 변환 | references/sql_conversion_rules.md 참조 |

> 날짜 형변환은 `:param::timestamp`보다 `CAST(:param AS timestamp)`를 쓴다 (조건부 SQL `{{ }}` 파서가 `::type`을 가짜 파라미터로 오인할 수 있음).

### 빈 그리드 패턴 — placeholder/업로드 전용 그리드

DB 조회 없이 **form_script로만 데이터를 채우는 그리드**(CSV 업로드, AI 응답 표시 등)는 컬럼 정의가 frontend에 도달해야 헤더가 노출된다. 윈폼의 "오픈으로 초기화하지 않으면 그리드가 먹통" 경험과 같은 본질 — 컬럼 정의가 데이터보다 먼저다.

**해결 — bndquery에 빈 SELECT를 포함한다.** `WHERE 1=0`으로 0행 반환 + 컬럼 메타만 전달.

```sql
-- frmbnd: OPEN 트리거 자동 실행
INSERT INTO frmbnd (bis_id, frm_id, bnd_id, bnd_type, bound_fcmp_ids, open_trigger, open_sq, ...)
VALUES ('bnk', 'SMS100', 'ws_recipients', 'Grid', '["grid_recipients"]'::jsonb, 'OPEN', 1, ...);

-- bndquery: 빈 SELECT (컬럼 정의 + 0행)
INSERT INTO bndquery (bis_id, frm_id, bnd_id, crudm, execution_order, query, c_id)
VALUES ('bnk', 'SMS100', 'ws_recipients', 'R', 1,
  'SELECT ''''::varchar AS phone, ''''::varchar AS recipient, ''''::varchar AS language WHERE 1=0',
  1);

-- bndcmp: 그리드 컬럼 매핑 (label/width/align)
INSERT INTO bndcmp (bis_id, frm_id, bnd_id, fcmp_id, data_field, data_type, col_seq, label, ...) VALUES
('bnk', 'SMS100', 'ws_recipients', 'col_phone',     'phone',     'string', 1, '수신번호', ...),
('bnk', 'SMS100', 'ws_recipients', 'col_recipient', 'recipient', 'string', 2, '수신자명', ...),
('bnk', 'SMS100', 'ws_recipients', 'col_language',  'language',  'string', 3, '언어',     ...);
```

**form_script에서 데이터 채우기**:
```javascript
registerNativeFunction('onUploadCsv', () => {
  // ... CSV 파싱 후
  await grid.loadData('ws_recipients', rows);  // 컬럼 정의는 OPEN에서 이미 포함됨
});
```

**원칙**:
- 컬럼 타입 캐스팅 명시: `''::varchar AS phone` (NULL 타입 추론 회피)
- `WHERE 1=0` 으로 0행 반환
- `form_script`에서 `setColumns` 호출 금지 — 우회 패턴. bndquery가 단일 진실 소스
- AI 응답 그리드, 임시 작업 그리드 등 DB 영속 없는 모든 경우에 같은 패턴 반복

### 디버깅 — SQL 확인/작성

기존 폼이 조회·저장에서 막힐 때 bndquery부터 점검한다.

```
## 목표
R/C/U/D SQL이 있는지 확인하고, 없으면 작성한다. 있으면 PostgreSQL 문법 검증.

## 현재 SQL 조회 (잘림 없이 -- psql -c는 80자 컷, Python으로 전문 조회)
python3 -c "
import psycopg2, sys
conn = psycopg2.connect(host='${FRAMEWEB_DB_HOST}', port=${FRAMEWEB_DB_PORT}, user='${FRAMEWEB_DB_USER}', password='${FRAMEWEB_DB_PASSWORD}', dbname='${FRAMEWEB_DB_NAME}')
cur = conn.cursor()
cur.execute(\"SELECT bnd_id, crudm, query FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' ORDER BY bnd_id, crudm\")
for row in cur.fetchall(): print(f'--- {row[0]} [{row[1]}] ---'); print(row[2]); print()
cur.close(); conn.close()
"

## PostgreSQL SQL 규칙 검증
아래 항목 확인:
- 선택적 조건: `{{AND col = :param}}` 사용하는가?
- 타입 캐스팅: `:param::timestamp`, `:param::integer` 적절한가?
- COALESCE(timestamp, '') 없는가? → COALESCE(timestamp, NULL) 또는 `col >= :param::timestamp`
- UPDATE WHERE: `:key_old` 유지하는가?
- DELETE WHERE: `_old` 없이 원본 필드명인가?
- 세션변수: `:co_cd`, `:reg_id` (서버 자동 치환)

## 없는 SQL 작성
CUD가 없고 BindChain 단계에서 new_yn='Y'이거나 save_sq > 0인 바인딩만 작성.

INSERT 방법 (Python -- :param 가 SQLAlchemy bind 와 충돌해서 psql 직접 실행 어려움):
python3 -c "
import psycopg2
conn = psycopg2.connect(host='${FRAMEWEB_DB_HOST}', port=${FRAMEWEB_DB_PORT}, user='${FRAMEWEB_DB_USER}', password='${FRAMEWEB_DB_PASSWORD}', dbname='${FRAMEWEB_DB_NAME}')
cur = conn.cursor()
cur.execute(\"INSERT INTO bndquery (bis_id, frm_id, bnd_id, crudm, execution_order, source_type, query, description, c_id) VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'R', 1, 'SQL', %s, '조회', 1)\", ('''SQL문''',))
conn.commit()
print(f'Inserted {cur.rowcount}')
cur.close(); conn.close()
"

## SQL 직접 테스트
{{AND}} 제거 후 비즈니스 DB에서 실행:
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT database FROM bisdb WHERE bis_id='{bis_id}' AND is_active='Y';"
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {조회된_database} -c "SELECT 쿼리"

## 완료 조건
- 모든 바인딩 R SQL 존재 + PostgreSQL 문법 통과
- CUD 대상 C/U/D SQL 존재 + 직접 실행 검증
```

### GET / SEND (외부 API)

외부/내부 API를 호출하는 바인딩은 source_type='API' + crudm='G'(GET) 또는 'S'(SEND)로 등록한다. crudm에 'R'을 쓰면 폼 디자이너 UI가 API 설정을 인식하지 못한다. backend 분기는 `binding_executor.py`의 `source_type == 'API'` 경로(`ApiExecutor.execute_get`)다. 설계 상세는 `bnd-design` 스킬 §6.5를 참조한다.

---

## Pull — BNDPULL (R 파라미터)

bndpull은 검색 대상 바인딩셋이 SELECT SQL의 WHERE 절에 필요한 **검색 조건 및 컨텍스트 값을 외부로부터 입력받는 통로**다.
즉, 검색 대상 바인딩셋은 스스로 검색 조건을 가지는 것이 아니라, 컴포넌트·다른 바인딩셋·SESSION 등의 입력 소스로부터 값을 받아 WHERE 절 구성에 사용한다.

**바인딩셋이 검색 조건을 하나라도 사용하는 경우, 그 입력 값을 제공하는 개체가 폼에 반드시 존재해야 한다.** 입력 개체는 다음 중 하나가 된다:

- 검색 입력 컴포넌트 → `source_type=COMPONENT` (검색 영역은 Field 바인딩셋으로 묶지 않는다. 아래 "검색 영역은 COMPONENT로 매핑한다" 규칙 참조)
- 다른 바인딩셋(예: 그리드의 선택 행) → `source_type=GRID`
- SESSION 컨텍스트 → `source_type=SESSION`

R SQL의 `:파라미터`와 데이터 소스를 연결한다.

```sql
INSERT INTO bndpull (bis_id, frm_id, bnd_id, crudm, param_nm, source_type, source_ref, source_column)
VALUES ('{bis_id}', 'FRM', 'ws_detail', 'R', 'parent_pk', 'GRID', 'ws_master', 'parent_pk')
ON CONFLICT (bis_id, frm_id, bnd_id, crudm, param_nm) DO NOTHING;
```

### source_type 6종

| source_type | source_ref | source_column | 용도 |
|-------------|-----------|---------------|------|
| GRID | bnd_id | col_name | 다른 GridSet 선택행의 컬럼 |
| COMPONENT | fcmp_id | -- | 검색 입력·일반 컴포넌트 값 (**검색 영역 표준**) |
| FIELDSET | fcmp_id | -- | Field 바인딩셋 값 (**검색 영역에는 쓰지 않는다** — 아래 규칙 참조) |
| SESSION | `key`(평문, 예 `reg_id`) | -- | 세션 변수 |
| CONST | -- | -- | default_val에 상수값 |
| SQL | sql_func | -- | SQL 함수 |

**source_type=GRID일 때 source_column 필수** (비우면 500 에러)

**source_type=COMPONENT·FIELDSET은 source_column을 비운다** — 컴포넌트 값을 통째로 전달하므로 컬럼 지정이 필요 없다. `binding_executor`가 화면 데이터(form_data)에서 `source_ref`(fcmp_id) 키로 값을 찾아 SELECT 파라미터에 넣는다.

**source_type=SESSION의 source_ref는 세션 변수 키를 평문으로 쓴다** (예: `reg_id`, `co_cd`). `<$reg_id>` 같은 꺾쇠 형식은 backend가 세션 키와 매칭하지 못해 NULL이 된다 → R의 `WHERE reg_id = :reg_id`가 NULL이라 0행. (실증 PROF100: `<$reg_id>` → 0행 / `reg_id` → 정상 조회.) 단 BNDSAVE의 SESSION은 처리 경로가 달라 `<$reg_id>` 형식을 쓴다(BNDPULL과 구분 — 아래 BNDSAVE 절 참조).

### 검색 영역은 COMPONENT로 매핑한다 (Field 바인딩셋 금지)

사용자가 직접 입력하는 검색 조건 영역(날짜·상태·키워드 등)은 **별도 Field 바인딩셋으로 등록하지 않는다.** 검색 입력 컴포넌트는 어느 바인딩셋에도 속하지 않은 채 화면에만 배치하고, 결과를 조회하는 바인딩셋의 bndpull에서 `source_type=COMPONENT`, `source_ref=검색 컴포넌트 fcmp_id`로 값을 직접 받는다. `source_column`은 비운다.

**Field 바인딩셋을 검색 영역에 쓰지 않는 이유:**

1. Field 바인딩셋은 조회(OPEN) 실행 시 "조회 대상이 아닌 입력 영역"으로 분류되어 값이 초기화된다. 그 결과 검색을 실행하면 입력칸 값이 사라진다.
2. Field 바인딩셋은 바인딩 체인 트리거(`open_trigger`)로 SQL 조회를 받아 값을 채우는 구조다. 사용자가 손으로 입력하는 검색 조건과는 맞지 않는다.

따라서 "검색 → 목록" 화면은 다음 2종으로 구성한다:

- **검색 입력 컴포넌트들** — Field 바인딩셋 없음. 조회 바인딩셋이 bndpull(`COMPONENT`)로 값을 끌어온다.
- **결과 그리드** — Grid 바인딩셋, `open_trigger='OPEN'`. 우상단 OPEN(조회) 버튼으로 검색 컴포넌트 값을 읽어 재조회한다.

```sql
-- 검색 → 목록: 결과 그리드(ws_result)가 검색 컴포넌트 값을 COMPONENT로 받는다
INSERT INTO bndpull (bis_id, frm_id, bnd_id, crudm, param_nm, source_type, source_ref, source_column)
VALUES
  ('{bis_id}', 'FRM', 'ws_result', 'R', 'send_dt_from', 'COMPONENT', 'inp_send_dt_from', NULL),
  ('{bis_id}', 'FRM', 'ws_result', 'R', 'send_status',  'COMPONENT', 'inp_send_status',  NULL)
ON CONFLICT (bis_id, frm_id, bnd_id, crudm, param_nm) DO NOTHING;
```

> 검색 조건이 코드 드롭다운(optionSource)이면, 드롭다운이 넘기는 값(popcod `fcd`)이 SELECT의 비교 대상 컬럼에 저장된 실제 데이터값과 일치해야 검색된다. 라벨(`nm`)이 아니라 값(`fcd`)이 데이터와 같아야 한다.

### 디버깅 — GET 파라미터 점검

```
## 목표
R SQL의 `:파라미터`에 대한 소스 매핑을 bndpull에 등록한다.

## 절차
1. R SQL에서 :파라미터 추출 (Query 단계의 SQL에서)
2. 각 파라미터의 source 결정 (위 source_type 6종 표 참조)
3. INSERT:
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
INSERT INTO bndpull
  (bis_id, frm_id, bnd_id, crudm, param_nm, source_type, source_ref, source_column)
  VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'R', 'param', 'SESSION', 'co_cd', NULL)
ON CONFLICT (bis_id, frm_id, bnd_id, crudm, param_nm) DO NOTHING;"

## 검색이 안 될 때 점검 순서
- 입력칸 값이 조회 후 사라지면 → 검색 영역이 Field 바인딩셋(frmbnd)으로 등록돼 있는지 확인. 있으면 제거하고 COMPONENT 매핑으로 전환.
- 입력값은 유지되는데 결과가 안 걸리면 → 코드 드롭다운이 넘기는 값(popcod fcd)이 대상 컬럼의 실제 데이터값과 같은지 확인 (라벨 nm 아님).
- 날짜 형변환은 `:param::timestamp`보다 `CAST(:param AS timestamp)`를 쓴다.

## 확인
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, param_nm, source_type, source_ref, source_column
FROM bndpull WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;"

## 완료 조건
- 모든 R SQL :파라미터에 대응하는 bndpull 존재
- GRID source_column 전부 채워짐
```

---

## Push / Save — BNDSAVE (CUD 파라미터 매핑) + 입력보조(bispop)

> **생략 가능 vs 명시 필수 (자동 통과 경계)**
> 자동 통과 경로는 request.params/request.data에 그 키가 그대로 있을 때만 우연히 동작한다. 의존하지 말 것.
> - bndpull 생략 OK: R에 `:param`이 전혀 없는 고정 WHERE(예: `WHERE system_cd='Smart'`).
> - bndpull 필수: R에 `:param`이 있으면 소스별 명시 — 검색입력=COMPONENT(source_column 비움), 다른 그리드 선택행=GRID(source_column 필수), 세션=SESSION.
> - bndsave 생략 OK: CUD param 이름 == data_field 이고 단순 통과만 필요한 경우.
> - bndsave 필수: SESSION 값 / 상수·매크로 default / 마스터-디테일 FK(하위가 마스터 bnd_id 참조) / 컬럼명 ≠ param 이름.
>
> **save 자동통과 함정**: bndsave에 매핑 안 된 행 필드는 그대로 SQL 파라미터에 통과된다(`bnd_param_resolver.py`). `origin` 같은 의도적 통과에는 유용하나, 오타 필드명도 조용히 흘러든다. 의도된 통과만 남기고 SQL `:param`과 대조한다.
>
> **SESSION source_ref 형식 비대칭**: bndpull=평문(`reg_id`), bndsave=꺾쇠(`<$reg_id>`). 서로 바꾸면 조용히 NULL/0행. 근거: `paramResolver.ts`(평문 키 조회) vs `bnd_param_resolver.py` `SESSION_VAR_PATTERN`(`<\$(\w+)>`).

> **저장 디버깅 1순위 — bndsave 저장 필드 전수 정의 확인 (자동통과 의존 금지)**
> 저장이 안 되는 폼은 bndsave부터 본다. C/U SQL이 쓰는 모든 `:파라미터`가 bndsave에 명시 정의돼 있어야 한다. 자동통과(bndsave에 없으면 화면 행 필드가 그대로 SQL 파라미터로 흘러가는 동작)에 의존하지 말 것 — 자동통과는 화면 데이터에 그 키가 마침 있을 때만 우연히 동작하고, 키가 없거나 이름이 다르면 파라미터가 NULL이 되어 조용히 저장 실패(0행 또는 필수값 에러)한다.
>
> 점검 절차:
> 1. C/U SQL에서 `:파라미터` 전부 추출(typecast·세션변수 제외).
> 2. 각 파라미터가 bndsave에 명시 정의됐는지 대조. 누락분을 GRID(자기 bnd_id, source_column=필드명) 또는 마스터 참조(하위 FK), SESSION 등으로 명시 등록.
> 3. 마스터-디테일이면 하위의 FK는 마스터 bnd_id 참조(자기 참조 금지).
>
> 실제 사례(ROL100 역할관리): ws_role bndsave가 0건(전 필드 자동통과 의존), ws_member는 rol_cd 하나만(reg_id·mng_yn 누락)이라 저장이 실패했다. C/U SQL 파라미터(ws_role: rol_cd·title·grp_bc·use_yn·disp_sq·dsc / ws_member: rol_cd·reg_id·mng_yn)를 bndsave에 전부 명시 정의해 해소.

### Save 탭 자동 매핑 버튼 (동작·한계·보정)

폼 디자이너 Binding > Save 탭 우상단에 자동 매핑 버튼 두 개가 있다. 둘은 bndsave 초안을 생성하는 도구다. 결과를 그대로 저장하지 않고, 아래 한계를 사람이 보정한 뒤 저장한다. 코드 근거는 `frontend/src/components/BindingUI/detail/tabs/BindingSaveTab.tsx`.

#### 두 버튼

| 버튼 | 함수 | 동작 |
|------|------|------|
| 새로고침 (RefreshCw, "쿼리에서 필드 추출") | `extractSqlFields` | 현재 바인딩의 C/U SQL을 읽어 `:param`을 파싱하고 Save 매핑 표에 필드(param_nm) 행만 추가한다. 이미 있는 필드는 건너뛴다. 대상(source_ref / source_column)은 비운 채로 둔다 — 대상 채우기는 링크 버튼 또는 수동 몫. |
| 링크 (Link2, "매핑에서 바인딩") | `applyBndcmpMappings` | 각 필드의 대상을 Model(bndcmp) 기준으로 자동으로 채운다. 규칙: bndcmp에 있는 컬럼이면 GRID 자기 그리드 참조(source_ref=자기 bnd_id, source_column=param_nm) / `_old` 접미사면 원본 컬럼 / 세션키 이름이면 SESSION(`<$key>`). |

#### 자동 매핑의 한계 — 사람이 보정한다

두 버튼은 초안 생성까지다. 아래는 자동으로 틀린다. 저장 전 수동 검수한다.

1. **마스터-디테일 FK를 모른다.** 하위 그리드의 FK(마스터에서 와야 하는 값)도 링크 버튼이 GRID 자기 그리드 참조로 채운다. 마스터-디테일 관계 정보가 없기 때문이다. source_ref를 마스터 bnd_id로 수동 보정한다(아래 "하위 GridSet FK는 마스터 참조" 규칙 참조). 미보정 시 자기참조라 그 값이 행에 없어 비고, 저장이 필수값 에러나 0행으로 실패한다. 실제 사례(ROL100): 링크 버튼이 `rol_cd`(마스터 ws_role의 역할코드)를 하위 ws_member 자기참조로 채워 "rol_cd 필수 입력" 에러 발생 → source_ref를 ws_role로 보정해 해소.
2. **세션키 판정이 폼마다 어긋날 수 있다.** `reg_id`·`co_cd` 같은 이름은 세션키로 자동 분류되어 SESSION으로 매핑되는데, 폼에 따라 의미가 다르다. 예: ROL100의 `reg_id`는 세션 작성자가 아니라 화면에서 지정하는 사용자ID(rolusr.reg_id)라 GRID(자기 그리드) 또는 사용자 선택값이어야 한다. 실제 의미를 확인해 GRID/COMPONENT 등으로 보정한다.

#### 권장 절차

`새로고침(필드 추출) → 링크(대상 1차 자동) → 마스터 FK·세션 의미 전수 검수 → 저장`. 자동 매핑 결과를 그대로 저장하지 않는다. 특히 마스터-디테일 폼이면 하위 FK의 source_ref가 자기 bnd_id로 잡혀 있지 않은지 반드시 확인한다.

**CUD SQL의 `:파라미터`와 데이터 소스를 연결한다. 누락 시 Save 에러.**

```sql
INSERT INTO bndsave (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, source_column, default_val)
VALUES ('{bis_id}', 'FRM', 'ws_xxx', 'col1', 'GRID', 'ws_xxx', 'col1', NULL)
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;
```

### 매핑 규칙

| 필드 패턴 | source_type | source_ref | source_column |
|-----------|-------------|-----------|---------------|
| 일반 컬럼 | GRID | 자기 bnd_id | 필드명 |
| _old 컬럼 (UPDATE WHERE) | GRID | 자기 bnd_id | 원본 필드명 (xxx_old -> xxx) |
| reg_id, m_id, c_id | SESSION | `<$reg_id>` | NULL |
| co_cd | SESSION | `<$co_cd>` | NULL |
| bs_cd | SESSION | `<$bs_cd>` | NULL |
| FK (하위 GridSet) | GRID | **마스터 bnd_id** | 필드명 |

### 하위 GridSet FK는 마스터 참조

```sql
-- 올바른 매핑: source_ref = 마스터 (ws_master)
VALUES (..., 'ws_detail', 'parent_pk', 'GRID', 'ws_master', 'parent_pk', ...)

-- 잘못된 매핑: source_ref = 자기 자신 (ws_detail) -> Save 시 값 비어있음
```

### 전체 CUD 파라미터 추출 방법

forms.sql 생성 시 CUD SQL에서 파라미터를 자동 추출하여 BNDSAVE INSERT를 만든다.

```python
# CUD SQL에서 :param 추출 후 typecast/세션변수 제거
import re
params = set(re.findall(r':(\w+)', cud_sql))
# ::typecast 제거
params = {p for p in params if p not in ('timestamp','integer','numeric','text','date','boolean')}
# 세션변수는 별도 SESSION 매핑
session_keys = {'co_cd','reg_id','bs_cd','m_id','c_id'}
grid_params = params - session_keys
```

각 `grid_params` 항목 → GRID 매핑 INSERT, 각 `session_keys` 교집합 → SESSION 매핑 INSERT.

### 디버깅 — SET 매핑 점검

```
## 목표
CUD SQL의 `:파라미터` 매핑(bndsave) + 필요시 입력보조(bispop) 설정.

## [bndsave] DB 직접 INSERT

### Step 1: CUD SQL에서 파라미터 추출 (Python -- :param 잘림 회피 + typecast 제거 + 세션변수 제외)
python3 -c "
import psycopg2, re
conn = psycopg2.connect(host='${FRAMEWEB_DB_HOST}', port=${FRAMEWEB_DB_PORT}, user='${FRAMEWEB_DB_USER}', password='${FRAMEWEB_DB_PASSWORD}', dbname='${FRAMEWEB_DB_NAME}')
cur = conn.cursor()
cur.execute(\"SELECT bnd_id, crudm, query FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' AND crudm IN ('C','U','D') ORDER BY bnd_id, crudm\")
for row in cur.fetchall():
    params = set(re.findall(r':(\w+)', row[2] or '')) - {'co_cd','reg_id','bs_cd'}
    # ::typecast 제거
    params = {p for p in params if not p.startswith(('timestamp','integer','numeric','text','date','boolean'))}
    print(f'{row[0]} [{row[1]}]: {sorted(params)}')
cur.close(); conn.close()
"

### Step 2: 매핑 규칙 (위 '매핑 규칙' 표 참조)

### Step 3: INSERT
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
INSERT INTO bndsave
  (bis_id, frm_id, bnd_id, param_nm, source_type, source_ref, source_column, default_val)
  VALUES ('{bis_id}', '{폼ID}', 'BND_ID', 'col1', 'GRID', 'BND_ID', 'col1', NULL)
ON CONFLICT (bis_id, frm_id, bnd_id, param_nm) DO NOTHING;"

### Step 4: 하위 Grid PK/FK 자체참조 확인 (★ 중요)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, param_nm, source_ref
FROM bndsave
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND source_type = 'GRID'
AND source_ref = bnd_id
AND param_nm IN (SELECT data_field FROM bndcmp
                 WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
                 AND visible = false);"
히트되면 마스터 참조로 수정 (UPDATE bndsave SET source_ref='ws_master' WHERE ...).

## 확인
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, param_nm, source_type, source_ref, source_column
FROM bndsave WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;"

## 완료 조건
- CUD 바인딩 전체 파라미터 매핑 완료
- 미매핑 필드 없음
- 하위 Grid PK/FK가 마스터 참조 확인 (자체 참조 없음)
```

### 입력보조 (bispop)

CODE/LOOKUP/POPUP editor_type 컬럼은 bispop으로 입력보조를 붙인다. bispop은 bis_id 단위로 관리된다 — 새 bis에 팝업이 없으면 EPIC_SELECT 403 에러가 난다. 다른 bis에서 이식하거나 신규 생성한다.

```
1. 필요 팝업 확인 (bndcmp에서 editor_type이 CODE/LOOKUP/POPUP인 것)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, fcmp_id, editor_type FROM bndcmp
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND editor_type IN ('CODE','LOOKUP','POPUP');"

2. bispop 조회 (비즈니스 DB):
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {bis_database} -c "
SELECT pop_id, pop_nm, pop_type FROM bispop ORDER BY pop_id;"

3. 없으면 다른 bis에서 이식 또는 신규 생성
★ bispop은 bis_id 단위. 팝업 없으면 EPIC_SELECT 403 에러
```

입력보조의 종류별 등록 구조(CODE/SUBCODE/LOOKUP/POPUP)는 아래 Model(bndcmp) 절에서 다룬다.

---

## Model — BNDCMP (그리드 컬럼 매핑)

### model 정합 규칙 (R 쿼리 출력 ≡ bndcmp.data_field)

R(조회) 쿼리의 SELECT 출력 컬럼명(별칭 포함)과 `bndcmp.data_field`는 정확히 1:1로 일치해야 한다.

- R이 SELECT하지 않는 `data_field`는 그리드/필드에 빈값으로 들어온다. 화면에 즉시 드러나지 않아 발견이 늦다.
- 그 빈 컬럼을 원본 키로 삼으려 해도 값이 없어 `_old`/`origin` 식별이 비고, 수정·삭제 시 0행 처리로 조용히 실패한다.
- 별칭을 쓸 때는 R의 별칭 이름과 `data_field`를 같은 이름으로 맞춘다. 예: `data_field='origin'`이면 R에 `menu_cd AS origin`. `menu_cd AS mn_origin`처럼 별칭이 어긋나면 `origin`은 영원히 빈값이다(MNU100 실제 사례).
- 근거: `createFieldSetStateCallbacks.ts`의 `originalValue: data[m.data_field]`가 R 결과를 `data_field` 키로 읽으므로, 이름이 다르면 `undefined`.

PK: `(bis_id, frm_id, bnd_id, fcmp_id)`

| 항목 | 규칙 |
|------|------|
| fcmp_id | 그리드 내 고유 ID. 관례: col_{data_field} |
| data_field | **필수**. DB 컬럼명 (SELECT alias) |
| data_type | string, number, date, boolean |
| col_seq | 1부터 순차 |
| col_width | px 단위. 0이면 숨김 |
| visible | false면 표시 안 함 (FK 숨김용) |
| is_required | Save Registry 검증 |
| is_readonly | true면 편집 불가 |
| editor_type | TEXT, CHECKBOX, DATE, CODE, SUBCODE, LOOKUP, POPUP |
| editor_config | JSON 문자열. 입력보조 설정 (아래 참조) |
| **properties.optionSource** | **입력보조 연결의 핵심. pop_id를 여기에 넣어야 폼 디자이너 UI + 런타임 양쪽에서 동작** |

### 입력보조 (editor_type별 설정)

4가지 입력보조 도구. **코드는 비즈니스 DB(bispop + popcod)에 등록.**

> **optionSource는 properties에 넣는다 (editor_config 아님)**
> - `properties`: JSONB 컬럼 -> 폼 디자이너 Model 탭의 option_source 필드와 연동
> - `editor_config`: Text 컬럼 -> 레거시 호환용, properties가 우선
> - forms.sql 작성 시 **properties에 반드시 포함**: `'{"optionSource":"POP_ID"}'`

#### CODE -- 코드 드롭다운

```sql
-- 1. 비즈니스 DB에 코드 등록 (seed.sql)
INSERT INTO bispop (pop_id, pop_nm, pop_type)
VALUES ('MSSTAT', '마일스톤 상태', 'C') ON CONFLICT (pop_id) DO NOTHING;

INSERT INTO popcod (pop_id, fcd, sub_cd, nm, sort_order) VALUES
('MSSTAT', 'MSSTAT100', '100', 'open', 1),
('MSSTAT', 'MSSTAT200', '200', 'active', 2),
('MSSTAT', 'MSSTAT300', '300', 'closed', 3)
ON CONFLICT (pop_id, fcd) DO NOTHING;

-- 2. BNDCMP에 연결
INSERT INTO bndcmp (..., editor_type, properties) VALUES
(..., 'CODE', '{"optionSource": "MSSTAT"}');
```

| 항목 | 규칙 |
|------|------|
| pop_type | 'C' (Code) |
| fcd | pop_id + sub_cd (예: MSSTAT100) -- **DB 저장값** |
| sub_cd | 코드 번호 (예: 100, 200, 300) |
| nm | 화면 표시 라벨 (예: open, active, closed) |
| properties | `{"optionSource": "POP_ID"}` -- bispop.pop_id 참조 |

#### SUBCODE -- 서브코드 드롭다운

CODE와 구조 동일. 차이: **저장값이 fcd가 아닌 sub_cd**.

```sql
-- 예시: 통화코드 (CURRTY)
-- fcd = CURRTYUSD, sub_cd = USD -> DB에는 'USD'만 저장됨
INSERT INTO bndcmp (..., editor_type, properties) VALUES
(..., 'SUBCODE', '{"optionSource": "CURRTY"}');
```

| 타입 | DB 저장값 | 언제 사용 | 예시 |
|------|----------|-----------|------|
| **CODE** | **fcd** (MSSTAT200) | 코드 의미가 변할 수 있을 때 | 상태, 우선순위 |
| **SUBCODE** | **sub_cd** (USD) | 코드 자체가 불변일 때 | 통화, 국가코드 |

**왜 CODE는 fcd를 저장하나?** `active`를 `in_progress`로 바꾸거나, 하나의 코드가 둘로 분리되어도 DB에 저장된 fcd(MSSTAT200)는 그대로 유지된다. nm만 변경하면 됨. 기존 데이터가 깨지지 않는다.

**왜 SUBCODE는 sub_cd를 저장하나?** USD는 영원히 USD다. fcd(CURRTYUSD)를 저장하면 오히려 불편.

#### LOOKUP -- SQL 검색 드롭다운

popsql 쿼리 결과를 드롭다운으로 보여준다.

```sql
-- 1. bispop (비즈니스 DB)
INSERT INTO bispop (pop_id, pop_nm, pop_type)
VALUES ('IMPFRML', '폼 목록', 'L') ON CONFLICT (pop_id) DO NOTHING;

-- 2. popsql (비즈니스 DB)
INSERT INTO popsql (pop_id, query)
VALUES ('IMPFRML', 'SELECT frm_id, frm_nm FROM frmmst ORDER BY frm_id')
ON CONFLICT DO NOTHING;

-- 3. popio OUT 매핑 (시스템 DB)
INSERT INTO popio (pop_id, direction, field_name, role) VALUES
('IMPFRML', 'OUT', 'frm_id', 'value'),
('IMPFRML', 'OUT', 'frm_nm', 'label');

-- 4. BNDCMP 연결
INSERT INTO bndcmp (..., editor_type, properties) VALUES
(..., 'LOOKUP', '{"optionSource": "IMPFRML"}');
```

#### POPUP -- 팝업 검색 모달

텍스트 + `[...]` 버튼. 클릭하면 그리드 팝업이 열려 행을 선택한다.
LOOKUP보다 복잡: **여러 컬럼을 동시에 채울 수 있다** (OUT 매핑 다건).

```sql
-- 1. bispop (비즈니스 DB)
INSERT INTO bispop (pop_id, pop_nm, pop_type)
VALUES ('IMPFRM', '폼 목록', 'P') ON CONFLICT (pop_id) DO NOTHING;

-- 2. popsql (비즈니스 DB)
INSERT INTO popsql (pop_id, query)
VALUES ('IMPFRM', 'SELECT frm_id, frm_nm, frm_desc FROM frmmst ORDER BY frm_id')
ON CONFLICT DO NOTHING;

-- 3. popio IN/OUT (시스템 DB)
INSERT INTO popio (pop_id, direction, field_name, role) VALUES
('IMPFRM', 'OUT', 'frm_id', 'value'),
('IMPFRM', 'OUT', 'frm_nm', 'label'),
('IMPFRM', 'OUT', 'frm_desc', NULL);
-- field_name은 대상 그리드의 data_field와 일치해야 함

-- 4. popcmp: 팝업 그리드 컬럼 (시스템 DB)
INSERT INTO popcmp (pop_id, field_name, label, width, visible, display_order) VALUES
('IMPFRM', 'frm_id', '폼ID', 120, true, 1),
('IMPFRM', 'frm_nm', '폼명', 200, true, 2),
('IMPFRM', 'frm_desc', '설명', 300, true, 3);

-- 5. BNDCMP 연결
INSERT INTO bndcmp (..., editor_type, properties) VALUES
(..., 'POPUP', '{"optionSource": "IMPFRM"}');
```

#### LOOKUP vs POPUP 선택 기준

| 기준 | LOOKUP | POPUP |
|------|--------|-------|
| 선택지 수 | ~50건 이하 | 대량 |
| 결과 매핑 | 1컬럼 (value+label) | **다건 컬럼 동시 채우기** |
| UI | 드롭다운 | 모달 그리드 + 검색 |
| 검색 | 프론트 필터링 | 서버사이드 쿼리 |

### 숨김 FK 패턴

마스터-디테일에서 자식 그리드의 FK(부모 PK)는 숨김:
```sql
-- visible=false, col_width=0
('col_parent_pk', 'parent_pk', 'string', 1, '부모PK', false, false, 0, 'center', 'TEXT')
```

### bndcmp is_pk 설정 필수

```yaml
문제: is_pk가 false면 PK 변경 시 _old가 잘못 채워져 0행/오행 UPDATE(조용한 실패)
규칙:
  - bndquery UPDATE/DELETE의 WHERE 식별 컬럼 -> 해당 bndcmp.is_pk = true
  - 메커니즘: is_pk=true 컬럼은 프론트가 원본값을 추적해 저장 시 {pk}_old를 자동 첨부
    (FieldSet: worksetTrackingStore.buildFieldSetParams)
  - is_pk 누락 시: backend가 현재값으로 _old를 채워, PK를 바꾸면 새 값으로 WHERE가 잡혀 0행
  - 폼 유형별 차이:
      FieldSet(단건): is_pk=true면 _old 자동 생성됨
      GridSet(그리드): 체인 SAVE 경로는 is_pk와 무관하게 _old를 자동 생성하지 않음
                       -> 원본 키는 R 쿼리 별칭 컬럼으로 내려받아야 함('PK 변경 + CTE cascade' 절)
  - WHERE가 별칭(:origin 등)으로 식별하면 그 별칭도 R에서 SELECT + bndcmp 같은 이름 등록
  예시(일반 조회): WHERE emp_cd = :emp_cd -> bndcmp(data_field='emp_cd').is_pk = true
  예시(PK 변경):  WHERE grp_cd = :grp_cd_old, SET grp_cd = :grp_cd -> grp_cd is_pk=true
  안티패턴: MNU100은 식별 컬럼 is_pk가 전부 false -> _old 미생성 구조
```

---

## 특수 패턴

### 피봇 그리드 (가로 매트릭스)

한 테이블의 세로 데이터(키·분류·값 구조)를 가로 매트릭스로 펼쳐 보여주는 그리드. 분류 컬럼의 값들을 그리드 컬럼으로, 값 컬럼을 셀로 전개한다.
예: langtxt(txt_key, lang_cd, txt_value) → (txt_key | ko | en | la). 잔고(종목, 항목, 수량) → 종목별 집계 컬럼.

#### 언제 쓰나
- 같은 키에 대해 분류(언어/항목/기간)별 값을 한 행에 나란히 보여주고 싶을 때.
- 원본 테이블은 세로(키·분류·값)인데 화면은 가로 매트릭스가 보기 편할 때.

#### 조회 R — CASE WHEN 피봇
별도 피봇 함수 없이 R SQL로 만든다. `MAX(CASE WHEN 분류컬럼='값' THEN 값컬럼 END) AS 별칭` + 키로 GROUP BY.

```sql
SELECT txt_key,
  MAX(CASE WHEN lang_cd='KO' THEN txt_value END) AS ko,
  MAX(CASE WHEN lang_cd='EN' THEN txt_value END) AS en,
  MAX(CASE WHEN lang_cd='LA' THEN txt_value END) AS la
FROM langtxt
WHERE txt_key LIKE :key_prefix
GROUP BY txt_key
ORDER BY txt_key
```

- 피봇 컬럼(ko/en/la)은 **고정**이다. 분류 값이 늘면(언어 추가) SQL의 CASE WHEN과 그리드 컬럼도 함께 추가한다.
- 대상 범위는 BNDPULL 파라미터(prefix·조건)로 좁힌다. 검색 입력이 비면 전체가 조회되지 않도록 LIKE 조건을 둔다.

#### BNDCMP — 피봇 결과 컬럼
피봇 별칭(ko/en/la)을 그대로 data_field로 둔다. 키 컬럼은 is_pk=true, is_readonly=true. 편집 매트릭스면 값 컬럼은 is_readonly=false.

#### 그리드 properties
EPIC_HANDSONTABLE을 쓴다. 편집 매트릭스면 tb_save=true, 조회 전용이면 tb_save=false. windmilld95b/WM010의 grid_pivot properties를 참조한다.

#### 크로스-DB 주의
피봇 R은 **한 DB의 한 테이블**에서 한다. 폼 메타(bisfrm 등)는 시스템 DB에 있지만 R 쿼리는 비즈 DB에서 실행되므로, 비즈 DB 테이블만 피봇한다. 시스템 DB 테이블과 JOIN이 필요해지면 안 된다 — 원문 같은 보조값은 피봇 대상 테이블에 미리 시드해 둔다. 예: langtxt의 KO 행에 원문을 넣어두면 ko 컬럼이 곧 원문이 되어 frmcmp 등 시스템 DB와의 JOIN이 불필요하다.

#### 저장 주의 — 가로 1행 ↔ 원본 N행
피봇 그리드 한 행(키 + ko/en/la)을 저장하면 원본 테이블 N행(분류별)으로 나눠 써야 한다. 표준 BNDSAVE(1행 → 1테이블행)로는 처리되지 않는다. 두 방법 중 선택:
- 분류별 다중 UPSERT — bndquery C/U를 언어(분류) 단위로 나눠 작성.
- 폼 스크립트(beforeSave)에서 그리드 행을 분류별로 분해해 UPSERT (frameweb-script 스킬 참조).

조회 전용 피봇(WM010 같은 요약 대시보드)은 저장이 없어 이 문제가 없다.

#### 선례
- windmilld95b / WM010 — GROUP BY 집계 피봇 (조회 전용 대시보드, frm_desc "종목별 피봇 잔고").
- I18N100 (다국어 번역 관리) — langtxt 언어 피봇 (편집 매트릭스, `packages/pkg-02-menu-permission/forms-i18n.sql`).

### FTP/Image 컴포넌트 바인딩

FTP(파일 첨부)와 Image(이미지 표시/편집)는 **컴포넌트 자체가 API를 직접 호출**한다.
BindingSet은 **추가 정보(rmks 등)만 저장**하는 보조 역할.

> **이 예외는 멀티파트 업로드(파일 바이너리 직접 전송)라는 특수 사례에만 적용.** 일반 외부 API(날씨/환율/주가 같은 JSON 응답)에는 적용 안 된다 — 그건 BNDQUERY `source_type='API'` 가 표준 (`bnd-design` 스킬 §6.5). 컴포넌트 자체 fetch 우회 같은 패턴 반복 금지.

```yaml
FTP_업로드_삭제: "컴포넌트 -> /api/ftp/* API 직접 호출 (BindingSet 경유 안 함)"
Image_업로드_삭제: "컴포넌트 -> /api/image API 직접 호출"
BindingSet_Save: "추가 정보(rmks, 비고 등)만 BNDSAVE로 저장"
R_SQL: "ftpmst(파일메타) LEFT JOIN ftpimg(추가정보) -- 항상 JOIN 구조"
```

---

## 디버깅 흐름

기존 폼이 동작하지 않을 때 점검 순서. frmbnd(BindChain 단계)가 이미 보정됐다는 전제다.

> **저장 디버깅은 SAVE 탭(bndsave) 저장 필드 정의 확인부터.** C/U SQL의 `:파라미터`가 bndsave에 전부 명시 정의됐는지 먼저 본다. 자동통과 의존이 저장 실패의 흔한 원인이다 (위 "저장 디버깅 1순위" 박스 참조).

```
Query(bndquery) 점검 → Pull(bndpull) 점검 → Save(bndsave) + 입력보조(bispop) 점검 → frameweb-verify
```

| 증상 | 우선 점검 |
|------|-----------|
| 조회(Open) 0행 / 에러 | bndquery R SQL 문법 + bndpull 파라미터 소스 |
| 검색해도 결과 안 걸림 | bndpull COMPONENT 매핑 + 코드 드롭다운 fcd 값 |
| 검색 입력칸이 조회 후 비워짐 | 검색 영역이 Field 바인딩셋으로 등록됨 → COMPONENT로 전환 |
| **저장(Save) 에러 / 0행** | **bndsave 저장 필드 전수 정의(자동통과 의존 금지) ← 1순위** + _old + 하위 FK 마스터 참조 |
| 그리드 컬럼 빈값 | bndcmp.data_field ≡ R SELECT 별칭 정합 |
| PK 수정이 0건으로 응답 | is_pk + R 별칭(origin) 보존 + CTE 메인 문장 위치 |
| 코드 드롭다운/팝업 미동작 | bndcmp.properties.optionSource + bispop(bis_id 단위) |

---

## 후속 — frameweb-verify 연계

Query/Pull/Save/Model 보정을 마치면 실제 런타임 동작을 검증한다. 폼 열기 조회 테스트와 CRUD 저장·삭제 테스트는 ERP Preview에서 브라우저로 확인해야 하므로, 모리 런타임 검증 스킬 `frameweb-verify`로 연계한다.

- 디버깅 완료 → `/frameweb-verify {frm_id} {version} {bis_id}` 로 폼 열기·CRUD 런타임 검증 (version 미지정 시 최신 배포본)
- 검증 실패(조회/저장 미동작) → 실패 항목에 따라 이 스킬의 Query/Pull/Save 절로 복귀
- 배포가 필요하면 → `frameweb-deploy`

> 런타임 검증을 직접 실행하지 않는다. ERP Preview URL(`${FRAMEWEB_APP_URL}/erp/form/{폼ID}?bis_id={bis_id}`)에서의 동작 확인은 모리(Windows Chrome) 영역이며 `frameweb-verify`가 의뢰서를 생성한다.

---

## 후속 과제

설계 청사진 `docs/specs/form-skill-tab-restructure.md` 9절의 미결 항목.

- Query 하위(SELECT/INSERT/UPDATE/DELETE/GET/SEND)를 이 스킬 한 곳에서 단계로 둘지, 더 잘게 나눌지는 1차 구현 후 사용량 보고로 결정한다.
- frameweb-canvas(컴포넌트 도메인)와의 책임 경계 — frmcmp 컴포넌트 속성과 bndcmp 컬럼 매핑이 겹치는 영역 정리가 필요하다.

## 참조 문서

- [sql_conversion_rules.md](references/sql_conversion_rules.md) -- MSSQL → PostgreSQL SQL 변환 규칙 (BNDQUERY R/C/U/D 작성 시)
- 도메인 단일 소스: `docs/architecture/domains/forms.md`

<!-- writer-check: 위 원칙 전원 준수. 출처(Binding 탭 SQL·파라미터·매핑 작업 + "Task 간 관계"·"다오 직접 vs 모리 위임"·"실행 예시", frameweb-canvas BNDQUERY/BNDPULL/BNDSAVE/BNDCMP) 원문 보존. TaskCreate 절차 본문은 복제하지 않고 기존 Query/Pull/Save 절 인용으로 중복 제거. 저장 디버깅 1순위(bndsave 전수 정의 / 자동통과 의존 금지)는 메인 다오가 코드/DB로 확인한 사실(ROL100 사례) 그대로 반영. 기존 "save 자동통과 함정" 박스(통과 위험 관점)와 새 박스(누락 시 실패 관점)는 관점이 달라 둘 다 유지. 추측 없음. -->
