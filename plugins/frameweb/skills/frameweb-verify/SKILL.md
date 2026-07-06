---
name: frameweb-verify
description: >
  폼 라이프사이클 6단계: 런타임 검증. 모리에게 테스트 의뢰서를 자동 생성한다.
  배포된 폼이 ERP에서 정상 동작하는지 UI 검증을 위임.
  Trigger: 폼 검증, 테스트 의뢰, 모리 테스트, frameweb-verify, 런타임 검증,
  ERP 테스트, 검증해줘, 모리한테 테스트.
user-invocable: true
---

# Form Verify — 모리 테스트 의뢰서 생성

배포된 폼의 ERP 런타임 검증을 모리에게 위임한다.

## 도메인 지식

Epic Framework는 메타데이터 기반 로우코드 폼 플랫폼이다. 폼 = bisfrm 행 + frmcmp 컴포넌트 메타데이터. React/Vue 같은 frontend 프레임워크 개념 아님 — 모든 폼은 DB 메타 row가 정의하고 런타임이 동적 렌더링한다. 핵심 테이블: bisfrm(폼 마스터), frmcmp(컴포넌트), frmbnd(바인딩), bndquery(쿼리), bndcmp(바인딩 컴포넌트), bndpull(조회 매핑), bndsave(저장 매핑). 시스템 DB(`${FRAMEWEB_DB_NAME}`)와 비즈니스 DB(bis_id별 격리)가 분리되어 있다.

## 라이프사이클 위치

```
frameweb-require → frameweb-canvas → frameweb-bindchain → frameweb-binding → frameweb-load → frameweb-deploy → **frameweb-verify**
```

## 환경

| 항목 | 값 |
|------|-----|
| 시스템 DB | ${FRAMEWEB_DB_HOST}:${FRAMEWEB_DB_PORT} / ${FRAMEWEB_DB_USER} / ${FRAMEWEB_DB_PASSWORD} / ${FRAMEWEB_DB_NAME} |
| 의뢰서 경로 | /mnt/z/Obsidian/browser-test/request/ |
| 로컬 ERP | ${FRAMEWEB_APP_URL}/erp/form/{frm_id}?bis_id={bis_id} |
| 프로덕션 ERP | {bis_id}.erp.mupai.studio |
| 로그인 | ${FRAMEWEB_LOGIN_USER} / ${FRAMEWEB_LOGIN_PW} |

---

## 호출

```
/frameweb-verify {frm_id} {version} {bis_id}
```

## 실행 절차

### Step 1: 폼 정보 수집

```sql
-- 시스템 DB
SELECT frm_id, frm_nm, version, status FROM bisfrm
WHERE frm_id='{frm_id}' AND bis_id='{bis_id}';

-- 바인딩 구조 (체인 파악)
SELECT bnd_id, bnd_type, open_trigger, open_sq, save_sq, trigger_yn
FROM frmbnd WHERE frm_id='{frm_id}' AND bis_id='{bis_id}'
ORDER BY open_sq;

-- 메뉴 경로 (비즈니스 DB)
SELECT menu_nm, frm_id FROM mnumst
WHERE frm_id='{frm_id}' AND bis_id='{bis_id}';
```

### Step 2: 의뢰서 생성

파일: `/mnt/z/Obsidian/browser-test/request/{frm_id}_v{version}_verify.md`

```markdown
# 테스트 의뢰: {frm_id} v{version} — {frm_nm}

## 시작점
- **URL**: ${FRAMEWEB_APP_URL}?bis_id={bis_id}
- **로그인**: ${FRAMEWEB_LOGIN_USER} / ${FRAMEWEB_LOGIN_PW}
- **메뉴 경로**: {메뉴 경로}

## 종착점 (검증 항목)

> **인라인 메시지는 위치까지 적는다**: 저장 성공·실패·검증 에러 메시지는 텍스트 + 뜨는 화면 위치(폼 상단 배너 / 우상단 토스트 / 그리드 위 / 필드 아래)를 함께 명시한다. Epic은 메시지가 모두 인라인이라 모리는 어디서 뜨는지 모른다. 예: "저장 → 폼 우상단 토스트 '저장되었습니다'", "변경 없이 저장 → 그리드 위 배너 '변경 사항이 없습니다'" (mori-no-browser-native-dialog 참고).

### 기본 렌더링
- [ ] 폼이 FormPage에서 정상 렌더링
- [ ] SplitContainer 레이아웃 (상하/좌우, 비율)
- [ ] 탭 전환 동작 (탭 있는 경우)

### R (조회)
- [ ] 폼 Open 시 데이터 표시
- [ ] Observer Chain: 마스터 선택 → 디테일 갱신
- [ ] 검색 조건 → 필터링 (검색 있는 경우)

### C (등록)
- [ ] 새 행 추가 (툴바 + 버튼)
- [ ] 필수값 입력
- [ ] 저장 → DB 반영 확인

### U (수정)
- [ ] 기존 행 수정
- [ ] 저장 → DB 반영 확인

### D (삭제)
- [ ] 행 삭제
- [ ] 확인 대화상자
- [ ] DB 반영 확인

### 입력보조 (있는 경우)
- [ ] CODE: 셀렉트 옵션 정상 로드
- [ ] LOOKUP/POPUP: 팝업 동작

## 개선점
(테스트 중 발견된 문제를 여기에 기록)

## 바인딩 구조 참고
{frmbnd 조회 결과를 표로 삽입}
```

### Step 3: 의뢰 완료 안내

```
의뢰서 생성 완료:
  /mnt/z/Obsidian/browser-test/request/{frm_id}_v{version}_verify.md

모리에게 테스트를 요청해 주세요.
결과는 /mnt/z/Obsidian/browser-test/response/ 에서 확인합니다.
```

---

## 모리 위임 — 폼 열기 테스트 / CRUD 테스트

위 통합 의뢰서 대신, 검증 범위를 둘로 나눠 의뢰할 수 있다. 모리 위임 흐름 전체는 `references/mori-protocol.md`를 참조한다(공유 폴더 경로, 결과 파일 형식, 결과 기반 판단표 포함). 통신 경로는 Epic 표준 `/mnt/z/Obsidian/browser-test/`이고, 의뢰서 안의 결과 기록 경로는 Windows 스타일 `Z:\Obsidian\browser-test\response\`로 적는다(WSL `/mnt/z`로 적으면 모리가 접근 못 한다).

### ido 확인 게이트 (의뢰 전 필수)

모리 위임은 실 브라우저 테스트다. 특히 CRUD 테스트는 실제 데이터를 쓰고 지운다. 자동 위임 금지. 다오는 ido에게 진행 방법을 묻는다:

1. **모리 실 테스트** (정석) — 지시서 생성 → 모리 위임 → 결과 회수 + DB 검증
2. **다오 우회 검증** — backend EP 직접 호출(curl + JWT) + DB SELECT로 동작만 확인. UI 검증 안 함을 보고에 명시.
3. **이미 검증됨 건너뛰기** — 동일 폼 최근 통과 이력 있으면 skip. 근거 명시 요구.
4. **수동 테스트 후 결과 paste** — ido가 직접 브라우저 확인 후 결과 전달.

ido 답변 받기 전엔 의뢰서 생성으로 넘어가지 않는다. 1번 외 선택 시 검증 한계 + 검증 안 한 항목을 보고에 명시한다.

### 폼 열기 테스트 (R 검증)

ERP 런타임에서 폼 Open 시 조회(R)가 정상 동작하는지 확인한다.

- **시작점**: `${FRAMEWEB_APP_URL}?bis_id={bis_id}` 로그인(${FRAMEWEB_LOGIN_USER}/${FRAMEWEB_LOGIN_PW}) → 메뉴 경로 진입 → hard refresh(Ctrl+Shift+R) 1회.
- **종착점**: 폼 정상 렌더링 / Open 클릭 시 전체 바인딩 GET 성공 / 그리드 데이터 표시(0건도 정상) / 체인 바인딩은 마스터 행 클릭 후 디테일 갱신 확인 / 탭 안 바인딩은 탭 클릭 후 확인(레이지 오픈 정상).
- **개선점**: 발견한 문제를 의뢰서 개선점 절에 기록.
- **결과 기록**: `Z:\Obsidian\browser-test\response\{frm_id}_open_result.md`.

### CRUD 테스트 (R/C/U/D 검증)

ERP 런타임에서 등록·수정·삭제 전체를 검증한다. 테스트 순서: Open → New → Save(Insert) → Update → Delete.

- **시작점**: 폼 열기 테스트와 동일(로그인 + 메뉴 진입 + hard refresh).
- **종착점**:
  - R: Open → 전체 바인딩 GET 성공.
  - C: New(또는 그리드 + 버튼) → 필수 필드 입력 → Save → 성공 메시지 → Open 재조회로 새 레코드 확인.
  - U: 셀 더블클릭 → 값 수정 → Enter → Save → "저장: +0 ~1 -0" 확인 → Open 재조회로 수정값 유지 확인.
  - D: 행 선택 → Delete → 성공 메시지 → Open 재조회로 레코드 제거 확인.
  - 메시지는 모두 인라인이다. 모리에게 텍스트 + 뜨는 위치(폼 상단 배너 / 우상단 토스트 / 그리드 위 / 필드 아래)를 함께 적어준다(mori-no-browser-native-dialog).
- **개선점**: 발견한 문제 기록.
- **결과 기록**: `Z:\Obsidian\browser-test\response\{frm_id}_crud_result.md`.
- **조회 전용 폼**: CUD 없으면 R만 검증하고 종료.

### 실패 시 복귀 라우팅

모리 결과를 읽고 원인별로 이전 라이프사이클 단계로 되돌린다.

| 모리 결과 | 원인 | 복귀 |
|-----------|------|------|
| Open 시 SQL 에러 / 바인딩 GET 실패 | 바인딩 체인·쿼리 결함 | `frameweb-bindchain` |
| 파라미터 에러 / bound_fcmp_ids 오류 / 0건 조회 | 바인딩 매핑 결함 | `frameweb-binding` |
| Save/Update/Delete 시 "bind parameter" 에러 | SET 매핑 누락 | `frameweb-binding` |
| "dirty 없음" / "삭제 대상 없음" | 그리드 상태·new_yn | `frameweb-bindchain` |
| 500 에러 | SQL 오류 | `frameweb-binding` |

## 후속

- 모리 PASS → 완료
- 모리 FAIL → 위 복귀 라우팅표대로 `frameweb-bindchain` 또는 `frameweb-binding`으로 수정 → 재검증
## 도구 권한

이 단계에서 호출 가능한 forge.tool ID 명단 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)

## 산출물 형식

다음 단계가 입력으로 받는 모양 (v0에서 미정의, 후속 sprint에서 채움):
- (TBD)
