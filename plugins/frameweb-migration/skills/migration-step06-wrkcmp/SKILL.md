---
name: migration-step06-wrkcmp
description: FRAME9 마이그레이션 마법사 Step 6 — EMAX SCW120/SCF170/SCC100을 WRKCMP(워크셋 컴포넌트·컬럼-필드 매핑)로 변환한다. editor_type·is_pk·optionSource 결정 로직 포함. WRKCMP 마이그레이션, 컬럼 매핑, editor_type, is_pk 판정, optionSource, required codes, Step 6 관련 작업이나 wrkcmp_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 6 — WRKCMP (워크셋 컴포넌트 매핑)

EMAX SCW120(워크셋-필드) + SCF170(필드-컨트롤 바인딩) + SCC100(컨트롤 속성) → **WRKCMP**.
워크셋의 각 필드가 어느 컴포넌트·DB 컬럼에 대응하는지(라벨·필수·PK·에디터 타입 포함)를 만드는 단계다.

파이프라인 위치: [migration-step05-wrkget](../migration-step05-wrkget/SKILL.md) 다음, [migration-step07-bispop](../migration-step07-bispop/SKILL.md) 이전.
이 탭에서 필요한 팝업 코드 목록(required-codes)도 조회한다 — Step 7의 입력이 된다.

## 실행 방법

- UI: Migration 화면 → "Step 6: WRKCMP" 탭
- API: `POST /api/migration9/migrate/{frm_id}/wrkcmp?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/wrkcmp-list?bis_id={bis_id}` (워크셋별 분포 포함)
- 필요 코드 조회: `GET /api/migration9/forms/{frm_id}/required-codes` (SCC100 popup_id 기반 → Step 7 입력)

## 선행 조건

1. **FRMWRK 존재(Step 3)** — 워크셋 없으면 즉시 에러
2. FRMCMP(Step 2) — label·cmp_id·optionSource 보강의 출처
3. EMAX SCW120/SCF170/SCC100/SCL100 + BISPOP 접속

## 핵심 동작

- 워크셋별 SCW120 필드 순회: `fcmp_id=fld_nm`, `data_field=col_nm`, SCF170으로 fld_nm→ctr_nm 바인딩 후 SCC100 속성 조인
- `col_seq`는 SCW120.reg_no 우선. is_required/is_readonly/default_value/label은 SCC100 우선
- editor_type: SCC100.ctr_ty 매핑 기본, popup_id 있으면 SCL100.main_cd 기반(CODE/LOOKUP/POPUP) override
- optionSource 결정 순서: FRMCMP.optionSource → SCL100 popup_id→BISPOP pop_id 이름매칭 → props.popup_cd
- MSSQL INFORMATION_SCHEMA로 PK 컬럼 조회해 `is_pk` 판정. cmp_id는 FRMCMP 값 우선, 없으면 추론(popup→EMAX_SELECT, `_yn`→EMAX_CHECKBOX, 그 외 EMAX_TEXT_INPUT)

## 재실행 시 동작

dry_run이 아니면 WRKCMP를 frm_id+bis_id로 삭제 후 재삽입. insert는 `ON CONFLICT (bis_id,frm_id,wrk_id,fcmp_id) DO UPDATE`. 전체 단일 트랜잭션 — 예외 시 롤백.

## 함정

- **이 마이그레이터만 SQLAlchemy Session이 아니라 psycopg2 직접 연결**(pg_config 환경변수·Cloud SQL 소켓 분기) — 다른 스텝과 커넥션 모델이 다르다. DB 연결 문제 디버깅 시 먼저 확인할 지점
- SCW120에 필드 없는 워크셋은 조용히 skip — `skipped_fields` 리포트는 항상 빈 배열(미완성)이라 믿지 말 것
- SCW120/SCF170/SCC100 로드 실패는 warning 후 빈값 → 매핑이 조용히 누락될 수 있다
- Step 6의 editor_type 체계(EMAX_* 계열)는 Step 2 property_transformer의 CMP_* 체계와 **다른 매핑 테이블**이다 — 한쪽만 고치면 화면과 그리드가 어긋난다
- is_pk가 정확해야 그리드 PK 변경 저장이 동작한다(R쿼리 원본키 별칭 규약과 연동)

## 코드 참조

- `backend/migration/frame9/services/wrkcmp_migrator.py` — `migrate_form`(:621), `_load_scw120_data`(:435), `_load_scf170_bindings`(:486), `_build_popup_id_mapping`(:300)
