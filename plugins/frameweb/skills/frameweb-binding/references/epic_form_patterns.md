# Epic Framework 폼 패턴 정답지

> 28개 실제 운영 폼(9개 패키지)에서 추출한 유형별 INSERT 패턴
> ROCA 유형(A~E)과 1:1 매핑 + Epic 전용 유형(F, X) 추가

---

## INSERT 순서 (모든 유형 공통)

```
BISFRM → FRMCMP → FRMBND → BNDCMP → BNDQUERY → BNDPULL → BNDSAVE
  (폼)    (컴포넌트)  (바인딩)   (컬럼매핑)  (SQL)     (R파라미터) (CUD파라미터)
```

- 모든 INSERT에 `ON CONFLICT DO NOTHING`
- FRMCMP의 parent_id는 자기 참조 → 부모 컴포넌트를 먼저 INSERT

---

## 유형 A: SimpleGrid — 단일 그리드

```
대표 폼: MNU100, USR100, FRM100, DSP200, EX_GRID_CRUD
ROCA 대응: D-SearchGrid (검색 없이 그리드만)
```

### 구조

```
┌─────────────────────────────┐
│ EPIC_HANDSONTABLE (fill)    │
│ layout_mode='fill'          │
│ parent_id = NULL (루트)     │
└─────────────────────────────┘
```

### 테이블별 건수

```
BISFRM:   1건
FRMCMP:   1건 (Grid만)
FRMBND:   1건
BNDCMP:   N건 (컬럼 수)
BNDQUERY: R + C + U + D (각 1건)
BNDPULL:  없음
BNDSAVE:  없음 (Grid는 자동 매핑)
```

### FRMBND 설정

```sql
bnd_type      = 'Grid'
open_trigger  = 'OPEN'       -- 폼 열릴 때 자동 조회
open_sq       = 1
save_sq       = 1             -- 0이면 R-only (FRM100)
trigger_yn    = false
new_yn        = 'Y'           -- 'N'이면 행 추가 불가
bound_fcmp_ids = '["grid_id"]'
```

### BNDQUERY 패턴

```sql
-- R: 전체 조회
'SELECT * FROM table ORDER BY ...'

-- C: INSERT
'INSERT INTO table (col1, col2, ..., created_by, created_at)
 VALUES (:col1, :col2, ..., :reg_id, now())'

-- U: UPDATE
'UPDATE table SET col1 = :col1, ..., updated_by = :reg_id, updated_at = now()
 WHERE pk = :pk_old'

-- D: 자식 먼저 삭제 → 본체 삭제 (execution_order로 순서)
'DELETE FROM child WHERE fk = :pk'   -- execution_order = 1
'DELETE FROM parent WHERE pk = :pk'  -- execution_order = 2
```

---

## 유형 B: Field+Grid Chain — 목록 선택 → 상세 필드

```
대표 폼: EX_FIELD_CRUD, EX_SQLEDITOR, EX_IMAGE_GUIDE
ROCA 대응: 해당 없음 (ROCA에는 FieldSet 개념 없음)
```

### 구조

```
┌──────────────────────────────────┐
│ CMP_SPLIT_CONTAINER (horizontal) │
│ ┌──────────────┐ ┌────────────┐ │
│ │ SplitPanel 0 │ │SplitPanel 1│ │
│ │              │ │            │ │
│ │ Grid (목록)  │ │ Field들    │ │
│ │ [BND_LIST]   │ │ [BND_DETAIL]│ │
│ │              │ │ fld_name   │ │
│ │              │ │ fld_email  │ │
│ │              │ │ [저장]     │ │
│ └──────────────┘ └────────────┘ │
└──────────────────────────────────┘
```

### 테이블별 건수

```
BISFRM:   1건
FRMCMP:   5+건 (Split+Panel+Grid+Field들)
FRMBND:   2건 (목록 Grid + 상세 Field)
BNDCMP:   N건
BNDQUERY: 목록 R + 상세 RCUD
BNDPULL:  1+건 (상세 R에 목록 PK 전달)
BNDSAVE:  N건 (상세 CUD 파라미터)
```

### FRMBND 체인

```sql
-- 목록 (Grid) — 폼 열릴 때 자동 조회, 행 선택 시 디테일 트리거
bnd_type='Grid', open_trigger='OPEN', open_sq=1, trigger_yn=true, save_sq=0

-- 상세 (Field) — 목록 선택에 의존, 저장 담당
bnd_type='Field', open_trigger='BND_LIST', open_sq=2, trigger_yn=false, save_sq=1
```

### BNDPULL (상세 R 파라미터)

```sql
-- 상세 R 쿼리의 :pk 를 목록 그리드 선택행에서 가져옴
source_type='GRID', source_ref='BND_LIST', source_column='pk_column'
```

### BNDSAVE source_type 3종

```sql
-- 마스터 그리드에서 FK
source_type='GRID',    source_ref='BND_LIST',   source_column='pk_col'

-- 자기 필드값
source_type='FIELD',   source_ref='BND_DETAIL', source_column='field_name'

-- 세션 변수
source_type='SESSION', source_ref='<$reg_id>'
```

### bound_fcmp_ids 주의

```
Grid: bound_fcmp_ids = '["grid_cmp_id"]'
Field: bound_fcmp_ids = '[]'   ← 빈 배열! BNDCMP에서 개별 매핑
```

---

## 유형 C: MasterDetail — 상하 2개 그리드

```
대표 폼: ROL100, WM200, WM500, DSP300, DSP400, EX_MASTER_DETAIL
ROCA 대응: C-MasterDetail
```

### 구조

```
┌──────────────────────────────────┐
│ CMP_SPLIT_CONTAINER (vertical)   │
│ ┌──────────────────────────────┐ │
│ │ SplitPanel 0                 │ │
│ │ Grid: 마스터 [ws_master]     │ │
│ ├──────────────────────────────┤ │
│ │ SplitPanel 1                 │ │
│ │ Grid: 디테일 [ws_detail]     │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘

ws_master ──trigger──→ ws_detail
 (open_sq=1)            (open_sq=2)
```

### FRMBND 체인

```sql
-- 마스터
bnd_type='Grid', open_trigger='OPEN', open_sq=1, save_sq=1, trigger_yn=true

-- 디테일
bnd_type='Grid', open_trigger='ws_master', open_sq=2, save_sq=2
```

### BNDPULL (디테일 R 파라미터)

```sql
-- 디테일 R의 :master_pk를 마스터 그리드 선택행에서 추출
bnd_id='ws_detail', crudm='R',
param_nm='master_pk',
source_type='GRID', source_ref='ws_master', source_column='master_pk'
```

### BNDSAVE (디테일 CUD — FK 주입)

```sql
-- FK: 마스터에서
param_nm='master_pk', source_type='GRID', source_ref='ws_master', source_column='master_pk'

-- 자기 컬럼: 자기 그리드에서
param_nm='detail_col', source_type='GRID', source_ref='ws_detail', source_column='detail_col'

-- UPDATE용 _old 패턴
param_nm='pk_old', source_type='GRID', source_ref='ws_detail', source_column='pk'
```

### 마스터 삭제 시 연쇄

```sql
-- D execution_order=1: 자식 일괄 삭제
'DELETE FROM detail_table WHERE fk = :pk'
-- D execution_order=2: 마스터 삭제
'DELETE FROM master_table WHERE pk = :pk'
```

### 디테일 R-only 변형 (WM500)

```
ws_detail: save_sq=0, BNDQUERY에 R만, BNDSAVE 없음
```

---

## 유형 D: TabbedMasterDetail — 탭으로 다중 디테일

```
대표 폼: PMS100, EX_TABS_CHAIN, DSP010
ROCA 대응: C-TabForm + E-SearchMasterDetail (탭 부분)
```

### 구조

```
┌───────────────────────────────────────┐
│ CMP_TABS                              │
│ ┌─Tab0──────────┬─Tab1──────────┐    │
│ │ Split          │ Split          │    │
│ │ ┌────────────┐│ ┌────────────┐│    │
│ │ │마스터 Grid ││ │마스터 Grid ││    │
│ │ ├────────────┤│ ├────────────┤│    │
│ │ │디테일 Grid ││ │디테일 Grid ││    │
│ │ └────────────┘│ └────────────┘│    │
│ └───────────────┴───────────────┘    │
└───────────────────────────────────────┘
```

### FRMCMP 트리

```sql
CMP_TABS (루트)
  └─ CMP_TAB_PAGE (tab_index=0, label='탭이름')
       └─ CMP_SPLIT_CONTAINER
            ├─ CMP_SPLITPANEL (panel_index=0) → Grid
            └─ CMP_SPLITPANEL (panel_index=1) → Grid
  └─ CMP_TAB_PAGE (tab_index=1)
       └─ ...
```

### FRMBND 체인 — 2가지 패턴

**패턴 1: 탭별 독립 체인 (PMS100)**

```sql
-- 탭 0
ws_role  (open_sq=1, open_trigger='OPEN')  → ws_perm  (open_sq=2, open_trigger='ws_role')
-- 탭 1
ws_user  (open_sq=3, open_trigger='OPEN')  → ws_uperm (open_sq=4, open_trigger='ws_user')
```

**패턴 2: 1 마스터 → N 디테일 (EX_TABS_CHAIN)**

```sql
BND_EMP    (open_sq=1, 'OPEN', trigger_yn=true)   -- 마스터
BND_EDU    (open_sq=2, 'BND_EMP')                  -- 탭1: 학력 Grid
BND_CAREER (open_sq=3, 'BND_EMP')                  -- 탭2: 경력 Grid
BND_INFO   (open_sq=4, 'BND_EMP', bnd_type='Field') -- 탭3: 필드
```

**패턴 3: 탭별 독립 R-only (DSP010)**

```sql
ws_summary (open_sq=1, 'OPEN')  -- 모두 OPEN에 연결
ws_bottle  (open_sq=2, 'OPEN')  -- 서로 체인 관계 없음
ws_agents  (open_sq=3, 'OPEN')  -- 폼 Open 시 3개 동시 조회
```

---

## 유형 E: SearchGrid — 검색 조건 → 결과 그리드

```
대표 폼: WM100, WM400, DSP500, EX_SEARCH_GRID, COD100
ROCA 대응: D-SearchGrid, E-SearchMasterDetail
```

### 구조

```
┌──────────────────────────────────────┐
│ CMP_SPLIT_CONTAINER (vertical)       │
│ ┌──────────────────────────────────┐ │
│ │ SplitPanel 0: 검색 Panel         │ │
│ │ ┌────────┐ ┌────────┐ ┌──────┐ │ │
│ │ │inp_from│ │inp_to  │ │combo │ │ │
│ │ └────────┘ └────────┘ └──────┘ │ │
│ ├──────────────────────────────────┤ │
│ │ SplitPanel 1: 결과 Grid          │ │
│ │ ┌──────────────────────────────┐ │ │
│ │ │ EPIC_HANDSONTABLE            │ │ │
│ │ └──────────────────────────────┘ │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

### FRMBND 설정

```sql
bnd_type      = 'Grid'
open_trigger  = 'OPEN'
open_sq       = 1
save_sq       = 0 또는 1    -- 조회 전용이면 0
trigger_yn    = false
```

### BNDPULL — COMPONENT 소스 (핵심)

```sql
-- 검색 컴포넌트에서 직접 값 추출
source_type = 'COMPONENT'
source_ref  = 'inp_asset_code'   -- FRMCMP의 cmp_id
source_column = NULL              -- 컴포넌트는 단일 값이므로 불필요
```

### BNDQUERY — 선택적 WHERE `{{ }}`

```sql
'SELECT a.*, b.name
 FROM main_table a
 LEFT JOIN ref_table b ON a.fk = b.pk
 WHERE 1=1
   {{AND a.date_col >= :date_from}}
   {{AND a.date_col <= :date_to}}
   {{AND a.code = :code}}
 ORDER BY a.date_col DESC'
```

`{{ }}` 안의 조건: 파라미터가 NULL/빈값이면 엔진이 자동 제거

### 3중 체인 변형 (COD100 — 코드관리)

```sql
ws_pop    (open_sq=1, 'OPEN', trigger_yn=true)   -- 코드그룹 Grid
ws_code   (open_sq=2, 'ws_pop', trigger_yn=true) -- 코드목록 Grid
ws_sub    (open_sq=3, 'ws_code')                  -- 서브코드 Grid
```

```
ws_pop ──trigger──→ ws_code ──trigger──→ ws_sub
 (그룹)              (코드)               (서브코드)
```

---

## 유형 F: API 전용 — UI 없는 데이터 파이프라인

```
대표 폼: DSP100
ROCA 대응: 없음 (Epic 전용)
```

### 구조

```
FRMCMP 없음 (UI 없음)
BISFRM: canvas_width=0, canvas_height=0

FRMBND: bnd_type='Data', is_public=true
BNDQUERY: DO $$ PL/pgSQL 블록 $$;
```

---

## 유형 X: 특수 — Form Script / 커스텀 렌더러

```
대표 폼: WM600 (Form Script), CWK100 (커스텀), EX_INPUTER (디스플레이)
```

- WM600: BNDQUERY R이 `WHERE 1=0` (항상 빈 결과), 로직은 Form Script
- CWK100: FRMCMP 1건만, FRMBND 없음, 커스텀 렌더러가 전담
- EX_INPUTER: FRMBND 없음, 컴포넌트 쇼케이스

---

## properties JSON 공통 구조

### CMP_SPLIT_CONTAINER

```json
{
  "orientation": "horizontal|vertical",
  "split_size": 300,
  "split_mode": "pixel|ratio",
  "resizable": true,
  "divider_size": 3
}
```

### CMP_SPLITPANEL

```json
{
  "panel_index": 0,
  "showLabel": true,
  "showBorder": true,
  "borderWidth": 1,
  "headerHeight": 28,
  "showMaximize": true
}
```

### EPIC_HANDSONTABLE

```json
{
  "showToolbar": {
    "add": true, "save": true, "delete": true,
    "search": true, "refresh": true
  },
  "showFooter": true
}
```

### CMP_TABS

```json
{
  "tabPosition": "top"
}
```

### CMP_TAB_PAGE

```json
{
  "tab_index": 0,
  "label": "탭이름"
}
```

---

## ROCA → Epic 유형 매핑 요약

```
ROCA                    Epic                     비고
─────────────────────────────────────────────────────
A-FieldOnly          →  (해당 없음)              Epic에선 Grid 기반
B-SimpleGrid         →  A: SimpleGrid            단일 그리드 CRUD
C-MasterDetail       →  C: MasterDetail          상하 2 Grid, trigger 체인
C-TabForm            →  D: TabbedMasterDetail    탭별 Grid, open_sq로 순서
D-SearchGrid         →  E: SearchGrid            검색 Panel + Grid, {{ }} WHERE
E-SearchMasterDetail →  E + C 결합               검색 → 마스터 → 디테일 체인
(없음)               →  B: Field+Grid Chain      FieldSet 바인딩 (Epic 전용)
(없음)               →  F: API 전용              Data bnd_type (Epic 전용)
```
