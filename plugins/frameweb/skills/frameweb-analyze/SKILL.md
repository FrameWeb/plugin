---
name: frameweb-analyze
description: >
  폼 분석. 기존 폼의 메타데이터를 DB에서 읽고 구조를 설명한다. 읽기 전용.
  bnd-analyze 계승. 바인딩 체인, 컬럼 매핑, SQL 구조, 폼 유형 시각적 정리.
  Trigger: 폼 분석, 폼 구조, frameweb-analyze, 이 폼 뭐야, 이 폼 어떻게 되어있어,
  폼 파악, 폼 리버스, 구조 분석, 체인 분석, 매핑 분석.
user-invocable: true
---

# Form Analyze -- 폼 분석 (읽기 전용)

기존 폼의 메타데이터를 DB에서 읽고 **"이 폼이 뭘 하는 건지"** 한눈에 파악하는 도메인 지식형 스킬.
**라이프사이클 독립 -- 어디서든 진입 가능.**

## 포지셔닝

| 스킬 | 시작점 | 목적 | 행동 |
|------|--------|------|------|
| **frameweb-analyze** | **frm_id + bis_id** | **기존 폼 이해** | **읽기 전용** |
| frameweb-canvas | 요구사항 (백지) | 신규 폼 생성 (레이아웃·컴포넌트) | forms.sql 생성 |
| frameweb-binding | bisfrm+frmcmp 존재 | 바인딩 작동·디버깅 | DB 수정 |

**frameweb-binding Task 1과의 차이:**
- frameweb-binding Task 1: "셋업하려면 뭐가 빠졌나" (다음 Task로 이어짐)
- frameweb-analyze: "이 폼이 뭘 하는 건지" (독립 산출물)

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 비즈니스 DB | bisdb 테이블에서 조회 |

## 참조 문서

- [references/form_patterns.md](references/form_patterns.md) -- ROCA 유형별 통계 (비교용)
- [references/epic_form_patterns.md](references/epic_form_patterns.md) -- Epic 유형별 정답 패턴

---

## 호출

```
/frameweb-analyze {폼ID} {bis_id}
```

---

## 분석 절차 (3단계)

### Step 1: 메타데이터 수집

8개 테이블을 순서대로 조회한다.

```sql
-- 1. bisfrm (폼 마스터)
SELECT frm_id, frm_nm, frm_desc, status, canvas_width, canvas_height
FROM bisfrm WHERE frm_id='{폼ID}' AND bis_id='{bis_id}';

-- 2. frmcmp (캔버스 컴포넌트)
SELECT fcmp_id, cmp_id, label, parent_id, layout_mode, x, y, width, height, display_order
FROM frmcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY COALESCE(parent_id,''), display_order;

-- 3. frmbnd (바인딩 마스터)
SELECT bnd_id, bnd_type, bnd_nm, bound_fcmp_ids,
       open_sq, trigger_yn, open_trigger, new_yn, save_sq
FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY open_sq;

-- 4. bndquery (SQL 쿼리)
SELECT bnd_id, crudm, source_type, LEFT(query, 120) AS query_preview,
       execution_order, description
FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, crudm;

-- 5. bndpull (GET 파라미터)
SELECT bnd_id, crudm, param_nm, source_type, source_ref, source_column, default_val
FROM bndpull WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;

-- 6. bndsave (SET 파라미터)
SELECT bnd_id, param_nm, source_type, source_ref, source_column, default_val
FROM bndsave WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, param_nm;

-- 7. bndcmp (컬럼 매핑)
SELECT bnd_id, fcmp_id, data_field, data_type, editor_type,
       col_width, visible, is_required, col_seq
FROM bndcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, col_seq;

-- 8. bndpush (체인 참조)
SELECT bnd_id, ref_bnd_id, push_type
FROM bndpush WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id;
```

### Step 2: 구조 해석

수집한 메타데이터를 아래 5가지 관점으로 정리한다.

#### 2-1. 폼 개요

```
폼 ID:    {frm_id}
폼 이름:  {frm_nm}
bis_id:   {bis_id}
유형:     {A~F 판단}
바인딩 수: {frmbnd 건수}
컴포넌트 수: {frmcmp 건수}
```

#### 2-2. 레이아웃 트리

frmcmp의 parent_id 관계를 트리로 그린다.

```
ROOT
+-- [split_main] SplitContainer (vertical, 250px)
    +-- [panel_top] SplitPanel "코드분류"
    |   +-- [grid_master] Grid (fill)
    +-- [panel_bottom] SplitPanel "기초코드"
        +-- [grid_detail] Grid (fill)
```

#### 2-3. 바인딩 체인 다이어그램

frmbnd의 open_trigger + bndpush 관계를 시각화한다.

```
[Open 클릭]
  -> ws_master (Grid, open_sq=1, trigger=OPEN)
    -> ws_detail (Grid, open_sq=2, trigger=ws_master)  <- 마스터 행 선택 시

[Save 클릭]
  -> ws_master (save_sq=1) -> C/U/D SQL
  -> ws_detail (save_sq=2) -> C/U/D SQL
```

#### 2-4. SQL 요약

각 바인딩별 R/C/U/D SQL의 핵심을 요약한다.

```
| bnd_id    | crudm | 대상 테이블 | 조건 | 비고 |
|-----------|-------|-----------|------|------|
| ws_master | R     | code_group | co_cd | |
| ws_master | C     | code_group | -    | INSERT |
| ws_master | U     | code_group | grp_cd_old | UPDATE |
| ws_master | D     | code_detail+code_group | grp_cd | 연쇄 삭제 |
| ws_detail | R     | code_detail | co_cd, grp_cd | 마스터 연계 |
```

#### 2-5. 컬럼 매핑 요약

bndcmp에서 주요 컬럼을 정리한다.

```
| bnd_id    | data_field | editor_type | visible | 비고 |
|-----------|-----------|-------------|---------|------|
| ws_master | grp_cd    | TEXT        | true    | PK |
| ws_master | grp_nm    | TEXT        | true    | |
| ws_detail | grp_cd    | TEXT        | false   | FK (숨김) |
| ws_detail | sub_cd    | TEXT        | true    | PK |
```

### Step 3: 산출물 출력 + 후속 연결

위 5가지를 하나의 분석 리포트로 출력한다.
추가 액션이 필요하면 다음 스킬로 연결:
- 재설계 -> frameweb-canvas (레이아웃) / frameweb-binding (바인딩)
- 디버깅 -> frameweb-binding
- 스크립트 추가 -> frameweb-script

---

## 폼 유형 판단 기준 (A~F)

frmbnd 구조를 보고 유형을 판단한다.

| 유형 | 바인딩 수 | trigger 패턴 | 대표 예시 |
|------|----------|-------------|----------|
| A. SimpleGrid | 1 | trigger=true, open_trigger=OPEN | 단순 목록 |
| B. Field+Grid Chain | 2+ | Grid + Field 체인 | 목록+상세 |
| C. MasterDetail | 2+ | 마스터 trigger + 디테일 체인 | 마스터-디테일 |
| D. TabbedMasterDetail | 2+ | 마스터 + 탭 디테일들 | 탭 멀티 디테일 |
| E. SearchGrid | 1+ | trigger=false, open_trigger=OPEN | 검색 조건+결과 |
| F. API 전용 | 1+ | Data bnd_type, UI 없음 | 백엔드 전용 |

---

## ROCA 비교 인사이트

form_patterns.md의 유형별 평균과 현재 폼을 비교하여 복잡도를 평가한다.

```
예시:
  "이 폼은 C유형(MasterDetail) 평균 대비:
   - 바인딩 2개 (평균 2.5개) -> 표준
   - 컬럼 12개 (평균 45개) -> 단순
   - CRUD 8개 (평균 4.1개) -> 표준"
```

ROCA 유형별 평균 (form_patterns.md 기준):
| ROCA 유형 | 폼 수 | 필드 평균 | 그리드 평균 | WorkSet 평균 |
|----------|------|----------|-----------|-------------|
| D-SearchGrid | 1233 | 10.0 | 1.8 | 2.7 |
| E-SearchMasterDetail | 307 | 16.7 | 3.8 | 5.3 |
| A-FieldOnly | 207 | 15.2 | 0 | 1.5 |
| C-MasterDetail | 187 | 7.0 | 2.4 | 2.8 |
| B-SimpleGrid | 89 | 0.5 | 1.2 | 1.6 |

---

## bnd* 테이블 관계도

```
bisfrm (폼 마스터)
  +-- frmcmp (캔버스 컴포넌트)
  +-- frmbnd (바인딩 마스터)
        +-- bndquery (SQL: R/C/U/D)
        +-- bndpull (GET 파라미터: R SQL의 :param 소스)
        +-- bndsave (SET 파라미터: CUD SQL의 :param 소스)
        +-- bndcmp (컬럼-컴포넌트 매핑)
        +-- bndpush (체인 참조: 다른 바인딩셋 트리거)
```

### PK 구조

| 테이블 | PK |
|--------|-----|
| bisfrm | bis_id + frm_id |
| frmcmp | bis_id + frm_id + fcmp_id |
| frmbnd | bis_id + frm_id + bnd_id |
| bndquery | bis_id + frm_id + bnd_id + crudm + execution_order |
| bndpull | bis_id + frm_id + bnd_id + crudm + param_nm |
| bndsave | bis_id + frm_id + bnd_id + param_nm |
| bndcmp | bis_id + frm_id + bnd_id + fcmp_id |
| bndpush | bis_id + frm_id + bnd_id + ref_bnd_id |

---

## 비즈니스 테이블 확인

R SQL에서 참조하는 비즈니스 테이블의 존재/데이터를 확인한다.

```sql
-- 비즈니스 DB에서 테이블 존재 확인
SELECT table_name FROM information_schema.tables
WHERE table_schema='public' AND table_catalog='{bis_id}';

-- 데이터 건수 확인
SELECT count(*) FROM {테이블명};
```

---

## 활용 시나리오

### 1. "이 폼 어떻게 되어있어?"
-> `/frameweb-analyze {폼ID} {bis_id}` -> 분석 리포트 출력

### 2. "이 폼을 참고해서 새 폼 만들고 싶어"
-> frameweb-analyze로 현재 구조 파악 -> frameweb-require/frameweb-canvas에 분석 결과를 입력으로 전달

### 3. "폼이 안 되는데 원인을 모르겠어"
-> frameweb-analyze로 구조 파악 -> 문제 위치 특정 -> frameweb-binding으로 수정

### 4. "가이드 폼을 만들려면 뭘 참고해야 해?"
-> frameweb-analyze로 샘플 폼 분석 -> 가이드 문서 작성 기초
