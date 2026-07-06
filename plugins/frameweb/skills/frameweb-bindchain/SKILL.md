---
name: frameweb-bindchain
description: >
  폼 디자이너 BindChain 탭: 컴포넌트와 바인딩을 연결하고 체인(Observer Chain) 순서를 구성한다.
  frmbnd(바인딩 체인)·bound_fcmp_ids를 손댄다. 생성(FRMBND 정의 작성)과 디버깅(빠진 연결/체인 순서 보정)을 모두 담당한다.
  Trigger: BindChain, 바인딩 연결, frmbnd, bound_fcmp_ids, Observer Chain, open_sq, open_trigger,
  trigger_yn, 체인 순서, 마스터-디테일 연결, 그리드 클릭해도 디테일 안 떠, frameweb-bindchain.
user-invocable: true
---

# Form BindChain — 컴포넌트와 바인딩 연결, 체인 순서 구성

폼 디자이너의 BindChain 탭이 다루는 대상을 담당한다. 컴포넌트(frmcmp)와 바인딩(frmbnd)을 연결하고, 마스터→디테일 같은 연쇄 조회의 순서를 Observer Chain으로 구성한다. FRMBND 정의를 새로 작성하는 생성과, 진입점에 빠진 연결·잘못된 체인 순서를 찾아 고치는 디버깅을 함께 처리한다.

## 폼 디자이너 탭 스킬 묶음

이 스킬은 폼 디자이너 탭 스킬 6종(frameweb-canvas / frameweb-bindchain / frameweb-binding / frameweb-script / frameweb-sandbox / frameweb-state) 중 하나다. 각 탭과 1:1로 맞고 `form-` prefix를 통일해, 나중에 포지(Forge, Cloud Dev AI 영역)로 폴더 단위 이식이 가능하도록 설계됐다. BindChain 탭은 frmbnd(바인딩 체인)과 bound_fcmp_ids를 손댄다. 전체 구조는 `docs/specs/form-skill-tab-restructure.md` 참조.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

BindChain 탭이 손대는 frmbnd는 어떤 컴포넌트(bound_fcmp_ids)에 어떤 바인딩을 묶을지, 그 바인딩이 언제 실행될지(trigger_yn, open_trigger), 어떤 순서로 연쇄될지(open_sq)를 정의한다. SQL 자체(bndquery)와 컬럼/파라미터 매핑(bndpull/bndsave/bndcmp)은 frameweb-binding 탭이, 컴포넌트 배치는 frameweb-canvas 탭이 담당한다.

## 환경

| 항목 | 값 |
|------|-----|
| Frontend | ${FRAMEWEB_APP_URL} |
| Backend | localhost:8181 |
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 비즈니스 DB | bisdb 테이블에서 조회 |
| ERP Preview | ${FRAMEWEB_APP_URL}/erp/form/{폼ID}?bis_id={bis_id} |
| 모리 공유 폴더 | /mnt/z/Obsidian/browser-test/request/ |
| 접속 정보 단일 소스 | ~/.claude/memory/db-connections.md |

## 참조 문서

- [references/design-patterns.md](references/design-patterns.md) — 폼 유형 A~E, trigger 설계 판단, Observer Chain
- [references/db-queries.md](references/db-queries.md) — bnd* DB 쿼리 카탈로그
- [references/epic_form_patterns.md](references/epic_form_patterns.md) — 유형별 정답 패턴

## 절대규칙

> 1. **bound_fcmp_ids 추측 금지** — DB 조회로 실제 컴포넌트 ID를 확인한다. 잘못된 ID는 조회에 성공해도 그리드가 빈 채로 남는다.
> 2. **open_sq > 0 필수** — 0이면 Observer Chain이 동작하지 않는다.

---

## 호출

```
/frameweb-bindchain {폼ID} {bis_id}
```

생성(새 바인딩 연결 작성)과 디버깅(빠진 연결/잘못된 체인 순서 보정)을 모두 이 스킬에서 진행한다.

---

## 작업 자동 생성 (TaskCreate)

호출 시 자기 영역 작업 2개를 TaskCreate로 먼저 생성한다. 이 스킬은 옛 form-setup의 7개 작업 중 Task 1(현황 파악)·Task 2(frmbnd 보정)를 담당한다. SQL(frameweb-binding)·컴포넌트(frameweb-canvas)·런타임 테스트(frameweb-verify)는 각 탭 스킬이 자기 작업을 만든다.

### 생성하는 작업

```
TaskCreate:
  subject: "[{폼ID}] 현황 파악"
  description: |
    아래 SKILL의 "Step 1: 현황 파악" 절차를 따른다.
    전체 스냅샷 쿼리 → 상세 조회(frmcmp/frmbnd/bndquery/bndpull/bndsave) → 현황표 작성.
    폼 유형(A~E) 판단 + 비즈니스 테이블 테스트 데이터 확인.
    완료 조건: 현황표 작성 + 작업 범위 결정 + 테스트 데이터 존재 확인.

TaskCreate:
  subject: "[{폼ID}] frmbnd 보정"
  description: |
    아래 SKILL의 "Step 2: FRMBND 정의 작성 / 설정 보정" 절차를 따른다.
    bnd_type 결정 → trigger_yn × open_trigger 조합 판단 → bound_fcmp_ids DB 확인 →
    open_sq/new_yn/save_sq 설정 → UPDATE 실행 → 확인 쿼리.
    완료 조건: bound_fcmp_ids 실 ID 일치 + open_sq 전부 > 0 +
    trigger 조합이 폼 유형에 맞음 + CUD 대상 new_yn='Y'·save_sq > 0.
```

### 다오 직접 vs 모리 위임 — 이 스킬 해당분

| 작업 | 실행자 | 이유 |
|------|--------|------|
| 현황 파악 (DB 조회) | **다오** | psql 직접 접근 |
| frmbnd 보정 (DB UPDATE) | **다오** | psql 직접 접근 |

두 작업 모두 DB 작업이라 다오가 직접 실행한다. 모리 위임은 이 스킬에 없다. 런타임 화면 검증(Open/CRUD 테스트)이 필요하면 frameweb-verify로 넘기고, 그 스킬이 모리 위임 작업을 만든다.

### 작업 흐름과 라우팅

```
[현황 파악] → [frmbnd 보정]
                   |
                   +- SQL 없음/오류         → frameweb-binding (자기 작업 생성)
                   +- 컴포넌트 부재/오류     → frameweb-canvas (자기 작업 생성)
                   +- 연결·체인 정상, 런타임 검증 → frameweb-verify (모리 위임 작업 생성)
```

현황 파악에서 bndquery·bndpull·bndsave가 비어 있으면 보정 전에 frameweb-binding으로, 가리킬 컴포넌트 자체가 없으면 frameweb-canvas로 넘긴다. 두 작업이 끝나고 연결·체인이 올바르면 frameweb-verify로 런타임 검증을 넘긴다.

### 실행 예시

사용자가 `/frameweb-bindchain COD100 PACKAGE` 입력 시:
1. 작업 2개 TaskCreate 실행 (subject에 `[COD100]` 포함).
2. "현황 파악" 진행 — DB 조회로 현황표 작성, 폼 유형 판단 (다오 직접).
3. "frmbnd 보정" 진행 — bound_fcmp_ids·open_sq·trigger 조합 UPDATE (다오 직접).
4. SQL 누락이면 frameweb-binding, 컴포넌트 부재면 frameweb-canvas로 라우팅.
5. 연결·체인 정상이면 frameweb-verify로 런타임 검증 넘김 (모리 위임).

---

## 실행 절차

### Step 1: 현황 파악

폼의 현재 frmbnd 상태와 연결 구조를 조회해 작업 범위를 결정한다.

**전체 스냅샷 쿼리** — 어느 테이블이 채워졌는지 한눈에 본다.

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT 'bisfrm' AS tbl, count(*) FROM bisfrm WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmcmp', count(*) FROM frmcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmbnd', count(*) FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndquery', count(*) FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndpull', count(*) FROM bndpull WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndsave', count(*) FROM bndsave WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndcmp', count(*) FROM bndcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndpush', count(*) FROM bndpush WHERE frm_id='{폼ID}' AND bis_id='{bis_id}';"
```

**상세 조회** — 컴포넌트 목록과 바인딩 설정을 본다.

```bash
# 1. frmcmp: fcmp_id, cmp_id, label 목록 (bound_fcmp_ids 확인의 근거)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT fcmp_id, cmp_id, label, parent_id, layout_mode
FROM frmcmp WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY COALESCE(parent_id,''), fcmp_id;"

# 2. frmbnd: bnd_id, bnd_type, 핵심 설정 (체인 구조의 핵심)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, bnd_type, bound_fcmp_ids, open_sq, trigger_yn, open_trigger, new_yn, save_sq
FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY open_sq;"

# 3. bndquery: bnd_id, crudm별 존재 여부 (연결 대상 SQL 유무 — 상세 작성은 frameweb-binding)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, crudm, LEFT(query, 80) AS query_preview
FROM bndquery WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
ORDER BY bnd_id, crudm;"

# 4. bndpull 건수
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, count(*) FROM bndpull
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' GROUP BY bnd_id;"

# 5. bndsave 건수
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, count(*) FROM bndsave
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' GROUP BY bnd_id;"
```

**산출물: 현황표**

| 항목 | 상태 | 필요 작업 |
|------|------|----------|
| bisfrm | O/X | |
| frmcmp | N개 | |
| frmbnd | N개, 설정값 | 보정 필요 여부 |
| bndquery R | O/X | (없으면 frameweb-binding) |
| bndquery CUD | O/X | (없으면 frameweb-binding) |
| bndpull | N건 | (frameweb-binding) |
| bndsave | N건 | (frameweb-binding) |
| bndcmp | N건 | (frameweb-binding) |

**폼 유형 판단** — design-patterns.md 참조하여 A~E 유형을 결정한다. 워크셋 수, 그리드 수, 마스터-디테일 관계를 파악한다.

**테스트 데이터 확인** — R SQL이 조회하는 비즈니스 테이블에 데이터가 있는지 확인한다. bisdb에서 비즈 DB 이름 조회 후 접속한다.

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT database FROM bisdb WHERE bis_id='{bis_id}' AND is_active='Y';"
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {조회된_database} -c "SELECT count(*) FROM 테이블;"
# 0건이면 테스트 데이터 INSERT (최소 2~3건)
```

**완료 조건**

- 현황표 작성 완료
- 작업 범위 결정 (어떤 연결을 새로 작성하고, 어떤 설정을 보정할지)
- 비즈니스 테이블에 테스트 데이터 존재 확인 (없으면 투입)

---

### Step 2: FRMBND 정의 작성 / 설정 보정

새 바인딩을 정의하거나, 기존 frmbnd의 핵심 컬럼을 올바르게 보정한다. 생성과 디버깅이 같은 컬럼을 다룬다.

#### bnd_type 결정

| 조건 | bnd_type | open_trigger |
|------|----------|-------------|
| 그리드 바인딩 (메인) | Grid | 'OPEN' |
| 그리드 바인딩 (체인 자식) | Grid | '{부모_bnd_id}' |
| 입력 필드 바인딩 | Field | NULL 또는 '{부모_bnd_id}' |
| 참조 쿼리 (화면 없음) | Data | NULL |

> **bnd_type은 반드시 'Grid'/'Field'/'Data' 사용.** 'GridSet'/'FieldSet'/'DataSet'은 wrk* 레거시 체계다.

#### trigger 설계 판단 플로우

**trigger_yn × open_trigger 조합**

| 조합 | trigger_yn | open_trigger | 용도 |
|------|-----------|-------------|------|
| **A** | true | 'OPEN' | 폼 열면 바로 데이터 표시 |
| **B** | true | NULL | 조건 없이 전체 즉시 표시 |
| **D** | false | 'OPEN' | 조건 입력 후 Open 버튼으로 조회 |
| **E** | false | '{bnd_id}' | 마스터 행 클릭 시 연쇄 조회 |
| **F** | false | NULL | Script/버튼으로 직접 호출 |

**기본값: trigger_yn = false** (ido 명시 요청 없으면 false)

**판단 플로우**

```
1. 다른 바인딩에 종속?
   +- YES -> open_trigger = '{마스터_bnd_id}', trigger_yn = false (E)
   +- NO -> 독립 바인딩
       2. 폼 열자마자 자동 실행?
          +- YES, OPEN과 함께 -> trigger_yn=true, open_trigger='OPEN' (A)
          +- YES, 단독 -> trigger_yn=true, open_trigger=NULL (B)
          +- NO, Open 버튼 -> trigger_yn=false, open_trigger='OPEN' (D)
          +- NO, 수동 -> trigger_yn=false, open_trigger=NULL (F)
```

**실무 패턴**

| 패턴 | 마스터 | 디테일 |
|------|--------|--------|
| 마스터-디테일 | D (false/OPEN) | E (false/{마스터}) |
| 단일 Grid | B (true/NULL) | -- |
| 조건 검색 | D (false/OPEN) | E (false/{검색}) |

#### Observer Chain 필수 전제

그리드 컴포넌트의 CMPMST.chain_events가 없으면 체인이 동작하지 않는다.

```sql
-- EPIC_HANDSONTABLE에 이미 설정되어 있어야 함. 없으면:
UPDATE cmpmst SET chain_events = '[{"state":"selectedRow","fields":["*"]}]'
WHERE cmp_id = 'EPIC_HANDSONTABLE';
```

#### NEW/SAVE 규칙

| 항목 | 규칙 |
|------|------|
| new_yn='Y' | **FieldSet에만** 적용, **폼당 1개만** |
| new_yn='N' | GridSet은 항상 'N' (그리드 Add 버튼 사용) |
| save_sq | CUD 대상에 순서 (마스터=1, 디테일=2). 0이면 Save 스킵 |
| open_sq | **반드시 > 0** (0이면 Observer Chain 미동작) |
| bound_fcmp_ids | JSON 배열 `'["grid_xxx"]'`. GridSet만 설정 |

#### bound_fcmp_ids 확인 (★ 절대규칙)

추측하지 않고 DB에서 실제 컴포넌트 ID를 확인한다.

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT fcmp_id, cmp_id FROM frmcmp
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}'
AND cmp_id IN ('EPIC_HANDSONTABLE','EMAX_HANDSONTABLE');"
```

- Grid: `["grid_xxx"]` — 위 조회로 확인한 fcmp_id
- Field/Data: `[]`
- 잘못된 ID → 조회는 성공하지만 그리드가 빈 채로 남는다.

#### bnd_id 명명 규칙

```
ws_{역할}        예: ws_role, ws_member, ws_code, ws_detail
{frm_id}_{grid}  예: BCA101_G10 (마이그레이션 시)
```

#### UPDATE 실행 (기존 바인딩 보정)

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
UPDATE frmbnd SET
  bound_fcmp_ids = '값',
  open_sq = N,
  trigger_yn = true/false,
  open_trigger = '값',
  new_yn = 'Y/N',
  save_sq = N
WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' AND bnd_id='BND_ID';"
```

#### 확인

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT bnd_id, bnd_type, bound_fcmp_ids, open_sq, trigger_yn, open_trigger, new_yn, save_sq
FROM frmbnd WHERE frm_id='{폼ID}' AND bis_id='{bis_id}' ORDER BY open_sq;"
```

**완료 조건**

- bound_fcmp_ids가 실제 컴포넌트 ID와 일치
- open_sq 전부 > 0
- trigger_yn/open_trigger 조합이 폼 유형에 맞음
- CUD 대상의 new_yn='Y', save_sq > 0

---

## 디버깅 진입점

진입점에 빠진 연결 또는 잘못된 체인 순서가 증상의 원인일 때 이 스킬에서 처리한다.

| 증상 | 원인 후보 | 점검 |
|------|----------|------|
| 그리드 클릭해도 디테일 안 뜸 | open_sq=0 또는 디테일 open_trigger가 마스터 bnd_id와 불일치 | Step 2 open_sq, open_trigger 조합 |
| 조회는 되는데 그리드가 빈 채 | bound_fcmp_ids가 실제 fcmp_id와 불일치 | Step 2 bound_fcmp_ids 확인 |
| 체인 자체가 안 걸림 | cmpmst.chain_events 부재 | Step 2 Observer Chain 전제 |
| 폼 열어도 자동 조회 안 됨 / 원치 않게 자동 조회됨 | trigger_yn × open_trigger 조합이 폼 유형과 불일치 | Step 2 trigger 판단 플로우 |

## 라우팅 — 원인이 BindChain 밖일 때

연결과 체인 순서가 올바른데도 동작하지 않으면, 원인은 다른 탭에 있다. 해당 스킬로 보낸다.

| 보낼 곳 | 조건 |
|---------|------|
| **frameweb-binding** | SQL 자체 문제 — R/CUD 쿼리 없음·문법 오류, bndpull/bndsave 파라미터 매핑 누락, 500 에러가 SQL 실행 단계에서 발생 |
| **frameweb-canvas** | 컴포넌트 문제 — 그리드/필드 컴포넌트가 frmcmp에 없음, 컴포넌트 type·레이아웃·parent_id 오류, bound_fcmp_ids로 가리킬 대상 자체가 부재 |

## 후속

- 연결·체인 구성 완료 → 런타임 검증은 **frameweb-verify** (폼 열기·CRUD 테스트, 모리 위임)
- SQL/파라미터 매핑 작업 필요 → **frameweb-binding**
- 컴포넌트 배치/속성 작업 필요 → **frameweb-canvas**

<!-- writer-check: 위 원칙 전원 준수. -->
