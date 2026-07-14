---
name: migration-step11-rename
description: FRAME9 마이그레이션 마법사 Step 11 — 레거시 MSSQL 테이블/컬럼명을 새 이름으로 치환하는 migrename 매핑 사전을 관리한다. Step 4(SQL 치환)·Step 8(테이블 생성명)이 이 매핑을 소비한다. 리네임 매핑, migrename, bulk-import, RenameResolver, is_active, MSSQL에서 가져오기, Step 11 관련 작업이나 치환이 안 될 때 이 스킬을 참조한다.
---

# Step 11 — RENAME (테이블/컬럼 리네임 매핑)

레거시 MSSQL 테이블/컬럼명 → 새 이름 매핑 사전(**migrename** 테이블).
폼 단위 단계가 아니라 전역 사전 관리다(폼 선택 없이 사용 가능한 유일한 탭).
소비자는 **RenameResolver** — Step 4(WRKSQL 본문 치환)와 Step 8(물리 테이블 생성명), business_data_migrator가 쓴다.

파이프라인 위치: 마지막 탭이지만 **실제로는 Step 4·8보다 먼저 정비해야 효과가 있다**.
전체 순서는 [migration-frame9](../migration-frame9/SKILL.md) 참조.

## 화면 조작

1. "Step 11: RENAME" 탭 — Rename Mapping (ROCA) 목록, 테이블명 검색
2. **MSSQL에서 가져오기**(bulk-import) — EMAX `INFORMATION_SCHEMA.TABLES`를 읽어 미등록 테이블만 추가
3. Target Table 인라인 편집(onBlur 저장), Active 체크박스, Memo, 삭제

## API

- 목록: `GET /api/migration/rename/?site_id=ROCA`
- 수정: `PUT /api/migration/rename/{id}` — `{target_table, target_column, is_active, memo}`
- 일괄 가져오기: `POST /api/migration/rename/bulk-import` — `{site_id}`

주의: prefix가 `/api/migration/rename`(migrename_router)이다. `/api/migration-bnd`(BindingSet 스테이징, `backend/migration/migrename/` 디렉토리)는 **이름만 비슷한 별개 도구**다.

## bulk-import 동작 (재실행 안전)

- 이미 테이블 레벨 매핑이 있으면 skip — 기존 매핑 보존, 신규 테이블만 추가
- 신규는 `target_table = 원본.lower()`, **`is_active=False`(비활성)** 로 생성

## 함정

- **bulk-import 직후엔 전부 비활성이라 치환이 전혀 일어나지 않는다** — RenameResolver는 `is_active=True`인 매핑만 로드한다. target을 다듬고 Active를 켜야 Step 4·8에 반영된다. "리네임했는데 안 먹혀요"의 원인 1순위
- 미매핑 테이블의 `resolve_table` 결과는 `source.lower()` — PG 타깃에선 자연스럽지만, 의도한 새 이름이 아니면 매핑 등록 필요
- `resolve_sql`은 단어경계·긴 이름 우선·대소문자 무시 정규식 치환 — alias는 안 건드리지만 문자열 리터럴 내부까지 완벽 회피는 보장하지 않는다. 치환 후 쿼리 검토 필요
- 매핑을 바꾸면 **Step 4·8을 다시 돌려야** 산출물에 반영된다(이미 저장된 WRKSQL·테이블은 소급 변경되지 않음)
- site_id는 프론트에서 'ROCA' 고정

## 코드 참조

- `backend/api/migrename_router.py:130`(bulk-import), `:56`(목록)
- `backend/migration/shared/rename_resolver.py:62`(resolve_sql)
- `backend/migration/frame8/models.py:85`(MIGRENAME 모델)
- `frontend/src/migration/frame9/MigrationPage.tsx:2859`(탭 11 UI)
