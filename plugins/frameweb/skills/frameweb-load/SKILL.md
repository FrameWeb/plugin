---
name: frameweb-load
description: >
  폼 라이프사이클 3단계: DB 적재. forms.sql을 시스템 DB에 실행하고 적재 결과를 검증한다.
  menu.sql이 있으면 비즈니스 DB에도 실행. FK/PK 충돌 체크 + 적재 건수 리포트.
  Trigger: 폼 적재, SQL 실행, forms.sql 실행, frameweb-load, 패키지 설치, DB 적재,
  forms.sql 돌려, 적재해줘.
user-invocable: true
---

# Form Load — forms.sql DB 적재 + 검증

frameweb-canvas/frameweb-binding이 생성한 forms.sql을 시스템 DB에 실행하고, 결과를 검증한다.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

## 절차


## 레퍼런스
- [references/epic_form_patterns.md](references/epic_form_patterns.md) — INSERT 순서/패턴 확인

## 라이프사이클 위치

```
frameweb-require → frameweb-canvas → frameweb-bindchain → frameweb-binding → **frameweb-load** → frameweb-deploy → frameweb-verify
```

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 비즈니스 DB | bisdb 테이블에서 host/port/database/username 조회 |

---

## 호출

```
/frameweb-load {forms.sql 경로} {bis_id}
```

## 실행 절차

### Step 1: 사전 확인

```bash
# forms.sql 파일 존재 확인
ls -la {forms.sql 경로}

# {bis_id} 치환이 필요한지 확인
grep '{bis_id}' {forms.sql 경로}
```

### Step 2: forms.sql 실행 (시스템 DB)

```bash
# bis_id 치환 후 실행
sed "s/{bis_id}/{실제_bis_id}/g" {forms.sql} | \
  PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME}
```

에러 발생 시:
- FK 위반 → INSERT 순서 확인 (BISFRM→FRMCMP→FRMBND→BNDCMP→BNDQUERY→BNDPULL→BNDSAVE)
- PK 중복 → ON CONFLICT DO NOTHING이면 무시됨, 아니면 기존 데이터 확인

### Step 3: menu.sql 실행 (비즈니스 DB, 있을 때만)

```bash
# 비즈니스 DB 접속 정보 조회
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT database FROM bisdb WHERE bis_id='{bis_id}' AND is_active='Y';"

# menu.sql 실행
sed "s/{bis_id}/{실제_bis_id}/g" menu.sql | \
  PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {비즈니스DB}
```

### Step 4: 적재 결과 검증

```sql
-- 시스템 DB에서 확인
SELECT 'bisfrm' AS tbl, count(*) FROM bisfrm WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmcmp', count(*) FROM frmcmp WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'frmbnd', count(*) FROM frmbnd WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndcmp', count(*) FROM bndcmp WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndquery', count(*) FROM bndquery WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndpull', count(*) FROM bndpull WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
UNION ALL SELECT 'bndsave', count(*) FROM bndsave WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
ORDER BY tbl;
```

### Step 5: 리포트 출력

```
적재 결과:
  bisfrm:   1건
  frmcmp:   12건
  frmbnd:   3건
  bndcmp:   25건
  bndquery: 8건
  bndpull:  4건
  bndsave:  12건
  
  menu.sql: mnumst 2건, mnupms 6건 (비즈니스 DB)
  
  상태: ✅ 정상 적재
```

## 후속

- 적재 성공 → `/frameweb-bindchain {frm_id} {bis_id}` (동작 검증)
- 적재 실패 → 에러 메시지 분석 → forms.sql 수정 → 재실행
## 도구 권한

이 단계에서 호출 가능한 forge.tool ID 명단 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)

## 산출물 형식

다음 단계가 입력으로 받는 모양 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)
