---
name: migration-step01-bisfrm
description: FRAME9 마이그레이션 마법사 Step 1 — VB .Designer.vb 파일을 파싱해 BISFRM(폼 컨테이너)+MIGCMP(레거시 컴포넌트 스냅샷)를 생성한다. Designer 업로드, import-designer, vb-parser, reparse-from-vbdesign, import_option(deleteupdate/insert/update), 빈 BISFRM 생성(emptymig), Step 1 관련 작업이나 폼 임포트 문제 디버깅 시 이 스킬을 참조한다.
---

# Step 1 — BISFRM (.Designer.vb 임포트)

VB `.Designer.vb` 파일 → **BISFRM(vbdesign 원문 보관, status='migration') + MIGCMP(site_id='ROCA')**.
폼 하나를 마이그레이션 파이프라인에 올리는 진입 단계다.

파이프라인 위치: [migration-step00-mapping](../migration-step00-mapping/SKILL.md) 다음, [migration-step02-frmcmp](../migration-step02-frmcmp/SKILL.md) 이전.

## 화면 조작

1. 상단 VB Parser 상태 배지 확인(online/offline) — `GET /api/migration9/vb-parser/health`
2. Form ID 입력(파일명 `XXX.Designer.` 패턴에서 자동 추출됨), `.Designer.vb` 파일 선택
3. Import Option 선택: Delete&Update(기본) / Insert Only / Update Only → Import
4. 파일 없이 Form ID만 입력하고 Import하면 **저장된 vbdesign 재파싱**(reparse)
5. 하단 Imported Forms 테이블에서 폼별 컴포넌트 수·캔버스 크기 확인, 행 삭제 가능

## API

- 업로드 파싱: `POST /api/migration9/migcmp/import-designer?import_option=deleteupdate[&form_id=...]` (multipart file)
- 재파싱: `POST /api/migration9/migcmp/reparse-from-vbdesign/{frm_id}?import_option=...`
- Designer 없는 폼의 최소 BISFRM만 생성: `POST /api/migration9/emptymig/create-bisfrm/{frm_cd}` (migtarget에서 폼명 조회, 캔버스 1200×800)
- 폼 목록/삭제: `GET /api/migration9/forms`, `DELETE /api/migration9/forms/{frm_cd}`

## import_option 의미 (재실행 동작)

| 옵션 | 동작 |
|------|------|
| `deleteupdate` (기본) | 해당 frm_cd의 **MIGCMP 전량 삭제(site 무관)** 후 재생성 |
| `insert` | 기존 migcmp_id는 skip, 새 컴포넌트만 추가 |
| `update` | 기존 것만 갱신, 없으면 skip |

BISFRM은 upsert(org_id='personal_1' 하드코딩, vbdesign 원문·캔버스 크기 저장).

## 함정

- **vb-parser 오프라인이어도 Import는 진행된다** — Python 정규식 폴백으로 자동 강등(source='designer_fallback'). 폴백은 속성·부모관계·panel_index 추출이 제한적이라 Step 2 결과 품질이 떨어진다. 배지가 offline이면 파서부터 살릴 것
- form_id 유도 우선순위: 쿼리 파라미터 > Node 파서 결과/파일명 > 본문. 못 찾으면 400
- **frm_id 대소문자 불일치 주의** — import-designer는 원문 그대로, `create-bisfrm`·vbcode(Step 10)는 `.upper()` 정규화. 소문자로 임포트한 폼은 Step 10에서 404가 날 수 있다
- MIGCMP의 migcmp_id는 `{frm_cd}.{컨트롤명}`, site_id='ROCA' 하드코딩

## 코드 참조

- `backend/migration/frame9/router.py:700`(import-designer), `:884`(reparse), `:170`(_call_vb_parser), `:1721`(create-bisfrm)
- `frontend/src/migration/frame9/MigrationPage.tsx:1155`(탭 1 UI), `:1223`(Import 분기)
