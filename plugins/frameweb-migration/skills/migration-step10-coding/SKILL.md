---
name: migration-step10-coding
description: FRAME9 마이그레이션 마법사 Step 10 — 폼의 원본 VB 소스 코드(.vb)를 BISFRM.vbcode에 보관해 form_script 수동 변환의 참고 자료로 쓴다. VB 코드 업로드, vbcode, Coding 탭, 원본 로직 열람, Step 10 관련 작업 시 이 스킬을 참조한다.
---

# Step 10 — Coding (원본 VB 코드 보관)

폼의 원본 VB 소스(`.vb`) → **BISFRM.vbcode** 컬럼에 원문 저장.
자동 변환 대상이 아니다 — 폼 이벤트 로직(form_script)을 수동으로 옮길 때 열람하는 참고 자료다.

파이프라인 위치: [migration-step09-setref](../migration-step09-setref/SKILL.md) 다음, [migration-step11-rename](../migration-step11-rename/SKILL.md) 이전.

## 화면 조작

1. "Step 10: Coding" 탭 진입 — 선택된 폼의 vbcode 자동 로드
2. `.vb` 파일 선택 → Upload (기존 내용을 통째로 덮어씀)
3. 저장된 코드는 줄 수·글자 수와 함께 본문 표시, 삭제 버튼 제공

## API

- 업로드: `POST /api/migration9/forms/{frm_id}/vbcode` (multipart, `.vb`만 허용)
- 조회: `GET /api/migration9/forms/{frm_id}/vbcode`
- 삭제: `DELETE /api/migration9/forms/{frm_id}/vbcode` (vbcode=NULL)

## 함정

- **Step 1의 vbdesign과 다른 컬럼이다** — vbdesign=화면 레이아웃 정의(.Designer.vb), vbcode=이벤트 로직(.vb). 같은 폼의 두 파일을 각각 올린다
- **BISFRM이 먼저 있어야 한다**(없으면 404) — Step 1 임포트 또는 create-bisfrm 선행
- frm_id를 `.upper()`로 정규화해 조회한다 — Step 1에서 소문자 frm_id로 만든 폼은 여기서 404가 난다
- 업로드는 누적이 아니라 전체 덮어쓰기

## 코드 참조

- `backend/migration/frame9/router.py:1917`(POST), `:1974`(GET), `:2009`(DELETE)
- `frontend/src/migration/frame9/MigrationPage.tsx:2782`(탭 10 UI)
