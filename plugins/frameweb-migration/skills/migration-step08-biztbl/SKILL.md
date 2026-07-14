---
name: migration-step08-biztbl
description: FRAME9 마이그레이션 마법사 Step 8 — 폼이 참조하는 비즈니스 테이블을 EMAX MSSQL에서 타깃 비즈 DB(PG 또는 MSSQL)로 구조+데이터 이관한다. BIZTBL 마이그레이션, 비즈 테이블 생성, MSSQL DDL 보존, IDENTITY_INSERT, delete_existing, required tables, Step 8 관련 작업이나 biztbl_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 8 — BIZTBL (비즈니스 테이블 이관)

WRKSQL_TABLE이 가리키는 테이블을 EMAX MSSQL → **타깃 비즈 DB**(bis_id의 BISDB 경유)로 구조+데이터 이관.
폼이 실제로 돌 수 있게 업무 테이블을 만드는 단계다.

파이프라인 위치: [migration-step07-bispop](../migration-step07-bispop/SKILL.md) 다음, [migration-step09-setref](../migration-step09-setref/SKILL.md) 이전.
대상 목록은 [migration-step04-wrksql](../migration-step04-wrksql/SKILL.md)이 채운 WRKSQL_TABLE에서 나온다.

## 실행 방법

- UI: Migration 화면 → "Step 8: BIZTBL" 탭 — 탭 진입 시 필요 테이블 존재 여부 자동 확인
- 필요 목록: `GET /api/migration9/biztbl/required/{frm_id}`
- 존재 확인: `POST /api/migration9/biztbl/check-exists?bis_id={bis_id}` (테이블명 배열)
- 미리보기: `GET /api/migration9/biztbl/preview/{table_name}`
- 실행: `POST /api/migration9/biztbl/migrate/{table_name}?delete_existing=false` 또는 `POST /biztbl/migrate-multiple` — `{table_names, delete_existing, bis_id}`

## 타깃 방언 분기 (ADR-0096)

`get_business_db_type(db, bis_id)`로 판정. `bis_id="default"`면 시스템 DB(PG)에 생성된다.

| 타깃 | DDL |
|------|-----|
| PostgreSQL | 소문자 식별자 + `_map_mssql_to_pg_type` 타입 변환, IDENTITY 없음 |
| MSSQL | `[대괄호]` 식별자 원형 케이스 보존 + `_map_mssql_source_type` 타입 보존(nvarchar 바이트/2 문자수 복원, -1→max) + IDENTITY(1,1) 부착 |

## 재실행 시 동작

- `delete_existing=False`(기본): 테이블 이미 존재하면 `success=False, "Table already exists"` — 건드리지 않음
- `delete_existing=True`: DROP 후 재생성. MSSQL은 `DROP TABLE IF EXISTS [name]`, PG는 소문자·대문자쌍 모두 `DROP ... CASCADE`

## 함정

- **IDENTITY 이식 경로는 단일 트랜잭션** — `SET IDENTITY_INSERT ON/OFF` 세션 설정이 커밋 시 커넥션 반납으로 풀리지 않도록, 배치 중간 커밋 없이 마지막 1회만 커밋한다. 이 경로를 수정할 때 배치 커밋을 넣으면 깨진다
- **mig_except 가드** — `mig_except.tbl`(target 소문자명)에 있으면 skipped
- 개별 행 INSERT 실패는 warning만 남기고 계속 → 완료 후 **행수 대조 필수**(AC-3: 소스와 일치)
- 테이블 존재 캐시(`_epic_tables_cache`)가 bis_id별로 분리돼 있지 않아, 서로 다른 bis_id를 연속 처리하면 첫 캐시가 재사용될 수 있다
- 리네임은 RenameResolver(활성 매핑만) 적용 — Step 11에서 매핑을 켜두지 않으면 원본 소문자명으로 생성된다
- `business_table_migrator.py`/`business_data_migrator.py`는 **별개 구형 CLI 계열**(bis_id 미전달 버그, 무조건 CASCADE DROP 등) — 마법사 Step 8은 `biztbl_migrator.py`다. 혼동하지 말 것

## 코드 참조

- `backend/migration/frame9/services/biztbl_migrator.py` — `migrate_table`(:273, 방언·IDENTITY 분기), `_map_mssql_source_type`(:552), `list_required_tables`(:96), `check_tables_exist`(:152)
- 검증 기록: `docs/raw/done/step8-biztbl-mssql-ddl.md`
