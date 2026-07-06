---
name: frameweb-require
description: >
  폼 라이프사이클 1단계: 요구사항 수집. 자연어/스크린샷/ROCA 참조에서 구조화된 폼 스펙을 생성.
  frameweb-canvas의 Q1~Q8을 자동 답변할 수 있는 수준의 스펙 출력.
  Trigger: 폼 요구사항, 요구사항 수집, 폼 스펙, frameweb-require, 새 폼 기획,
  어떤 폼 만들지, 폼 설계 시작, 폼 만들어줘.
user-invocable: true
---

# Form Require -- 폼 라이프사이클 1단계: 요구사항 수집

자연어, 스크린샷, ROCA 참조 등 다양한 입력에서 **frameweb-canvas/frameweb-binding이 바로 소비할 수 있는 구조화된 폼 스펙**을 생성한다.
**목표: Q1~Q8 답변이 포함된 YAML 스펙 출력.**

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

## 참조 문서

- [references/form_patterns.md](references/form_patterns.md) -- ROCA 2,024개 폼 유형별 통계

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 비즈니스 DB | bisdb 테이블에서 host/port/database/username 조회 |
| ROCA 자료 | /mnt/z/Obsidian/SHARE/roca_frame9_vb/ |
| 후속 스킬 | frameweb-canvas (레이아웃) / frameweb-binding (바인딩) |

---

## 입력 3가지

### A. 자연어 요구사항

```
예시: "직원 정보 관리 폼", "코드 분류 + 기초코드 마스터-디테일"
처리:
  1. 키워드에서 테이블/관계 추론
  2. Q1~Q8 체크리스트 판단
  3. 유형 결정 (A~F)
```

### B. 스크린샷 분석

```
처리:
  1. 이미지에서 레이아웃 영역 식별 (그리드, 필드, 탭, 검색 패널)
  2. 분할 방향/비율 추정
  3. 컬럼 목록 추출 (헤더 텍스트)
  4. 입력 컴포넌트 종류 식별 (텍스트, 드롭다운, 날짜, 체크박스)
```

### C. ROCA 참조

기존 ROCA 폼에서 유사 구조를 참고한다.

```
검색 방법:
  1. Grep으로 _analysis.md 검색
     Grep(pattern="키워드", path="/mnt/z/Obsidian/SHARE/roca_frame9_vb/", glob="*_analysis.md")
  2. 유사 폼의 구조 참고 (WorkSet 수, 그리드 수, 컬럼 매핑)
  3. form_patterns.md의 유형별 평균과 비교하여 복잡도 판단
```

---

## Q1~Q8 체크리스트 (자동 판단)

### Step 1 -- 데이터 구조 -> 바인딩 결정

| 질문 | 판단 기준 | 결과 |
|------|----------|------|
| **Q1. 테이블 몇 개?** | 1개=단일 Grid, 2개=마스터-디테일, 3+=중첩 | bnd_type 결정 |
| **Q2. 마스터 선택 방식?** | 그리드 행/트리 노드/검색만 | 마스터 UI 결정 |
| **Q3. 디테일 표시 방식?** | Grid(다건)/Field(단건)/둘 다 | 디테일 bnd_type |

### Step 2 -- 레이아웃 -> SplitContainer 설계

| 질문 | 판단 기준 | 결과 |
|------|----------|------|
| **Q4. 영역 수?** | 1=Grid만, 2=Split 1개, 3+=중첩+탭 | SplitContainer 수 |
| **Q5. 분할 방향?** | vertical(상하)/horizontal(좌우) | orientation |
| **Q6. 분할 비율?** | 검색=100~140, 목록좌=250~350, 균등=1/2 | split_size |

### Step 3 -- 입력 방식 -> 컴포넌트 선택

| 질문 | 판단 기준 | 결과 |
|------|----------|------|
| **Q7. 필드별 입력 방식?** | 문자열=TEXT, 코드=CODE/SUBCODE, 참조=LOOKUP/POPUP, 날짜=DATE, 불리언=CHECKBOX | editor_type |
| **Q8. 검색 조건 필요?** | 있으면 Panel("검색")+Input 구성 | 검색 영역 |

### Step 4 -- 폼 유형 확정

form_patterns.md의 유형별 통계를 참조하여 A~F 중 결정.

| 유형 | 구조 | ROCA 대응 | 빈도 |
|------|------|-----------|------|
| **A. SimpleGrid** | Grid 1개, 루트에 fill | B-SimpleGrid, D-SearchGrid(검색 없음) | 4.4% |
| **B. Field+Grid Chain** | 목록 Grid + 상세 FieldSet | Epic 전용 | -- |
| **C. MasterDetail** | 상하 2 Grid, trigger 체인 | C-MasterDetail | 9.2% |
| **D. TabbedMasterDetail** | 탭으로 다중 디테일 | C-TabForm, E(탭 부분) | 0.0% |
| **E. SearchGrid** | 검색 Panel + 결과 Grid | D-SearchGrid, E-SearchMasterDetail | 60.9%+15.2% |
| **F. API 전용** | UI 없음, Data bnd_type | Epic 전용 | -- |

---

## 출력: 구조화된 YAML 스펙

```yaml
# frameweb-require 출력 스펙
frm_id: "FRM_ID"
frm_nm: "폼 이름"
form_type: "C"  # A~F
reference_form: "COD100"  # 참고한 기존 폼 (있으면)

# Q1~Q8 답변
q1_tables:
  - name: code_group
    role: master
    pk: [co_cd, grp_cd]
  - name: code_detail
    role: detail
    pk: [co_cd, grp_cd, sub_cd]
    fk: {grp_cd: code_group.grp_cd}
q2_master_selection: grid_row  # grid_row / tree_node / search_only
q3_detail_display: grid  # grid / field / both
q4_area_count: 2
q5_split_direction: vertical  # vertical / horizontal
q6_split_size: 250
q7_fields:
  - {name: grp_cd, type: string, editor: TEXT, required: true, width: 100}
  - {name: grp_nm, type: string, editor: TEXT, required: true, width: 200}
  - {name: use_yn, type: boolean, editor: CHECKBOX, required: false, width: 60}
q8_search_conditions: []  # [{name: grp_nm, editor: TEXT, label: "분류명"}]

# 바인딩 설계
bindings:
  - bnd_id: ws_master
    bnd_type: Grid
    table: code_group
    trigger: {trigger_yn: false, open_trigger: "OPEN"}
    crud: [R, C, U, D]
  - bnd_id: ws_detail
    bnd_type: Grid
    table: code_detail
    trigger: {trigger_yn: false, open_trigger: "ws_master"}
    crud: [R, C, U, D]

# 레이아웃 개요
layout:
  type: SplitContainer
  orientation: vertical
  split_size: 250
  children:
    - {role: master, component: EPIC_HANDSONTABLE}
    - {role: detail, component: EPIC_HANDSONTABLE}
```

---

## 유형 자동 판단 로직

```
입력 키워드/구조 분석:

1. 테이블 1개 + 검색 없음 → A (SimpleGrid)
2. 테이블 1개 + 검색 있음 → E (SearchGrid)
3. 테이블 1개(Grid) + 상세(Field) → B (Field+Grid Chain)
4. 테이블 2개 + 마스터-디테일 → C (MasterDetail)
5. 테이블 3개+ + 탭 → D (TabbedMasterDetail)
6. UI 없음, 데이터만 → F (API 전용)

ROCA 통계 참조:
  - D-SearchGrid (60.9%) + E-SearchMasterDetail (15.2%) = 76.1% → 검색이 있는 폼이 압도적 다수
  - A-FieldOnly (10.2%) + C-MasterDetail (9.2%) = 19.4%
  - B-SimpleGrid (4.4%) = 검색 없는 단순 그리드는 소수
```

---

## 후속 연결

스펙 완성 후:
- **"설계 시작"** / **"frameweb-canvas"** -> frameweb-canvas 스킬로 전달 (바인딩은 frameweb-binding)
- frameweb-canvas는 이 스펙의 Q1~Q8 답변을 받아 Step 1(분석)을 건너뛰고 바로 Step 2(설계)로 진입
## 도구 권한

이 단계에서 호출 가능한 forge.tool ID 명단 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)

## 산출물 형식

다음 단계가 입력으로 받는 모양 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)
