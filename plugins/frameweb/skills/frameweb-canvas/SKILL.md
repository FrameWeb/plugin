---
name: frameweb-canvas
description: >
  폼 디자이너 Form Design 탭: 캔버스/레이아웃/컴포넌트(FRMCMP) 담당. 새 폼 전체를 0에서 만드는 진입점.
  요구사항·대화·EMAX 마이그레이션 입력을 받아 폼 유형을 결정하고 와이어프레임·SplitContainer 레이아웃·FRMCMP 컴포넌트 배치를 산출한다.
  폼 골격을 만든 뒤 frameweb-bindchain → frameweb-binding 으로 바인딩을 잇는다.
  Trigger: 폼 디자인, 폼 생성, 새 폼, 캔버스, 레이아웃, 컴포넌트 배치, frameweb-canvas,
  BISFRM, FRMCMP, SplitContainer, 와이어프레임, 폼 유형, Handsontable 툴바, customButtons.
user-invocable: true
---

# Form Canvas — 폼 디자이너 Form Design 탭 (캔버스/레이아웃/컴포넌트)

폼 디자이너의 Form Design 탭이 다루는 캔버스·레이아웃·컴포넌트(FRMCMP)를 담당한다.
**새 폼 전체를 0에서 만드는 진입점이다.** 요구사항·대화·EMAX 원본 중 하나를 입력받아 폼 유형을 결정하고, 와이어프레임과 SplitContainer 레이아웃을 설계한 뒤, FRMCMP 컴포넌트를 배치해 폼 골격(BISFRM + FRMCMP)을 만든다.

## 폼 디자이너 탭 스킬 묶음에서의 위치

이 스킬은 폼 디자이너 탭 스킬 6종(`frameweb-canvas` / `frameweb-bindchain` / `frameweb-binding` / `frameweb-script` / `frameweb-sandbox` / `frameweb-state`) 중 하나이며, 포지(Forge, Cloud Dev AI 영역)로 이식할 대상이다. 각 탭과 1:1로 맞춰져 있어 폴더 이동만으로 포지로 옮길 수 있다.

| 탭 스킬 | 담당 | 손대는 대상 |
|---------|------|-------------|
| **frameweb-canvas** (본 스킬) | Form Design 탭 — 캔버스/레이아웃/컴포넌트 | BISFRM, FRMCMP, 레이아웃 좌표 |
| frameweb-bindchain | BindChain 탭 — 컴포넌트와 바인딩 연결, 체인 순서 | frmbnd, bound_fcmp_ids |
| frameweb-binding | Binding 탭 — 조회·입력 SQL, 컬럼 매핑 | bndquery, bndpull, bndsave, bndcmp |
| frameweb-script | Script 탭 — 이벤트 핸들러 | form_script |
| frameweb-sandbox | Sandbox 탭 — 폼 전용 AI 도우미 규칙 | forge.agent.config.sandbox |
| frameweb-state | 상태 매트릭스 — 상태별 화면 칸 제어 | form_property_config.state_matrix |

**컴포넌트 도메인 세부**(렌더러 카탈로그, cmp_id별 동작, 변종 컴포넌트 신설 등)는 `epic-component` 스킬을 참조한다. frameweb-canvas는 컴포넌트를 폼에 **배치·속성 설정**하는 데까지 책임지고, 컴포넌트 자체의 신설·렌더러 계약은 epic-component 소관이다.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

## 참조 문서

- [references/form_patterns.md](references/form_patterns.md) -- 폼 유형별 통계 (ROCA 2,024개 기반), 유형별 대표 ASCII 레이아웃
- [references/component_mapping.md](references/component_mapping.md) -- ROCA->Epic 컴포넌트 매핑 사전, FRMCMP properties/레이아웃 컨테이너 패턴

> 유형별 INSERT 정답지(epic_form_patterns.md)와 MSSQL->PostgreSQL 변환 규칙(sql_conversion_rules.md)은 바인딩 단계(frameweb-binding)에서 참조한다.

## 환경

| 항목 | 값 |
|------|-----|
| Frontend | ${FRAMEWEB_APP_URL} |
| Backend | localhost:8181 |
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 패키지 설치 API | POST /api/package/install/{pkg_id} |
| 모리 공유 폴더 | /mnt/z/Obsidian/browser-test/request/ |
| 접속 정보 단일 소스 | ~/.claude/memory/db-connections.md |
| 후속 연결 | 바인딩 → frameweb-bindchain / frameweb-binding, 검증 → frameweb-verify |

---

## 입력 인터페이스

### A. frameweb-require 출력에서 (자동)

frameweb-require 스킬이 생성한 스펙 파일을 입력으로 받는다.
스펙에 Q1~Q8 답변이 포함되어 있으면 Step 1의 기획 판단을 건너뛰고 바로 Step 2(설계)로 진입한다.

```
frameweb-require 스펙 포함 항목:
  - 폼 유형 (A~F)
  - 테이블 목록 + PK/FK 관계
  - 바인딩셋 목록 + 체인 관계
  - 컬럼 목록 (타입, 필수, 입력보조, 너비)
  - Q1~Q8 답변
```

### B. 직접 대화에서 (수동)

요구사항을 직접 받아 Step 1(분석)부터 시작한다. Q1~Q8 체크리스트로 폼 유형을 결정한다.

### C. EMAX 원본에서 마이그레이션

```
입력: frm_id (예: BCA101)
소스:
  1. 캡처 이미지 -> /mnt/z/Obsidian/SHARE/roca_frame9_vb/{모듈}/{frm_id}/{frm_id}.png
  2. .Designer.vb -> 같은 폴더의 {frm_id}.Designer.vb
  3. ROCA MSSQL (<외부 MSSQL — 접속정보는 .env 참조>)
     - SCF150: 워크셋 구조 (wset_ty, grid_nm, link_grid, open_sq)
     - SCC100: 컨트롤 속성 (ctr_ty, title, tbl_nm, col_nm, popup_id, width)
     - SCW120: 필드-DB 매핑 (fld_nm, tbl_nm, col_nm, reg_no)
     - SCF170: 필드-컨트롤 바인딩 (fld_nm <-> ctr_nm)
     - SCL100: 팝업/코드 정의 (popup_id, main_cd, sub_cd)

매핑 규칙 (-> references/component_mapping.md):
  SCF150.wset_ty -> bnd_type (Grid->Grid, FreeForm->Field, SQL->Data)
  SCF150.link_grid -> open_trigger (있으면 체인, 없으면 OPEN/NULL)
  SCC100.ctr_ty -> editor_type (Text->TEXT, Combo->CODE, Date->DATE, Check->CHECKBOX)
  SCC100.popup_id -> SCL100.main_cd (code->CODE, sql->LOOKUP, popup->POPUP)
  MSSQL -> PostgreSQL 변환은 frameweb-binding 단계에서 sql_conversion_rules.md 참조
```

---

## 워크플로우

폼 디자인 전체는 5단계(분석 → 설계 → 생성 → 검증 → 수정)로 진행한다. 이 중 분석·설계·컴포넌트 배치(Step 1, 2, 2.5)가 frameweb-canvas 핵심이다. 생성·검증·수정(Step 3, 4, 5)은 바인딩까지 합쳐 폼 전체를 다루므로 frameweb-bindchain / frameweb-binding 산출물과 결합해 완성한다.

### Step 1: 분석 (What) — Q1~Q8 기획 판단 체크리스트

**frameweb-require 스펙이 있으면 이 단계를 건너뛴다.**

```
기획 판단 체크리스트 (순서대로):

  Step 1 -- 데이터 구조 -> 바인딩 결정
    Q1. 테이블 몇 개? -> 단일 Grid / 마스터-디테일 / 중첩
    Q2. 마스터 선택 방식? -> 그리드 행 / 트리 노드 / 검색만
    Q3. 디테일 표시 방식? -> 그리드(다건) / FieldSet(단건) / 둘 다

  Step 2 -- 레이아웃 -> SplitContainer 설계
    Q4. 영역 수? -> 1(Grid만) / 2(Split 1개) / 3+(중첩+탭)
    Q5. 분할 방향? -> vertical(상하) / horizontal(좌우)
    Q6. 분할 비율? -> 검색=100~140 / 목록좌=250~350 / 균등=1/2

  Step 3 -- 입력 방식 -> 컴포넌트 선택
    Q7. 필드별 입력 방식? -> TEXT / SELECT / LOOKUP / POPUP / DATE / CHECKBOX
    Q8. 검색 조건 필요? -> Panel("검색") + Input들

  Step 4 -- 폼 유형 확정 -> A / B / C / D / E / F

출력:
  - 폼 유형 (A~F, epic_form_patterns.md 기준)
  - 테이블 목록 + PK/FK 관계
  - 바인딩셋 목록 + 체인 관계
  - 컬럼 목록 (타입, 필수, 입력보조, 너비)
```

### Step 2: 설계 (How)

분석 결과를 Epic Framework 구조로 변환한다.
**이 단계에서 와이어프레임을 그리고 ido의 승인을 받은 후 Step 3로 진행한다.**

```
설계 산출물:

  1. 와이어프레임 (ASCII 레이아웃)
     -> form_patterns.md 유형별 대표 레이아웃 + epic_form_patterns.md 유형별 구조 참조
     -> 예시:
       +----------------+--------------------+
       | Grid "목록"     | FieldSet "상세"     |
       | (좌 300px)      | (우 나머지)         |
       +----------------+--------------------+

  2. SplitContainer 구조도
     -> SplitContainer (horizontal, 300)
        +-- Grid (fill)
        +-- SplitContainer (vertical, 200)
            +-- FieldSet (입력 필드들)
            +-- CMP_TABS -> [탭1 Grid, 탭2 FieldSet]

  3. 바인딩 + trigger 설계 (frameweb-bindchain 으로 이관)
     | bnd_id | bnd_type | open_trigger | open_sq | save_sq |
     |--------|----------|-------------|---------|---------|

  4. 컬럼 설계 (frameweb-binding 으로 이관)
     | data_field | data_type | editor_type | col_width | is_required |
     |-----------|-----------|-------------|-----------|-------------|

  5. 검색 영역 설계 (필요 시)
     -> 라벨:입력 2~3열, 높이 결정

  6. FieldSet 입력 배치 (필요 시)
     -> 라벨:입력 2열, 좌표 산출

  7. SQL + 파라미터 (BNDPULL + BNDSAVE, frameweb-binding 으로 이관)
     -> 누락 시 Save 실패, 반드시 포함
```

> 3·4·7번 산출물은 frameweb-bindchain / frameweb-binding 단계에서 실제 INSERT로 채운다. frameweb-canvas는 레이아웃(1·2번)과 컴포넌트 배치를 확정한다.

### Step 2.5: 컴포넌트 기본값 조회 (필수)

**forms.sql 생성 전에 반드시 실행. 하드코딩 금지 -- DB가 Single Source of Truth.**

폼 디자이너에서 드래그앤드롭하면 `cmpmst.component_default_props`가 `frmcmp.properties`에 자동 병합된다.
forms.sql을 직접 작성할 때도 이 값을 그대로 사용해야 렌더러가 정상 동작한다.

```bash
# 사용할 컴포넌트의 기본 properties 조회
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT cmp_id, component_default_props
FROM cmpmst
WHERE cmp_id IN (
  'EPIC_HANDSONTABLE', 'CMP_SPLIT_CONTAINER', 'CMP_SPLITPANEL',
  'CMP_TABS', 'CMP_TAB_PAGE', 'EPIC_TEXT_INPUT', 'EPIC_SELECT',
  'EPIC_DATEPICKER', 'EPIC_CHECKBOX', 'EPIC_BUTTON'
)
ORDER BY cmp_id;" -x
```

#### properties 규칙

| 규칙 | 설명 |
|------|------|
| **기본값 = component_default_props 전체** | 조회 결과를 frmcmp.properties에 그대로 넣는다 |
| **커스터마이즈 = 기본값 위에 오버라이드** | 변경할 키만 덮어쓴다 (나머지는 기본값 유지) |
| **하드코딩 금지** | properties를 추정하지 않는다. 반드시 DB 조회 후 사용 |

#### 컴포넌트별 커스터마이즈 가이드

| cmp_id | 기본값에서 바꿀 키 | 예시 |
|--------|-------------------|------|
| CMP_SPLIT_CONTAINER | `orientation`, `split_size`, `layout_mode->"fill"` | `{"orientation":"vertical","split_size":300,"layout_mode":"fill"}` |
| CMP_SPLITPANEL | `panel_index`, `label`, `layout_mode->"fill"` | `{"panel_index":0,"label":"목록","layout_mode":"fill"}` |
| EPIC_HANDSONTABLE | `showToolbar` (CRUD 버튼 on/off), `layout_mode->"fill"` | 읽기전용: `{"showToolbar":{"search":true,"refresh":true},"layout_mode":"fill"}` |
| CMP_TABS | 보통 기본값 그대로 | `layout_mode->"fill"` 추가 |
| CMP_TAB_PAGE | `title` (탭 라벨) | `{"title":"개요","layout_mode":"fill"}` |
| EPIC_TEXT_INPUT | `placeholder`, `label` | `{"placeholder":"검색어","label":"상태"}` |

---

## 폼 유형 분류 (Epic 기준)

> 상세 INSERT 패턴은 references/epic_form_patterns.md 참조 (frameweb-binding 단계)
> ROCA 통계는 references/form_patterns.md 참조

| 유형 | 구조 | ROCA 대응 | 난이도 |
|------|------|-----------|--------|
| **A. SimpleGrid** | Grid 1개, 루트에 fill | B-SimpleGrid, D-SearchGrid(검색 없음) | 쉬움 |
| **B. Field+Grid Chain** | 목록 Grid + 상세 FieldSet | Epic 전용 (ROCA 해당 없음) | 중간 |
| **C. MasterDetail** | 상하 2 Grid, trigger 체인 | C-MasterDetail | 중간 |
| **D. TabbedMasterDetail** | 탭으로 다중 디테일 | C-TabForm, E(탭 부분) | 높음 |
| **E. SearchGrid** | 검색 Panel + 결과 Grid | D-SearchGrid, E-SearchMasterDetail | 중간 |
| **F. API 전용** | UI 없음, Data bnd_type | Epic 전용 | 쉬움 |

---

## BISFRM (폼 마스터)

폼 헤더를 만든다. 캔버스 크기와 폼 식별자를 정의한다.

```sql
INSERT INTO bisfrm (bis_id, frm_id, frm_nm, frm_desc, status, canvas_width, canvas_height, created_by)
VALUES ('{bis_id}', 'FRM_ID', '폼이름', '설명', 'active', 1200, 600, 1)
ON CONFLICT (bis_id, frm_id) DO NOTHING;
```

| 항목 | 규칙 |
|------|------|
| canvas_width | 기본 1200px |
| canvas_height | 그리드 2개=600, 1개=400 |
| status | 'active' (즉시 사용) 또는 'draft' (설계 중) |
| frm_id | 대문자 영숫자, 모듈 접두사 (COD100, ROL100, MNU100) |

---

## FRMCMP (컴포넌트 배치)

### cmp_id 필수 확인

**절대 추정하지 말 것. DB에서 확인:**
```sql
SELECT cmp_id, cmp_nm, cmp_group FROM cmpmst ORDER BY cmp_group, cmp_id;
```

**EPIC 컴포넌트를 기본으로 사용한다.** (references/component_mapping.md 참조)

| cmp_id | 용도 | cmp_group |
|--------|------|-----------|
| **EPIC_HANDSONTABLE** | **그리드 (기본)** | Display |
| **EPIC_TEXT_INPUT** | **텍스트 입력** | Input |
| **EPIC_SELECT** | **드롭다운** | Input |
| **EPIC_DATEPICKER** | **날짜** | Input |
| **EPIC_CHECKBOX** | **체크박스** | Input |
| **EPIC_MULTISELECT** | **멀티셀렉트** | Input |
| **EPIC_MULTICHECK** | **멀티체크** | Input |
| **EPIC_BUTTON** | **버튼** | Action |
| **EPIC_IMAGE** | **이미지** | Input |
| **EPIC_TREEVIEW** | **트리뷰** | Display |
| **EPIC_FTP_GRID** | **FTP 그리드** | Display |
| CMP_SPLIT_CONTAINER | 분할 컨테이너 | Container |
| CMP_SPLITPANEL | 분할 패널 (자동) | Container |
| CMP_PANEL | 패널 (타이틀용) | Container |
| CMP_TABS | 탭 컨트롤 | Container |
| CMP_TAB_PAGE | 탭 페이지 | Container |

> 컴포넌트 신설·렌더러 동작·변종 컴포넌트 등록은 epic-component 스킬을 참조한다.

### 레이아웃 규칙: SplitContainer 기반

**영역 분할은 반드시 SplitContainer를 사용한다.**

| 컴포넌트 | 역할 | 특성 |
|----------|------|------|
| SplitContainer | 영역을 2개로 분할 | orientation, splitterDistance |
| SplitPanel | SplitContainer에서 **자동 생성**. 100% 부모 채움 | 별도 배치 불가, 항상 fill |
| Panel | 그룹핑/타이틀용 | 위치값 지정 가능, 또는 fill |

### 핵심 규칙

| 규칙 | 설명 |
|------|------|
| SplitPanel 안에 fill Panel 금지 | SplitPanel이 이미 100%이므로 fill Panel 중복 |
| Grid는 SplitPanel에 직접 배치 | Panel 없이 Grid를 SplitPanel 안에 넣을 수 있음 |
| Panel은 타이틀 필요 시만 사용 | 그룹 제목("코드분류" 등)이 필요하면 Panel 안에 Grid |
| SplitContainer 중첩 가능 | SplitContainer > SplitPanel > SplitContainer > SplitPanel > ... |
| Input/Action | 항상 layout_mode='none' (절대 좌표) |
| parent_id | SplitPanel의 parent_id = SplitContainer. 자식의 parent_id = SplitPanel |
| fcmp_id 명명 | split_xxx, panel_xxx, grid_xxx, inp_xxx, btn_xxx |

### orientation 규칙

```
horizontal -> flex-row  -> 좌우 분할 (LEFT | RIGHT)
vertical   -> flex-col  -> 상하 분할 (TOP / BOTTOM)
```

### 탭 레이아웃 (CMP_TABS + CMP_TAB_PAGE)

**탭 = CMP_TABS -> CMP_TAB_PAGE -> 자식 컴포넌트** 3계층 구조.

| 규칙 | 설명 |
|------|------|
| text | 탭 라벨 (TabsRenderer가 이 값으로 탭 바 생성) |
| sort_order | 탭 표시 순서 (0부터) |
| selectedTabPage | 기본 활성 탭 (CMP_TAB_PAGE의 fcmp_id) |
| padding:"none" | 내부 여백 제거 (Grid/SplitContainer 채움용) |
| hideTitle:true | 탭 바에 이미 라벨 -> 본문 중복 타이틀 숨김 |

### 체인 버튼 vs 그리드 툴바 (2개 별개 시스템)

폼 상단에 2종류 버튼이 있다. 혼동 금지:

| 구분 | 위치 | 버튼 | 제어 |
|------|------|------|------|
| **체인 버튼** | 폼 우상단 | Open / New / Save / Delete / Print | FRMBND (open_trigger, save_sq, new_yn) |
| **그리드 툴바** | 그리드 내부 상단 | + / save / delete / refresh / Search | frmcmp.properties.showToolbar |

- **체인 버튼 Open**: FRMBND.open_trigger='OPEN' -> R 쿼리 실행
- **체인 버튼 Save**: FRMBND.save_sq > 0인 바인딩의 C/U/D 일괄 실행
- **체인 버튼 New**: FRMBND.new_yn='Y'일 때만 활성 (FieldSet 전용)
- **그리드 툴바 +**: 그리드에 빈 행 추가 (GridSet은 이걸로 행 추가)
- **그리드 툴바 save**: 개별 그리드 저장 (체인 Save와 별도)

> 체인 버튼의 동작 정의(open_trigger/save_sq/new_yn 설정)는 frameweb-bindchain 소관이다. frameweb-canvas는 버튼의 배치·구분만 다룬다.

### Handsontable 그리드 속성 — 내장 툴바 (properties.tb_*)

**tb_* 키는 boolean. 컴포넌트 단위로 개별 토글한다.** (구버전 `showToolbar` JSON 객체는 폐기)

| 키 | 기본값 | 설명 |
|----|--------|------|
| tb_search | true | 검색 입력 |
| tb_add | true | 행 추가(+) |
| tb_save | false | 저장 |
| tb_delete | false | 행 삭제 |
| tb_copy | false | 행 복제 |
| tb_refresh | false | 새로고침 |
| tb_fullscreen | true | 전체화면 |
| tb_play | false | 실행 |

```sql
-- CRUD 폼 그리드 예시 (BNK_HANDSONTABLE)
properties = '{"tb_add":true,"tb_save":true,"tb_delete":true,"tb_search":true,"tb_refresh":true}'::jsonb

-- 읽기 전용 (조회 그리드)
properties = '{"tb_add":false,"tb_save":false,"tb_delete":false,"tb_search":true,"tb_refresh":true}'::jsonb
```

### Handsontable 그리드 — 사용자 정의 툴바 버튼 (properties.customButtons)

내장 7+1개 외에 폼 작성자가 추가 버튼을 정의할 수 있다. 클릭 시 form_script의 함수가 실행된다.

```sql
-- 사용자 정의 버튼 3개 (발송/다운로드/업로드)
UPDATE frmcmp SET properties = properties || '{
  "customButtons": [
    {"label":"발송","onClick":"onSendSms","variant":"primary","icon":"Send"},
    {"label":"샘플 다운로드","onClick":"onDownloadSample","variant":"default","icon":"Download"},
    {"label":"CSV 업로드","onClick":"onUploadCsv","variant":"default","icon":"Upload"}
  ]
}'::jsonb
WHERE bis_id='bnk' AND frm_id='SMS100' AND fcmp_id='grid_recipients';
```

| 키 | 필수 | 설명 |
|----|------|------|
| label | 필수 | 버튼 텍스트 (툴팁 + 표시) |
| onClick | 필수 | form_script의 `registerNativeFunction(이름, 함수)` 함수 이름 |
| variant | 선택 | `default`(회색) / `primary`(강조) / `danger`(빨강). 기본 default |
| icon | 선택 | **Lucide 아이콘 이름**. kebab-case(`send-horizontal`, `download`) 또는 PascalCase(`SendHorizontal`, `Download`) 모두 허용. lucide.dev 사이트의 이름 그대로 복사해서 사용 가능. 미지정 시 텍스트만 |

폼 디자이너의 그리드 속성 패널 "사용자 정의 버튼" 입력란 우측 [sample] 버튼을 누르면 위 JSON 형식이 자동 채워진다.

#### Lucide 아이콘 카탈로그 (자주 쓰는)
이름은 kebab-case / PascalCase 둘 다 사용 가능.
- 발송/전송: `send` `send-horizontal` `mail`
- 다운로드/내보내기: `download` `file-down` `file-spreadsheet`
- 업로드/가져오기: `upload` `file-up`
- 새로고침/조회: `refresh-cw` `search`
- 추가/삭제: `plus` `trash-2`
- 인쇄/공유: `printer` `share-2`
- 전체 목록: https://lucide.dev/icons/ (사이트의 이름 그대로 복사)

#### form_script 함수 등록 — registerNativeFunction

사용자 정의 버튼 클릭 시 호출되는 함수는 **메인 스레드 권한이 필요한 작업**(파일 다운로드/업로드, window 접근)에 적합하므로 `registerNativeFunction`을 사용한다.

```javascript
// 폼 디자이너 → 스크립트 탭에서 작성 후 저장
registerNativeFunction('onSendSms', (ctx) => {
  // ctx = { fcmpId, selectedRowIndex, selectedRow }
  console.log('[onSendSms]', ctx);
  // 실 동작: API 호출, 그리드 데이터 후처리 등
});

registerNativeFunction('onDownloadSample', () => {
  // CSV Blob 다운로드 (메인 스레드 권한 필요 → Native)
  const csv = '수신번호,수신자명,언어\n010-0000-0000,홍길동,ko';
  const blob = new Blob([csv], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'sample.csv'; a.click();
  URL.revokeObjectURL(url);
});
```

**registerNativeFunction vs registerFunction** — 그리드 사용자 정의 버튼은 native가 정합. 표준 sandboxAPI(form/workset/grid/util)만 쓰는 경우엔 registerFunction도 가능.

> form_script 작성·등록의 상세는 frameweb-script 스킬을 참조한다. frameweb-canvas는 버튼 정의(customButtons)와 함수 이름 연결까지 다룬다.

#### 작동 흐름
1. frmcmp.properties.customButtons에 버튼 정의
2. bisfrm.form_script에 registerNativeFunction 함수 등록
3. **재배포 필수** (POST /api/forms/deploy/) — ERP는 비즈 DB FRMMST 스냅샷을 본다
4. 그리드 툴바에 버튼 노출 → 클릭 → 함수 실행

#### 체인 버튼 (폼 우상단) vs 그리드 툴바 — 혼동 금지

| 구분 | 위치 | 버튼 | 제어 |
|------|------|------|------|
| **체인 버튼** | 폼 우상단 | Open / New / Save / Delete / Print | FRMBND (open_trigger, save_sq, new_yn) |
| **그리드 내장 툴바** | 그리드 좌상단 | + / save / delete / refresh / search 등 | frmcmp.properties.tb_* |
| **그리드 사용자 정의 버튼** | 그리드 우상단 (전체화면 옆) | 임의 | frmcmp.properties.customButtons + form_script |

- **체인 버튼 Open**: FRMBND.open_trigger='OPEN' → R 쿼리 실행
- **체인 버튼 Save**: FRMBND.save_sq > 0 바인딩의 C/U/D 일괄 실행
- **체인 버튼 New**: FRMBND.new_yn='Y'일 때만 활성 (FieldSet 전용)
- **그리드 내장 + (tb_add)**: 그리드에 빈 행 추가
- **그리드 내장 save (tb_save)**: 개별 그리드 저장 (체인 Save와 별도)
- **그리드 사용자 정의 버튼**: form_script 함수 호출 (자유 동작)

---

## FRMCMP 필수 설정 규칙

### frmcmp display_order/depth NULL 금지

```yaml
문제: display_order가 NULL이면 캔버스에서 자식 순서 불확정, depth가 0이면 계층 인식 불가
규칙:
  root(SplitContainer 등): depth=0, display_order=0
  패널(SplitPanel 등):     depth=1, display_order=0,1 (패널 순서)
  자식(Grid, Field 등):    depth=2, display_order=0,1,2... (패널 내 순서)
```

> bndcmp.is_pk 등 컬럼 매핑 설정은 frameweb-binding 소관이다.

---

## forms.sql 생성/실행 주의사항

폼 골격(BISFRM + FRMCMP)을 만든 뒤 바인딩(frameweb-bindchain / frameweb-binding)까지 합쳐 완전한 forms.sql을 생성한다. 전체 INSERT 순서와 실행 주의사항은 공통이다.

### INSERT 순서 (FK 의존성)

```
BISFRM -> FRMCMP(FK->CMPMST) -> FRMBND -> BNDCMP(FK->FRMBND) -> BNDQUERY -> BNDPULL -> BNDSAVE
```

모든 INSERT에 `ON CONFLICT DO NOTHING` 필수 (재설치 안전).

### exec_driver_sql 필수

BNDQUERY query 값에 `:param`이 포함되어 SQLAlchemy `text()`의 bind parameter 파싱과 충돌.
PackageService는 `exec_driver_sql()`로 raw 실행.

### {bis_id} 플레이스홀더

forms.sql의 모든 bis_id 값을 `'{bis_id}'`로 작성. 설치 시 실제 값으로 치환됨.

### 시스템 DB에서 실행

BISFRM, FRMCMP, FRMBND, BNDCMP, BNDQUERY, BNDPULL, BNDSAVE은 모두 시스템 DB 테이블.

---

## 표준 폼 템플릿 (약식)

레이아웃 골격 참고용. FRMBND/BNDQUERY/BNDPULL/BNDSAVE 부분은 frameweb-bindchain / frameweb-binding 단계에서 채운다.

### 마스터-디테일
```
BISFRM: 1200x600
FRMCMP: SplitContainer(vertical,250) > [grid_master, grid_detail]
FRMBND: ws_master(Grid,OPEN,sq=1) + ws_detail(Grid,ws_master,sq=2)
BNDCMP: ws_master[...] + ws_detail[fk(hidden),...]
BNDQUERY: ws_master[R,C,U,D] + ws_detail[R,C,U,D]
BNDPULL: ws_detail.R.fk <- GRID(ws_master.pk)
BNDSAVE: ws_master[일반+SESSION] + ws_detail[FK->ws_master, 일반+SESSION]
```

### 단일 그리드 CRUD
```
BISFRM: 1200x400
FRMCMP: grid_main(fill)
FRMBND: ws_main(Grid,OPEN,sq=1)
BNDCMP: ws_main[col1,col2,...]
BNDQUERY: ws_main[R,C,U,D]
BNDPULL: ws_main.R.co_cd <- SESSION
BNDSAVE: ws_main[일반+SESSION]
```

### 선택-매트릭스
```
BISFRM: 1200x600
FRMCMP: SplitContainer(horizontal,300) > [grid_select, grid_matrix]
FRMBND: ws_select(Grid,OPEN,sq=1,save_sq=0) + ws_matrix(Grid,ws_select,sq=2)
BNDCMP: ws_select[pk,label] + ws_matrix[pk(hidden),...]
BNDQUERY: ws_select[R] + ws_matrix[R(LEFT JOIN+COALESCE), C/U(UPSERT)]
BNDPULL: ws_matrix.R.pk <- GRID(ws_select.pk)
BNDSAVE: ws_matrix[일반+SESSION]
```

---

## 참고 폼 카탈로그

새 폼을 설계할 때 PACKAGE bis_id의 가이드 폼을 참조 소스로 사용한다.
DB에서 직접 조회해 실제 동작하는 메타데이터를 확인한다. 폼 유형별로 어느 폼을 본보기로 삼을지 아래 표에서 고른다.

### 가이드 폼 목록 (PACKAGE bis)

| frm_id | 폼 이름 | 패턴 | 핵심 학습 포인트 |
|--------|---------|------|-----------------|
| EX_FIELD_CRUD | FieldSet CRUD | 단일 FieldSet | 기본 CRUD, SESSION 파라미터 |
| EX_FTP_GRID | FTP Grid Guide | FTP 단독 | EPIC_FTP_GRID, UPSERT, SaveRegistry |
| EX_IMAGE_GUIDE | Image Guide | 마스터-디테일 + 이미지 | EPIC_IMAGE, 체인, bndpull 크로스 참조 |
| COD100 | 코드관리 | 마스터-디테일 | SplitContainer, Observer Chain |

> FTP/Image 컴포넌트는 컴포넌트 자체가 API를 직접 호출하고 BindingSet은 추가 정보(rmks 등)만 저장한다. 멀티파트 업로드(파일 바이너리 직접 전송)라는 특수 사례에만 적용한다. 일반 외부 API(날씨/환율/주가 같은 JSON 응답)는 BNDQUERY `source_type='API'`가 표준 — `frameweb-binding` 스킬 참조.

### 참조 방법 (시스템 DB 조회 SQL)

특정 폼의 전체 메타를 조회해 레이아웃·컴포넌트·바인딩 구조를 확인한다.

```sql
SELECT * FROM bisfrm   WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID';
SELECT * FROM frmcmp   WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID' ORDER BY display_order;
SELECT * FROM frmbnd   WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID';
SELECT * FROM bndcmp   WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID' ORDER BY bnd_id, col_seq;
SELECT * FROM bndquery WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID' ORDER BY bnd_id, crudm;
SELECT * FROM bndpull  WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID';
SELECT * FROM bndsave  WHERE bis_id='PACKAGE' AND frm_id='EX_FTP_GRID';
```

`frm_id` 값만 위 표의 다른 폼으로 바꾸면 해당 패턴의 실측 메타를 확인한다.

---

## 실행 예시 — frameweb-canvas 산출물 (BISFRM + FRMCMP)

frameweb-canvas 단계의 산출물은 폼 골격(BISFRM + FRMCMP)이다. COD100(코드관리, 마스터-디테일) 기준 골격 INSERT 예시는 다음과 같다. `properties` 값은 Step 2.5에서 `cmpmst.component_default_props`를 조회해 채운다.

```sql
-- BISFRM (폼 마스터)
INSERT INTO bisfrm (bis_id, frm_id, frm_nm, frm_desc, status, canvas_width, canvas_height, created_by)
VALUES ('{bis_id}', 'COD100', '코드관리', '코드분류 + 기초코드 마스터-디테일', 'active', 1200, 600, 1)
ON CONFLICT (bis_id, frm_id) DO NOTHING;

-- FRMCMP: SplitContainer(vertical, 250) > [panel_top > grid_master, panel_bottom > grid_detail]
INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, layout_mode, properties, display_order, c_id)
VALUES ('{bis_id}', 'COD100', 'split_main', 'CMP_SPLIT_CONTAINER', 0, 0, 1200, 600, 'fill',
  '{"orientation":"vertical","split_size":250,"layout_mode":"fill"}', 0, 1)
ON CONFLICT (bis_id, frm_id, fcmp_id) DO NOTHING;

INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, layout_mode, parent_id, properties, display_order, c_id)
VALUES ('{bis_id}', 'COD100', 'panel_top', 'CMP_SPLITPANEL', 0, 0, 1200, 250, 'fill', 'split_main',
  '{"label":"코드분류","panel_index":0,"layout_mode":"fill"}', 1, 1)
ON CONFLICT (bis_id, frm_id, fcmp_id) DO NOTHING;

INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, layout_mode, parent_id, properties, display_order, c_id)
VALUES ('{bis_id}', 'COD100', 'panel_bottom', 'CMP_SPLITPANEL', 0, 0, 1200, 350, 'fill', 'split_main',
  '{"label":"기초코드","panel_index":1,"layout_mode":"fill"}', 2, 1)
ON CONFLICT (bis_id, frm_id, fcmp_id) DO NOTHING;

INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, layout_mode, parent_id, properties, display_order, c_id)
VALUES ('{bis_id}', 'COD100', 'grid_master', 'EPIC_HANDSONTABLE', 0, 0, 1200, 250, 'fill', 'panel_top',
  '{"tb_add":true,"tb_save":true,"tb_delete":true,"tb_search":true,"tb_refresh":true,"layout_mode":"fill"}', 3, 1)
ON CONFLICT (bis_id, frm_id, fcmp_id) DO NOTHING;

INSERT INTO frmcmp (bis_id, frm_id, fcmp_id, cmp_id, x, y, width, height, layout_mode, parent_id, properties, display_order, c_id)
VALUES ('{bis_id}', 'COD100', 'grid_detail', 'EPIC_HANDSONTABLE', 0, 0, 1200, 350, 'fill', 'panel_bottom',
  '{"tb_add":true,"tb_save":true,"tb_delete":true,"tb_search":true,"tb_refresh":true,"layout_mode":"fill"}', 4, 1)
ON CONFLICT (bis_id, frm_id, fcmp_id) DO NOTHING;
```

> 위 골격에 이어지는 FRMBND / BNDCMP / BNDQUERY / BNDPULL / BNDSAVE까지 합친 완전한 COD100 forms.sql 예시는 `frameweb-binding` 스킬의 "완전한 forms.sql 예시: 마스터-디테일" 절을 참조한다. INSERT 순서는 BISFRM → FRMCMP → FRMBND → BNDCMP → BNDQUERY → BNDPULL → BNDSAVE이며, 모든 INSERT에 `ON CONFLICT DO NOTHING`을 붙인다.

---

## 주요 속성(storage='column')은 정규 컬럼에 — properties JSONB 금지

FRMCMP를 생성·수정할 때, 각 속성을 frmcmp의 정규 컬럼(예: `frmcmp.label`)에 넣을지 `frmcmp.properties` JSONB에 넣을지는 cmpmst가 정한다. 추정으로 결정하지 않는다.

### 저장 위치는 cmpmst가 정한다

cmpmst의 `component_props_schema`에서 각 속성은 `storage` 메타값을 가진다.

| storage 값 | 저장 위치 |
|------------|-----------|
| `'column'` | frmcmp의 정규 컬럼 (예: `frmcmp.label`) |
| 그 외 또는 빈 값 | `frmcmp.properties` JSONB |

분기 지점: frontend `PropertyPanel.tsx`의 storage 분기, backend `forms.py update_form_component_v2`.

패널(SPLITPANEL)·탭페이지(TAB_PAGE)·버튼·입력 등 대부분 컴포넌트의 `label`(제목/라벨)은 `storage='column'`이다. 즉 **제목은 `frmcmp.label` 정규 컬럼에 넣는다.**

### 잘못 넣으면 — 코드 노출 + 다국어 추출 누락

제목·라벨을 정규 컬럼(`frmcmp.label`)이 아니라 `properties` JSONB(예: `properties.label`, `properties.title`)에 넣으면, 렌더러가 정규 컬럼을 우선 읽는다. 결과:

- 화면에 제목 대신 컴포넌트 id(코드)가 노출된다.
- extract(키 가져오기)도 그 제목을 가져오지 못해 다국어 추출이 누락된다.

확인된 사례: SMS300 패널 `panel_search`/`panel_result`의 제목이 `properties.label`에 들어가 화면에 코드 노출 + 다국어 추출 누락.

### 규칙

1. 각 속성의 저장 위치는 cmpmst `component_props_schema`의 `storage` 값을 따른다. `storage='column'`인 속성(대표적으로 `label`)은 **frmcmp의 정규 컬럼**에 넣는다.
2. 제목·라벨 같은 주요 속성을 `properties` JSONB에 중복으로 넣거나, 정규 컬럼을 비운 채 properties에만 넣지 않는다.
3. `properties` JSONB에는 storage가 'column'이 아닌 보조 속성(예: 패널의 `showLabel`, `showMaximize`, `padding` 등)만 넣는다.

### 저장 위치 확인 SQL

특정 속성의 storage를 모르면 그 컴포넌트 타입의 schema를 조회한다.

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT component_props_schema FROM cmpmst WHERE cmp_id='{타입}';" -x
```

조회 결과에서 해당 필드의 `storage` 값을 확인한다.

### 연계

이 규칙은 아래 "라벨 빈 컴포넌트 점검" 절과 한 쌍이다. 라벨이 비어 화면에 코드가 노출될 때, 정규 컬럼이 비고 properties에만 제목이 든 경우(잘못 입력)도 같이 의심한다.

관련 메모리: `frmcmp-column-vs-properties` — storage:column 속성을 properties에 넣으면 Canvas+Runtime 렌더러가 둘 다 빈값.

---

## 라벨 빈 컴포넌트 점검 — 코드 노출 방지 + 적당한 라벨 제안·입력

폼을 만든 뒤, 제목을 표시하는 컴포넌트의 label이 비어 있는지 점검한다. 비어 있으면 런타임 화면에 컴포넌트 id가 그대로 노출된다.

### 증상

제목/텍스트를 표시하는 컴포넌트(컨테이너·패널·탭·그리드·버튼)의 label이 비어 있으면, 런타임 화면에 컴포넌트 id(fcmp_id, 예: `panel_left`)가 그대로 제목 자리에 표시된다. 사용자에게 의미 없는 코드가 제목으로 보이는 결함이다.

### 원인 — 렌더러 폴백

렌더러가 label이 없을 때 fcmp_id로 폴백한다. 확인된 폴백 지점:

- `shared/packages/renderers/src/transformer.ts:130` — `label: comp.label || comp.fcmp_id`
- `shared/packages/renderers/src/canvas/base/rectText.tsx:46` — `displayText = fcmp.label || fcmp.fcmp_id`
- `shared/packages/renderers/src/canvas/container/bnk_split_container.tsx`, `cmp_split_container.tsx` — 패널 영역 id 폴백

입력 필드(예: `*_INPUT`)는 label이 비어도 입력칸만 표시되므로 코드 노출 대상이 아니다 — 점검에서 제외한다.

### 코드 노출 대상 타입

제목 표시형 컴포넌트만 점검 대상으로 한다.

- cmp_id에 `SPLIT_CONTAINER` / `SPLITPANEL` / `PANEL` / `TABS`를 포함하는 컨테이너·패널·탭 계열
- 제목을 표시하는 그리드(`HANDSONTABLE`), 버튼(`BUTTON`)
- BNK_, CMP_, EPIC_ 등 접두어 변종 모두 해당

### 점검 SQL

특정 폼 점검:

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT fcmp_id, cmp_id, parent_id, label
FROM frmcmp
WHERE bis_id='{bis_id}' AND frm_id='{frm_id}'
  AND COALESCE(label,'')=''
  AND cmp_id ~* 'PANEL|CONTAINER|TABS|HANDSONTABLE|BUTTON'
ORDER BY parent_id NULLS FIRST, fcmp_id;"
```

비즈 전체 점검은 frm_id 조건을 빼고 `GROUP BY frm_id, cmp_id`로 집계한다. (접속 정보 단일 소스: ~/.claude/memory/db-connections.md / 로컬 시스템 DB ${FRAMEWEB_DB_NAME})

### 적당한 라벨 제안 규칙

fcmp_id만으로 단정하지 않는다. 세 단서를 함께 본다.

1. **fcmp_id 어휘** — `panel_left`(좌측 패널), `grid_results`(결과 그리드), `btn_send`(발송 버튼)처럼 id가 역할을 담은 경우가 많다.
2. **컴포넌트 위치/내용** — parent_id로 계층을 보고, 그 패널이 담은 자식(그리드의 바인딩 데이터, 형제 입력 필드의 라벨)으로 무엇을 보여주는 영역인지 추론한다.
3. **폼 이름 맥락** — 폼이 무슨 업무인지(예: SMS 발송)로 제목 어휘를 맞춘다.

예시 — SMS400 "[품의 실행] SMS 발송":

| fcmp_id | 내용 | 제안 라벨 |
|---------|------|-----------|
| panel_left | 좌측 배치 목록 그리드 | 발송 내역 |
| panel_summary | 처리상태·건수·발송 버튼 | 처리 현황 |
| panel_right_bottom | 결과 그리드 | 발송 상세 |

제안한 라벨은 폼 단위로 묶어 사용자에게 보여주고 확인받은 뒤 입력한다. 의미가 명확한 경우 추천안으로 제시한다.

### 입력 + 재배포

```bash
# 1) 라벨 입력 (시스템 DB의 frmcmp)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
UPDATE frmcmp SET label='{제안 라벨}' WHERE bis_id='{bis_id}' AND frm_id='{frm_id}' AND fcmp_id='{fcmp_id}';"

# 2) 재배포 — ERP 런타임은 비즈 DB의 frmmst 배포 스냅샷을 읽으므로 재배포 필수
curl -X POST localhost:8181/api/frameweb-deploy/deploy -H 'Content-Type: application/json' -d '{"bis_id":"{bis_id}","frm_id":"{frm_id}"}'
```

frmcmp만 고치고 재배포하지 않으면 ERP 화면은 옛 스냅샷을 계속 읽어 변화가 없다(메모리 form-deploy-required-after-bisfrm-edit).

### 다국어는 별도 단계

폼 라벨 다국어는 이 절차와 분리된 작업이다. 런타임은 `t('{frm_id}.{fcmp_id}', { defaultValue: label })`로 번역하므로, 다국어가 필요하면 langtxt에 `{frm_id}.{fcmp_id}` 키로 KO/EN/LA를 등록한다(번역관리 화면 = LanguageManager). 라벨 입력(이 절)과 번역 등록은 별개 단계로 처리한다.

---

## 검증 (모리 위임)

WSL에서 Chrome 접근 불가 -> 모리에게 위임. 폼 전체 렌더링·레이아웃·체인·CRUD를 검증한다.

```
1. POST /api/package/install/{pkg_id} (bis_id 헤더) -- 다오가 curl로 실행
2. 모리 테스트 지시서 생성 -> /mnt/z/Obsidian/browser-test/request/{frm_id}_verify.md
3. 모리가 브라우저에서 검증:
   - [ ] 폼이 FormPage에서 렌더링되는가
   - [ ] SplitContainer 레이아웃이 올바른가 (상하/좌우, 비율)
   - [ ] R(조회) 데이터가 표시되는가
   - [ ] Observer Chain (마스터 선택 -> 디테일 갱신)
   - [ ] C(등록) / U(수정) / D(삭제) 동작하는가
4. 모리 결과 파일 확인
```

검증 실패 시:
- **단순 수정** (SQL 오타, 컬럼 누락): forms.sql 수정 -> 재설치
- **구조적 문제** (파라미터, SET 매핑, trigger): frameweb-bindchain / frameweb-binding 으로 연계

```
DB 직접 수정 (forms.sql 재설치 없이):
  PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "SQL문;"

재설치:
  curl -X POST localhost:8181/api/package/install/{pkg_id} -H "X-Bis-Id: {bis_id}"
```

> 런타임 검증(폼 열기 테스트, CRUD 테스트의 모리 위임)은 frameweb-verify 스킬이 흡수한다. 검증 자체의 의뢰서 작성·결과 판단은 frameweb-verify 참조.

---

## 후속 연결

폼 골격(BISFRM + FRMCMP)을 만든 뒤 다음 순서로 바인딩을 잇는다.

```
frameweb-canvas (캔버스/레이아웃/컴포넌트)
   ↓
frameweb-bindchain (컴포넌트와 바인딩 연결, 체인 순서/Observer 구성)
   ↓
frameweb-binding (조회·입력 SQL, 컬럼 매핑, 조회/저장 파라미터)
   ↓
frameweb-verify (모리 런타임 검증 — 폼 열기, CRUD)
```

- **컴포넌트 도메인 세부** → epic-component 스킬
- **이벤트 핸들러** → frameweb-script 스킬
- **상태별 화면 칸 제어** → frameweb-state 스킬
