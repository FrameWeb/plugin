---
name: migration-step04-wrksql
description: FRAME9 마이그레이션 마법사 Step 4 — EMAX SCW100/SCW200의 T-SQL을 WRKSQL(워크셋 쿼리)+WRKSQL_TABLE(참조 테이블)로 변환한다. 타깃 방언(PG 변환 vs MSSQL 원문 보존)이 갈리는 핵심 단계. WRKSQL 마이그레이션, T-SQL 변환, ISNULL/TOP/getdate 변환, MSSQL 보존 모드, conversion_notes 경고, Step 4 관련 작업이나 wrksql_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 4 — WRKSQL (쿼리 변환·저장)

EMAX SCW100(SELECT) + SCW200(CRUD) → **WRKSQL + WRKSQL_TABLE**.
T-SQL을 타깃 방언에 맞게 변환(PG) 또는 보존(MSSQL)해 저장하는 핵심 단계다.

파이프라인 위치: [migration-step03-frmwrk](../migration-step03-frmwrk/SKILL.md) 다음, [migration-step05-wrkget](../migration-step05-wrkget/SKILL.md) 이전.
여기서 저장한 WRKSQL_TABLE이 [migration-step08-biztbl](../migration-step08-biztbl/SKILL.md)의 대상 테이블 목록이 된다.

## 실행 방법

- UI: Migration 화면 → "Step 4: WRKSQL" 탭
- API: `POST /api/migration9/migrate/{frm_id}/wrksql?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/wrksql-list?bis_id={bis_id}` (CRUDM 분포·경고 수 포함)
- 경고 조회: `GET /api/migration9/wrksql/warnings/{frm_cd}`

## 타깃 방언 분기 (ADR-0096, 이 단계의 핵심)

`dialect.get_business_db_type(db, bis_id)`가 BISDB의 `db_type`을 읽어 결정한다. UI에서 방언을 고르지 않는다.

| 타깃 | 동작 |
|------|------|
| PostgreSQL (기본) | 19단계 정규식 변환 전체 수행 + 테이블명 소문자화 |
| MSSQL (`db_type='mssql'`) | **보존 모드(preserve=True)** — 방언 변환·소문자화 생략, 원본 T-SQL 유지. 리네임 해석·경고 수집·출처 주석만 수행 |

`bis_id`가 비었거나 `"default"`면 PG로 판정된다 — **MSSQL 비즈로 돌릴 때 bis_id 누락이 가장 흔한 사고**(PG 변환이 걸려버림).

## 핵심 동작 (PG 변환 경로)

- SCW100 → crudm 'R', SCW200 → insert/update/delete_yn 플래그로 C/U/D 판정. `_test_` 워크셋 제외
- 변환 규칙: `@param→:param`, ISNULL→COALESCE, getdate→NOW, LEN→LENGTH, CHARINDEX→POSITION, CONVERT→TO_CHAR/CAST, TOP N→LIMIT, DATEADD/DATEDIFF, 문자열 `+`→`||`, andif/endif→`{{AND …}}`, UPDATE FROM 재작성, RenameResolver 치환
- SQL에서 테이블명 추출(FROM/JOIN/UPDATE/INSERT/DELETE) → RenameResolver 매핑 후 WRKSQL_TABLE 저장
- 미변환 패턴(dbo.*, exec, @@, sp_, 지역변수)은 `conversion_notes`(jsonb)에 경고 축적
- SQL마다 개별 commit — 한 건 실패해도 나머지는 저장됨

## 선행 조건 / 재실행

- EMAX SCW100/SCW200 접속. wrk_id는 wset_cd 대문자화로 독립 생성되지만 실무상 Step 3 완료 전제
- `delete_existing=True`: WRKSQL_TABLE→WRKSQL 순 삭제 후 재생성. insert는 `ON CONFLICT (bis_id,frm_id,wrk_id,crudm,execution_order) DO UPDATE`

## 함정

- **주석에 `:` 금지** — plpgsql_executor가 `:param`으로 오인 파싱한다. 출처 주석도 `-- Migrated from EMAX (T-SQL preserved for MSSQL target)`처럼 콜론 없이 쓴다
- alias·문자열 concat 변환은 단순 정규식이라 오변환 가능 — 변환 후 `conversion_notes`와 실제 쿼리를 반드시 검토
- 보존 모드의 테이블 추출(`lowercase=False`)은 "원본 케이스"가 아니라 **대문자 정규형**을 돌려준다(내부 `.upper()`). WRKSQL_TABLE 곁목록에만 쓰이므로 query 본문과 혼동하지 말 것
- SCW 로드 실패는 warning 후 빈 결과 — 소스가 비었는지 접속 실패인지 로그로 구분
- `list_available_sqls`가 존재하지 않는 `_load_scw100_data`를 호출하는 잔재 버그 있음

## 코드 참조

- `backend/migration/frame9/services/wrksql_migrator.py` — `migrate_form`(:769), `_convert_emax_sql_to_epic`(:158), `_collect_conversion_warnings`(:481), `_extract_tables_from_sql`(:582)
- `backend/migration/frame9/services/dialect.py` — `get_business_db_type`(:23)
- 검증 기록: `docs/raw/done/step4-wrksql-mssql-preserve.md`
