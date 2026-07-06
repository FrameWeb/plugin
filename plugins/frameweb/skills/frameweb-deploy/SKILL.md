---
name: frameweb-deploy
description: >
  폼 라이프사이클 5단계: 배포. build_form_export() → FRMMST에 JSONB 스냅샷 저장.
  폼 디자이너의 "배포" 버튼과 동일한 동작을 CLI에서 수행.
  Trigger: 폼 배포, deploy, FRMMST, 스냅샷, frameweb-deploy, 버전 배포,
  배포해줘, 폼 배포해줘.
user-invocable: true
---

# Form Deploy — FRMMST 배포 (JSONB 스냅샷)

frameweb-bindchain으로 동작이 검증된 폼을 FRMMST에 배포한다.
배포 = build_form_export()로 23개 테이블을 수집 → form_data JSONB로 저장.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

## 절차


## 라이프사이클 위치

```
frameweb-require → frameweb-canvas → frameweb-bindchain → frameweb-binding → frameweb-load → **frameweb-deploy** → frameweb-verify
```

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| Backend | localhost:8181 |
| 배포 API | POST /api/frameweb-deploy/deploy |

---

## 호출

```
/frameweb-deploy {frm_id} {version} {bis_id}
```

## 실행 절차

### Step 1: 사전 확인

```bash
# BISFRM 존재 확인
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT frm_id, frm_nm, version, status FROM bisfrm 
WHERE frm_id='{frm_id}' AND bis_id='{bis_id}';"
```

### Step 2: 배포 API 호출

```bash
curl -X POST localhost:8181/api/frameweb-deploy/deploy \
  -H "Content-Type: application/json" \
  -H "X-Bis-Id: {bis_id}" \
  -d '{"frm_id": "{frm_id}", "version": "{version}"}'
```

응답 확인:
- 200: 배포 성공
- 404: BISFRM 없음
- 500: build_form_export() 에러 → 서버 로그 확인

### Step 3: 배포 결과 검증

```bash
# 비즈니스 DB에서 FRMMST 확인
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {비즈니스DB} -c "
SELECT frm_id, version, 
       length(form_data::text) as json_size, 
       created_at
FROM frmmst 
WHERE bis_id='{bis_id}' AND frm_id='{frm_id}' AND version='{version}';"
```

### Step 4: BISFRM 버전 확인

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
SELECT frm_id, version FROM bisfrm 
WHERE frm_id='{frm_id}' AND bis_id='{bis_id}';"
```

### Step 5: 리포트

```
배포 결과:
  폼: {frm_id} v{version}
  bis_id: {bis_id}
  FRMMST form_data: {json_size} bytes
  배포 시각: {created_at}
  
  상태: ✅ 배포 완료
```

## 후속

- 배포 성공 → `/frameweb-verify {frm_id} {version} {bis_id}` (런타임 검증)
- 배포 실패 → 서버 로그 확인 → frameweb-binding으로 메타 수정
## 도구 권한

이 단계에서 호출 가능한 forge.tool ID 명단 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)

## 산출물 형식

다음 단계가 입력으로 받는 모양 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)
