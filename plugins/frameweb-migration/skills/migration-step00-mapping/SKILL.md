---
name: migration-step00-mapping
description: FRAME9 마이그레이션 마법사 Step 0 — VB 레거시 컨트롤 타입을 FrameWeb 코어 컴포넌트(cmp_id)+에디터 타입으로 매핑하는 migmap 사전을 관리한다. 레거시 타입 매핑, migmap, Legacy Types Mapped/Unmapped, 컴포넌트 매핑 패널, Step 0 관련 작업이나 매핑 누락으로 Step 2 변환이 빠질 때 이 스킬을 참조한다.
---

# Step 0 — MAPPING (레거시 타입 → 코어 컴포넌트 매핑 사전)

VB 레거시 컨트롤 타입(legacy_type) → **migmap**(cmp_id + editor_type).
이후 단계가 쓰는 변환 사전을 정의하는 준비 단계다. 폼별 작업이 아니라 전역 사전 관리다.

파이프라인 위치: 마법사의 첫 탭. 이 매핑은 [migration-step02-frmcmp](../migration-step02-frmcmp/SKILL.md)가 소비한다.
전체 순서는 [migration-frame9](../migration-frame9/SKILL.md) 참조.

## 화면 조작 (3열 레이아웃)

1. 좌열 Legacy Types — Mapped/Unmapped 배지·컴포넌트 수 표시. 상단 Form Group 셀렉트로 폼별 필터
2. 중앙 Mapping Panel — 선택한 legacy/core 확인, Editor Type 드롭다운(TEXT/DATE/NUMBER/EMAIL/DECIMAL), Map/Unmap 버튼
3. 우열 Core Components — cmpmst 목록

legacy 선택 → core 선택 → Map. 같은 키(legacy_type+cmp_id)면 upsert로 덮어쓴다.

## API

- 목록: `GET /api/migration9/maps`
- 등록(upsert): `POST /api/migration9/maps` — `{legacy_type, cmp_id, editor_type}`
- 수정/삭제: `PUT|DELETE /api/migration9/maps/{map_id}`
- 코어 컴포넌트 목록: `GET /api/forms/components` (cmpmst)

저장 테이블: **migmap** (`backend/migration/frame8/models.py:42` MIGMAP).

## 함정

- **Unmapped로 남은 legacy_type은 Step 2에서 조용히 변환 누락된다** — Step 2 실행 전 좌열의 Unmapped 배지를 0으로 만드는 것이 원칙. migmap에 없으면 property_transformer의 TYPE_MAPPING fallback을 타고, 거기에도 없으면 skip
- legacy_type은 site 구분 없는 전역 키 — 사이트별로 다른 매핑이 필요해지면 구조 변경이 선행돼야 한다
- cmp_id는 CMPMST(use_yn='Y')에 실재해야 Step 2가 받아준다

## 코드 참조

- `frontend/src/migration/frame9/MigrationPage.tsx:271`(handleMap), `:960`(탭 0 UI)
- `backend/migration/frame9/router.py:1031`(maps CRUD)
