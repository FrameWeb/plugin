---
name: migration-step07-bispop
description: FRAME9 마이그레이션 마법사 Step 7 — EMAX SCL100/BCA200/SCL150 등의 팝업·코드·룩업을 BISPOP+POPCOD/POPSQL/POPIO/POPCMP(입력보조)로 변환한다. BISPOP 마이그레이션, 입력보조, 팝업 코드, required codes 존재 확인, pop_type C/L/P, CMPIO, Step 7 관련 작업이나 bispop_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 7 — BISPOP (입력보조 마이그레이션)

EMAX SCL100/BCA200/SCL150/SCW100·110·120/SCC100 → **BISPOP + POPCOD/POPSQL/POPIO/POPCMP**.
폼이 참조하는 팝업·코드·룩업(입력보조)을 만드는 단계다.

파이프라인 위치: [migration-step06-wrkcmp](../migration-step06-wrkcmp/SKILL.md) 다음, [migration-step08-biztbl](../migration-step08-biztbl/SKILL.md) 이전.
대상 목록은 Step 6 탭의 required-codes(SCC100 popup_id 기반)가 준다.

## 실행 방법

- UI: Migration 화면 → "Step 7: BISPOP" 탭 — 탭 진입 시 required-codes 기준으로 존재 여부 자동 확인
- 존재 확인: `POST /api/migration9/bispop/check-exists` (code_ids 배열)
- 미리보기: `GET /api/migration9/bispop/preview/{pop_id}`
- 실행: `POST /api/migration9/bispop/migrate-multiple` — `{code_ids: [...], delete_existing: false}` (단건은 `POST /bispop/migrate/{pop_id}`)

## pop_type 분기 (SCL100.main_cd 기준)

| pop_type | 소스 | 산출 |
|----------|------|------|
| C (Code) | BCA200 코드 | POPCOD |
| L (Lookup/SQL) | SCL150 combo/dyna_sql | POPSQL + POPIO |
| P (Popup) | SCW100(src='Popup') SQL | POPSQL + POPIO + POPCMP(그리드 컬럼) |

SQL은 Step 4와 같은 변환기(`_convert_emax_sql_to_epic`)로 PG 변환 후 저장된다.

## 선행 조건 / 재실행

- POPCMP까지 채우려면 FRMCMP/WRKCMP(Step 2·6) 선행 — required-codes가 optionSource를 폼 컴포넌트에서 찾는다
- `delete_existing=False`(기본): 이미 BISPOP 존재하면 skip하고 성공 반환
- `delete_existing=True`: BISPOP 삭제 → ORM CASCADE로 하위(POPCOD/POPSQL/POPIO/POPCMP) 자동 삭제 후 재생성

## 함정

- **mig_except 가드** — `mig_except.pop`에 등록된 팝업은 마이그레이션하지 않고 skipped 처리된다. "왜 안 만들어지지?" 싶으면 제외 목록부터 확인
- SQL 변환 경고는 로그만 남기고 계속 진행 — POPSQL 저장 후 실제 실행 검증 필요
- POPCMP 조회는 `DECLARE @_frm_cd` T-SQL 배치라 MSSQL 소스 전용
- POPIO의 OUT 매핑은 reg_no=1→value, 2→label 규약
- 후속 보조 단계 **CMPIO**(`POST /api/migration9/migrate/{frm_id}/cmpio`, 화면 탭 없음): SCC120의 컴포넌트-팝업 IO 매핑을 CMPIO로 변환. BISPOP·POPIO·WRKCMP·FRMWRK 모두 선행 필요, 실행 시 frm_id 기준 전량 삭제 후 재생성

## 코드 참조

- `backend/migration/frame9/services/bispop_migrator.py` — `migrate_popup`(:593), `_migrate_popsql`(:382), `_migrate_popcmp`(:526)
- `backend/migration/frame9/services/cmpio_migrator.py` — `migrate`(:76)
