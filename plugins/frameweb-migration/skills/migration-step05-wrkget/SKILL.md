---
name: migration-step05-wrkget
description: FRAME9 마이그레이션 마법사 Step 5 — EMAX SCF160(GET 파라미터)을 WRKGET(워크셋 입력 파라미터)으로 변환하고 SCC100 기본값을 FRMCMP에 반영한다. WRKGET 마이그레이션, GET 파라미터, source_type 판정(GRIDSET/FIELDSET/COMPONENT), sql_id 매칭 실패, Step 5 관련 작업이나 wrkget_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 5 — WRKGET (조회 파라미터 바인딩)

EMAX SCF160(GET 파라미터) → **WRKGET**. 부수 효과로 SCC100 def_text를 FRMCMP.defaultValue에 반영한다.
워크셋 조회(R) 쿼리의 입력 파라미터가 어느 컴포넌트/그리드에서 값을 얻는지 정의하는 단계다.

파이프라인 위치: [migration-step04-wrksql](../migration-step04-wrksql/SKILL.md) 다음, [migration-step06-wrkcmp](../migration-step06-wrkcmp/SKILL.md) 이전.

## 실행 방법

- UI: Migration 화면 → "Step 5: WRKGET" 탭
- API: `POST /api/migration9/migrate/{frm_id}/wrkget?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/wrkget-list?bis_id={bis_id}` (source_type 분포 포함)
- 사전 미리보기: `GET /api/migration9/wrkget/preview/{frm_cd}`

## 선행 조건 — 순서 의존이 가장 강한 단계

1. **FRMWRK 존재(Step 3)** — 없으면 실패
2. **WRKSQL의 SELECT(crudm='R') 존재(Step 4)** — 각 파라미터는 해당 워크셋 R쿼리의 sql_id에 매달린다. **sql_id를 못 찾으면 그 파라미터는 통째로 skip**되므로 Step 4 이전에 돌리면 WRKGET이 빈다(가장 흔한 함정, `skipped_reasons`에 기록됨)
3. FRMCMP(Step 2) — defaultValue 갱신이 반영될 대상

## 핵심 동작

- SCF160 로드 → `wset_cd→wrk_id`, `in_param`의 `@` 제거해 `param_nm`
- `ctr_nm`에 점이 있으면(`g10.dept_cd`) 참조 워크셋 wrk_type으로 GRIDSET/FIELDSET 판정, `source_ref`=그리드, `source_column`=컬럼. 점이 없으면 COMPONENT
- master grid(open_sq=1, GridSet) 탐색으로 Observer 패턴 근거 확보
- WRKGET insert(param_type 'string' 고정, crudm 'R')
- 이후 SCC100 def_text(점 없는 컨트롤)로 FRMCMP defaultValue/defaultValues UPDATE(덮어쓰기)

## 재실행 시 동작

`delete_existing=True`: WRKGET 삭제 후 재생성. insert는 `ON CONFLICT (bis_id,frm_id,wrk_id,crudm,param_nm) DO NOTHING`(중복 무시).
FRMCMP defaultValue는 매번 덮어쓴다 — Step 2 이후 수동으로 고친 기본값이 있다면 되돌아간다.

## 함정

- 결과가 비면 먼저 `skipped_reasons`를 본다 — 대부분 "sql_id 미발견"(Step 4 미실행/실패)
- SCF160/SCC100 로드 실패는 warning 후 빈 결과 — 에러 없이 0건이면 소스 접속부터 의심
- MSSQL은 소스만. SQL 방언 분기 없음(타깃은 시스템 DB PG)

## 코드 참조

- `backend/migration/frame9/services/wrkget_migrator.py` — `migrate_form`(:278), `_infer_source_type`(:210), `_update_frmcmp_defaults`(:174), `_find_sql_id_for_workset`(:121)
