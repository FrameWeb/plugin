---
name: migration-step02-frmcmp
description: FRAME9 마이그레이션 마법사 Step 2 — MIGCMP(레거시 컴포넌트)를 FRMCMP(폼 컴포넌트 트리)+BISFRM(폼 캔버스)으로 변환한다. 폼 화면 레이아웃 생성 단계. FRMCMP 마이그레이션, 컴포넌트 트리 생성, SplitContainer 패널, 폼 캔버스 생성, migmap 매핑, Step 2 관련 작업이나 frmcmp_migrator.py 수정·디버깅 시 이 스킬을 참조한다.
---

# Step 2 — FRMCMP (폼 컴포넌트 트리 생성)

MIGCMP(레거시 컴포넌트, site_id='ROCA') + EMAX SCC100 → **FRMCMP + BISFRM**.
폼의 화면 레이아웃(컴포넌트 트리·캔버스)을 만드는 단계다.

파이프라인 위치: [migration-step01-bisfrm](../migration-step01-bisfrm/SKILL.md) 다음, [migration-step03-frmwrk](../migration-step03-frmwrk/SKILL.md) 이전.
전체 순서는 [migration-frame9](../migration-frame9/SKILL.md) 참조.

## 실행 방법

- UI: Dev > Development > Migration → "Step 2: FRMCMP" 탭 → 마이그레이션 실행 버튼
- API: `POST /api/migration9/migrate/{frm_id}/frmcmp?bis_id={bis_id}`
- 결과 확인: `GET /api/migration9/forms/{frm_id}/frmcmp-tree?bis_id={bis_id}` (트리 표시)
- 상태 확인: `GET /api/migration9/migrate/{frm_id}/status` → `step2_frmcmp.count`

## 선행 조건

1. MIGCMP에 해당 `frm_cd`(site_id='ROCA') 레코드 존재 — Step 0에서 .Designer 파일 임포트 완료
2. migmap(`legacy_type → cmp_id`)·cmpmst 마스터 정비 — Step 0의 MAPPING 탭
3. EMAX MSSQL 접속 가능(SCC100 — label·optionSource·defaultValue 병합용)

## 핵심 동작

- migmap으로 `legacy_type → cmp_id` 매핑, CMPMST(use_yn='Y')에 없으면 해당 컴포넌트 skip
- SplitContainer는 항상 Panel0/Panel1 2패널 자동 생성. panel_index 슬롯에 ePanel/Panel이 있으면 그 노드를 split panel로 승격
- PropertyTransformer로 VB 속성 변환. `layout_mode=fill`은 Container/Display 그룹만 허용(Input/Action은 절대좌표)
- SCC100/SCL100 조인으로 label(title)·optionSource(sub_cd)·defaultValue(def_text) 병합. Grid는 `bound_wrk_id={frm}_{fcmp.upper()}` 자동 설정
- 부모 체인으로 depth 재계산 → 위상정렬로 부모 먼저 insert → 후처리로 단일 Panel 붕괴

## 재실행 시 동작

`delete_existing=True`(기본): FRMCMP 전체 삭제 후 재생성. BISFRM은 `ON CONFLICT (bis_id, frm_id) DO UPDATE`(upsert).
**delete를 끄면 fcmp_id(원본 cmp_nm 그대로) PK 충돌** — insert 실패가 경고로만 찍히고 넘어간다.

## 함정

- **개별 insert 실패를 warning print로 삼키고 계속 진행** → 부분 마이그레이션 상태가 조용히 생길 수 있다. 실행 후 frmcmp-tree의 total_count를 MIGCMP 개수와 대조할 것
- SCC100 로드 실패는 warning 후 빈 dict로 진행 — label·기본값이 통째로 빠져도 성공처럼 보인다
- **Orientation은 VB↔FrameWeb 의미가 반대** — VB Horizontal(분할선 가로=상하분할) → vertical. property_transformer가 의도적으로 뒤집으므로 "버그"로 오인해 되돌리지 말 것
- depth 사이클 감지 시 ValueError → 전체 rollback
- MSSQL은 소스(SCC100)로만 쓰인다. 이 단계에 SQL 방언 분기는 없다(타깃은 시스템 DB PG 고정)

## 코드 참조

- `backend/migration/frame9/services/frmcmp_migrator.py` — `migrate_form`(:165), `_transform_migcmp_to_frmcmp`(:517), `_create_split_panels`(:384), `_collapse_splitpanel_single_panel`(:849)
- `backend/migration/frame9/services/property_transformer.py` — Step 2 전용. TYPE_MAPPING(migmap 미매핑 시 fallback), `_transform_orientation`(반전 로직)
