---
name: migration-step03-frmwrk
description: FRAME9 마이그레이션 마법사 Step 3 — EMAX SCF150(폼-워크셋 연결)을 FRMWRK(워크셋 정의)+placeholder WRKSQL로 변환한다. 워크셋 조회 순서(open_sq)·트리거 관계 생성 단계. FRMWRK 마이그레이션, 워크셋 생성, open_trigger, open_sq 계산, GridSet/FieldSet 매핑, Step 3 관련 작업이나 frmwrk_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 3 — FRMWRK (워크셋 정의 생성)

EMAX SCF150(use_yn=1) → **FRMWRK + placeholder WRKSQL**.
폼의 워크셋(WorkSet) 정의와 조회 순서·트리거 관계를 만드는 단계다.

파이프라인 위치: [migration-step02-frmcmp](../migration-step02-frmcmp/SKILL.md) 다음, [migration-step04-wrksql](../migration-step04-wrksql/SKILL.md) 이전.
전체 순서는 [migration-frame9](../migration-frame9/SKILL.md) 참조.

## 실행 방법

- UI: Migration 화면 → "Step 3: FRMWRK" 탭
- API: `POST /api/migration9/migrate/{frm_id}/frmwrk?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/frmwrk-list?bis_id={bis_id}`
- 마스터-디테일 관계 탐지: `GET /api/migration9/frmwrk/master-detail/{frm_cd}`

## 선행 조건

1. **BISFRM에 폼 존재(Step 2 완료)** — 없으면 "Run FRMCMP migration first" 에러
2. EMAX SCF150 접속 필수 — 로드 실패 시 접속정보 포함 RuntimeError로 하드 실패(다른 스텝의 warning 처리와 다름)

## 핵심 동작

- SCF150의 `wset_cd` → `wrk_id`(대문자), `wset_ty` → `wrk_type` 매핑(Grid→GridSet, FreeForm→FieldSet, SQL→DataSet 등)
- `link_grid` → `open_trigger` 변환: link_grid 있으면 참조 wrk_id, 없는 root GridSet은 'OPEN', 나머지 root는 None
- open_trigger 관계로 `open_sq` 재귀 계산(부모 우선·이름순)
- `trigger_yn = (open_sq==1 and open_trigger 없음)`, `bound_fcmp_ids`에 grid_nm을 jsonb 저장
- 워크셋마다 `SELECT 1 AS placeholder` WRKSQL 1건 생성(CRUDM은 wrk_type 기반) — Step 4가 실쿼리로 대체

## 재실행 시 동작

`delete_existing=True`: FRMWRK 삭제 후 재생성. insert 자체도 `ON CONFLICT (bis_id, frm_id, wrk_id) DO UPDATE`(upsert)라 중복 안전.
placeholder WRKSQL은 `DO NOTHING` — Step 4를 이미 돌렸다면 실쿼리를 덮지 않는다.

## 함정

- SCF150에 데이터가 없으면 **에러가 아니라 빈 결과 메시지** — 폼에 워크셋이 원래 없는 건지, wset 코드 불일치인지 소스에서 확인할 것
- MIGCMP 기반 fallback 경로(`_migrate_form_from_migcmp` 등)는 `migrate_form`에서 호출되지 않는 잔재 코드 — 수정 대상으로 오인하지 말 것
- MSSQL은 소스만. SQL 방언 변환 없음

## 코드 참조

- `backend/migration/frame9/services/frmwrk_migrator.py` — `migrate_form`(:224), `_calculate_open_sq`(:146), `_load_scf150_data`(:51)
