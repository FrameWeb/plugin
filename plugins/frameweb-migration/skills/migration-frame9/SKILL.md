---
name: migration-frame9
description: FRAME9 → FrameWeb 마이그레이션 마법사(12단계) 전체 지도 — 단계 순서·의존 관계·타깃 방언 결정·공용 인프라를 안내하고 각 단계 스킬(migration-step00~11)로 연결한다. FRAME9 마이그레이션, EMAX 폼 이관, 마이그레이션 마법사, 12스텝, 폼 마이그레이션 순서, 어떤 단계부터 돌려야 하는지 물을 때, 또는 backend/migration/frame9 코드를 다룰 때 반드시 이 스킬부터 참조한다.
---

# FRAME9 → FrameWeb 마이그레이션 마법사 (전체 지도)

EMAX 레거시(MSSQL) 폼 자산을 FrameWeb 폼 메타(시스템 DB=PG)와 비즈 테이블(비즈 DB=PG/MSSQL)로 옮기는 12단계 마법사.
화면: Dev > Development > Migration. API prefix: `/api/migration9`.
코드: `backend/migration/frame9/` + `frontend/src/migration/frame9/MigrationPage.tsx`.

## 단계 순서와 산출물

| 단계 | 산출물 | 단위 | 스킬 |
|------|--------|------|------|
| Step 0 MAPPING | migmap (legacy_type→cmp_id 사전) | 전역 | [migration-step00-mapping](../migration-step00-mapping/SKILL.md) |
| Step 1 BISFRM | BISFRM + MIGCMP (.Designer.vb 임포트) | 폼 | [migration-step01-bisfrm](../migration-step01-bisfrm/SKILL.md) |
| Step 2 FRMCMP | FRMCMP + BISFRM (컴포넌트 트리) | 폼 | [migration-step02-frmcmp](../migration-step02-frmcmp/SKILL.md) |
| Step 3 FRMWRK | FRMWRK + placeholder WRKSQL (워크셋) | 폼 | [migration-step03-frmwrk](../migration-step03-frmwrk/SKILL.md) |
| Step 4 WRKSQL | WRKSQL + WRKSQL_TABLE (쿼리 변환) | 폼 | [migration-step04-wrksql](../migration-step04-wrksql/SKILL.md) |
| Step 5 WRKGET | WRKGET (조회 파라미터) | 폼 | [migration-step05-wrkget](../migration-step05-wrkget/SKILL.md) |
| Step 6 WRKCMP | WRKCMP (컬럼-필드 매핑) | 폼 | [migration-step06-wrkcmp](../migration-step06-wrkcmp/SKILL.md) |
| Step 7 BISPOP | BISPOP+POPCOD/POPSQL/POPIO/POPCMP (입력보조) | 팝업 | [migration-step07-bispop](../migration-step07-bispop/SKILL.md) |
| Step 8 BIZTBL | 비즈 DB 테이블 구조+데이터 | 테이블 | [migration-step08-biztbl](../migration-step08-biztbl/SKILL.md) |
| Step 9 SETREF | WRKSET(저장 바인딩) + WRKREF(참조 관계) | 폼 | [migration-step09-setref](../migration-step09-setref/SKILL.md) |
| Step 10 Coding | BISFRM.vbcode (원본 VB 로직 보관) | 폼 | [migration-step10-coding](../migration-step10-coding/SKILL.md) |
| Step 11 RENAME | migrename (테이블/컬럼 리네임 사전) | 전역 | [migration-step11-rename](../migration-step11-rename/SKILL.md) |

보조(화면 탭 없음): CMPIO — `POST /migrate/{frm_id}/cmpio`, Step 7 산출물(POPIO)과 WRKCMP를 잇는 브리지. Step 7 스킬 참조.

## 의존 관계 (실행 순서의 근거)

- **전역 사전 먼저**: Step 0(migmap)과 Step 11(migrename, Active 켜기)은 폼 단계 전에 정비해야 효과가 있다. Step 11이 탭 순서상 마지막일 뿐, 매핑은 Step 4·8이 소비한다
- 폼 단위 강한 의존: 1 → 2 → 3 → 4 → 5(4의 sql_id 필수) → 6 → 7 → 8(4의 WRKSQL_TABLE 필수) → 9(4·6 필수)
- Step 10은 독립(BISFRM만 있으면 언제든)
- 상태 확인: `GET /api/migration9/migrate/{frm_id}/status` — step1~6 카운트 반환

## 타깃 방언 결정 (ADR-0096)

`dialect.get_business_db_type(db, bis_id)`가 BISDB `db_type`으로 자동 판정한다. UI에서 방언을 고르지 않는다.

- 방언 분기가 있는 단계는 **Step 4(T-SQL 보존 vs PG 변환)와 Step 8(MSSQL DDL vs PG DDL)** 뿐. 나머지 산출물은 시스템 DB(PG) 폼 메타라 방언 중립
- **bis_id 누락·"default"면 PG로 판정** — MSSQL 비즈 이관 시 bis_id를 빠뜨리면 PG 변환이 걸리는 것이 최다 사고 유형

## 공용 인프라

- **소스(EMAX MSSQL) 접속**: `migsite` 테이블이 단일 출처(`mssql_connector.get_mssql_connector(site_id)`, 기본 'ROCA', 비밀번호는 encryption 복호화). 미등록 시 `MIGRATION_MSSQL_*` 환경변수 fallback
- **제외 목록**: `mig_except` 테이블(frm/tbl/pop/func) — Step 7·8이 여기 등록된 항목을 조용히 skip한다. "왜 안 만들어지지?"의 단골 원인
- **리네임**: `migration/shared/rename_resolver.py` — is_active=True 매핑만 로드
- 인프라 두 계열 공존: 최신(SQLAlchemy Session — Step 2~4·7·8·cmpio) vs 구형(pymssql/psycopg2 직접 — Step 6·9·business_*). 구형은 bis_id 격리가 불완전하다

## UnSkein 진행 절차 (단계 = 카드 = 데이터 확인 게이트)

마이그레이션은 UnSkein PLANNER가 분석해 카드로 내리고, 실행기가 카드를 선점해 **한 단계씩** 진행한다.
단계를 건너뛰거나 여러 단계를 한 카드에 묶지 않는다 — 단계마다 데이터를 확인해야 다음으로 간다.

**단계 → 카드 매핑 규약**

- 배치(폼 묶음) 선행 카드 2장: Step 0(migmap 정비 — Unmapped 0 확인)과 Step 11(migrename bulk-import + Active 정비). 전역 사전이므로 폼 카드보다 먼저 완료한다
- 폼 1개 = 카드 묶음 1세트: Step 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 순서로 의존성을 걸어 등록한다(Step 10은 독립 카드). 등록 절차 자체는 unskein-scope/unskein-wbs 스킬을 따른다
- 각 카드의 plan_doc은 해당 단계 스킬(migration-stepNN-*)을 참조로 걸고, 완료 기준은 아래 데이터 확인으로 한다

**카드 완료 기준 (데이터 확인)**

- 공통: 해당 단계 list API(frmcmp-tree, frmwrk-list, wrksql-list, wrkget-list, wrkcmp-list, wrkset-list…)로 건수가 소스와 대응하는지 확인. `GET /migrate/{frm_id}/status`로 step1~6 카운트 교차 확인
- Step 4: `conversion_notes` 경고를 전부 읽고 처리 방침 기록. MSSQL 타깃이면 원문 보존 여부 표본 확인
- Step 8: 타깃 DB 실테이블 행수를 소스와 대조
- 최종 카드: 해당 비즈 테넌트에서 조회·저장 한 바퀴 실동작(브라우저, 콘솔 에러 0)
- 확인 결과(건수·경고·불일치)는 카드에 기록하고 넘어간다

**스킬은 살아있는 문서** — 진행 중 새 함정·절차 변경을 발견하면 해당 단계 스킬 SKILL.md를 그 자리에서 고치고 플러그인(frameweb-plugin 저장소)으로 배포한다. 스킬 수정도 산출물이다.

## 검증 관행

- 각 단계 실행 후 해당 list API(frmcmp-tree, frmwrk-list, wrksql-list…)로 건수·경고 확인
- Step 4는 `conversion_notes` 경고, Step 8은 소스 행수 대조까지가 완료 기준
- 마이그레이션된 폼의 최종 검증은 해당 비즈 테넌트에서 조회·저장 한 바퀴 실동작(브라우저)
- 스펙·검증 기록: `docs/specs/frame9-mssql-migration.md`, `docs/raw/done/step4-*.md`, `docs/raw/done/step8-*.md`
