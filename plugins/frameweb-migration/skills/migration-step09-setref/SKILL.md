---
name: migration-step09-setref
description: FRAME9 마이그레이션 마법사 Step 9 — EMAX SCW210/SCF170을 WRKSET(저장 파라미터 바인딩)으로, open_trigger/SCC110을 WRKREF(워크셋 간 참조)로 변환한다. WRKSET 마이그레이션, SETREF, 저장 파라미터 바인딩, MASTER_DETAIL, GRID_SET, CONTROL_REF, wrkref, Step 9 관련 작업이나 wrkset_migrator.py·wrkref_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 9 — SETREF (WRKSET·WRKREF 생성)

- **WRKSET**: EMAX SCW210(SQL 파라미터)+SCW120+SCF170 → 저장(C/U/D) 시 각 파라미터가 어느 컴포넌트/워크셋에서 값을 얻는지의 바인딩
- **WRKREF**: FRMWRK.open_trigger + SCC110(Push) + SCF170(read_yn='1') → 워크셋 간 참조 관계(MASTER_DETAIL/GRID_SET/CONTROL_REF)

파이프라인 위치: [migration-step08-biztbl](../migration-step08-biztbl/SKILL.md) 다음, [migration-step10-coding](../migration-step10-coding/SKILL.md) 이전.

## 실행 방법

- UI: Migration 화면 → "Step 9: SETREF" 탭 — 진입 시 WRKSET/WRKREF 목록 로드
- 실행: `POST /api/migration9/migrate/{frm_id}/wrkset?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/wrkset-list?bis_id={bis_id}`

## 선행 조건

1. **WRKSQL(Step 4)** — sql_id 매핑. 없으면 "No WRKSQL found"로 중단
2. **FRMWRK(Step 3)** — wrk_type으로 source_type(GRIDSET/FIELDSET) 판정
3. WRKREF는 WRKCMP(Step 6)의 is_pk가 ref_fld 판정에 쓰인다
4. 조회전용(R만) 폼은 SCW210이 비어 정상적으로 0건 — 에러 아님

## 핵심 동작

- WRKSET: SCW210 파라미터별 1행. `sql_sq`→CRUDM(2→C/U, 3→D, R 제외). `target_fld`=SCW120 역매핑, `source_fld/source_ref`=SCF170(read_yn='0'). ctr_nm 점 표기(`f20.emp_no`)는 cross-workset 참조. `org_yn='1'`은 원본(_old) pass-through
- WRKREF: open_trigger→MASTER_DETAIL(ref_fld=WRKCMP PK), SCC110 join_ty='Push'→GRID_SET, SCF170 read_yn='1'의 `XX.cmpnm`→GRID_SET / `cmpnm`→CONTROL_REF

## 재실행 시 동작

두 마이그레이터 모두 `delete_existing`이 아니라 **`dry_run`** 파라미터를 쓴다. 실제 저장 시 항상 DELETE 후 전량 재삽입(full refresh)이라 재실행 안전.

## 함정 — 구형 계열의 한계

- **pymssql+psycopg2 직접 연결 계열**(자체 CLI 보유) — Step 4·7·8의 SQLAlchemy 계열과 인프라가 다르다
- **bis_id 적용이 불완전**: WRKSET은 DELETE/INSERT·frmwrk 조회에만 bis_id 적용, EMAX 로드·WRKSQL 매핑은 무시. **WRKREF는 bis_id 미지원**(frm_id로만 삭제·삽입) — 멀티 비즈 격리에 취약. 여러 비즈에 같은 폼을 이관할 때 주의
- WRKREF의 PK 추정 fallback이 위험: is_pk 없으면 첫 컬럼, 그마저 없으면 **'emp_no' 하드코딩**(HR 폼 가정) — 비HR 폼에서 잘못된 참조가 생길 수 있다. Step 6의 is_pk 정확성이 선행 조건
- cross-workset ref_wrk_id 명명이 두 갈래(`{frm_id}_{ref.upper()}` vs `wset_cd.upper()`)로 혼재 — 언더스코어 유무 확인
- WRKSET 예외 시 명시 롤백 없음(errors 기록 후 커밋 안 됨)

## 코드 참조

- `backend/migration/frame9/services/wrkset_migrator.py` — `migrate_form`(:251), DELETE 후 재삽입(:466)
- `backend/migration/frame9/services/wrkref_migrator.py` — `migrate_form`(:296), SCF170 REF 파싱(:107), PK fallback(:207)
