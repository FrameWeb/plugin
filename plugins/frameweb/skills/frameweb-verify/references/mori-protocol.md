# 모리 위임 프로토콜

frameweb-verify의 폼 열기 테스트·CRUD 테스트에서 사용.
WSL 환경은 Chrome에 직접 접근할 수 없으므로 UI 테스트를 모리에게 위임한다.

---

## 1. 위임 흐름

```
다오 (WSL)                          모리 (Windows)
────────                          ────────────
0. ido 확인 게이트 (실 테스트 방법 4안)
   → 1번 선택 시에만 아래 계속
1. 테스트 지시서 생성
   → /mnt/z/Obsidian/browser-test/request/
2. ido에게 모리 위임 요청
                                  3. 지시서 읽기
                                  4. Chrome으로 테스트 실행
                                  5. 결과 파일 생성
                                     → Z:\Obsidian\browser-test\response\
6. 결과 파일 읽기
7. 판단 (성공→다음 / 실패→복귀)
```

### Step 0: ido 확인 게이트 (필수)

모리 위임 = 실 브라우저 테스트. 데이터 변경 동반 가능 (특히 CRUD 테스트).
다오는 자동 위임하지 않고 ido에게 진행 방법을 묻는다:

1. **모리 실 테스트** (정석) — 지시서 생성 → 모리 위임 → 결과 회수 + DB 검증
2. **다오 우회 검증** — backend EP 직접 호출(curl + JWT) + DB SELECT. UI 검증 안 함 명시.
3. **이미 검증됨 건너뛰기** — 최근 동일 폼 통과 이력 있으면 skip. 근거 명시 요구.
4. **수동 테스트 후 결과 paste** — ido가 직접 브라우저 확인 후 결과 전달

답변 받기 전엔 Step 1로 진행 금지. 1번 외 선택 시 검증 한계 및 검증 안 한 항목을 보고에 명시.

---

## 2. 공유 폴더 (Epic 표준)

```
/mnt/z/Obsidian/browser-test/
├── request/
│   ├── {폼ID}_open.md          ← 다오가 생성 (폼 열기 테스트 지시서)
│   └── {폼ID}_crud.md          ← 다오가 생성 (CRUD 테스트 지시서)
└── response/
    ├── {폼ID}_open_result.md   ← 모리가 생성 (결과)
    ├── {폼ID}_crud_result.md   ← 모리가 생성 (결과)
    └── screenshots/            ← 모리가 캡처 (선택)
```

의뢰서 안에 적는 결과 기록 경로는 **Windows 스타일**로: `Z:\Obsidian\browser-test\response\`. WSL `/mnt/z`로 적으면 모리가 접근 못 한다 (mori-windows-path).

---

## 3. 테스트 지시서 형식

### 폼 열기 테스트 지시서

```markdown
# [{폼ID}] 폼 열기 테스트 지시서

## 시작점
- ERP 접속 URL: ${FRAMEWEB_APP_URL}?bis_id={bis_id}
- 로그인: ${FRAMEWEB_LOGIN_USER} / ${FRAMEWEB_LOGIN_PW} (bis_id={bis_id})
- 메뉴 경로: {mnumst 조회 결과}
- hard refresh: Ctrl+Shift+R 1회 (vite hot reload 캐시 무효화)

## 바인딩 목록
| bnd_id | bnd_type | open_trigger | 확인 방법 |
|--------|----------|-------------|----------|
| ws_master | Grid | OPEN | Open 클릭 후 확인 |
| ws_detail | Grid | ws_master | 마스터 행 클릭 후 확인 |

## 종착점 (테스트 절차)
1. ERP URL 접속 후 로그인 → 메뉴 경로로 폼 진입 (hard refresh 후)
2. 폼 정상 렌더링 확인
3. Logs 탭 > Clear 클릭
4. Open 버튼 클릭 → 2초 대기
5. Logs에서 각 바인딩별 GET 성공/실패 확인
6. 그리드에 데이터 표시되는지 확인 (0건도 에러 아님)
7. 체인 바인딩: 마스터 행 클릭 → 디테일 GET 확인
8. 탭 안 바인딩: 탭 클릭 → GET 확인 (레이지 오픈 정상)

## 기대 결과
- 모든 바인딩 GET 성공
- 그리드에 데이터 표시 (0건도 에러 아님)
- 체인 동작 확인

## 개선점
(테스트 중 발견된 문제 기록)

## 결과 기록 → Z:\Obsidian\browser-test\response\{폼ID}_open_result.md
```

### CRUD 테스트 지시서

```markdown
# [{폼ID}] CRUD 테스트 지시서

## 시작점
(폼 열기 테스트와 동일: ERP URL + 로그인 + 메뉴 진입 + hard refresh)

## 테스트 순서

### R(조회)
1. Open 클릭 → 전체 GET 성공 확인

### C(입력)
1. New 버튼 클릭 (FieldSet이 있는 폼만)
   또는 그리드 Add(+) 버튼 (GridSet만 있는 폼)
2. 필수 필드 입력 (높은 ID 사용: TEST001, EMP999 등)
3. Save 클릭 → Logs 확인
4. Open 재조회 → 새 레코드 확인

### U(수정)
1. 셀 더블클릭 → 값 수정 → Enter
2. Save 클릭 → "저장: +0 ~1 -0" 확인
3. Open 재조회 → 수정값 유지 확인

### D(삭제)
1. 행 선택 → Delete 클릭
2. Logs 확인
3. Open 재조회 → 레코드 제거 확인

## 에러 대응
| 에러 | 다오에게 전달할 내용 |
|------|-------------------|
| "dirty 없음" | 더블클릭→입력→Enter 재시도 |
| "bind parameter 'xxx'" | 정확한 파라미터 이름 |
| "삭제 대상 없음" | bnd_id와 설정 상태 |
| 500 에러 | Response 전문 |
| 기타 | Logs 스크린샷 + 콘솔 에러 |

## 결과 기록 → Z:\Obsidian\browser-test\response\{폼ID}_crud_result.md
```

---

## 4. 결과 파일 형식

모리가 생성하는 결과 파일 표준:

```markdown
# [{폼ID}] Task N 결과

## 환경
- 테스트 일시: YYYY-MM-DD HH:MM
- 브라우저: Chrome xxx

## 결과

| 항목 | 결과 | 비고 |
|------|------|------|
| ws_master GET | ✅ 성공 (5건) | |
| ws_detail GET | ✅ 성공 (3건) | 마스터 1행 클릭 후 |
| C (입력) | ✅ 성공 | TEST001 생성 |
| U (수정) | ❌ 실패 | "bind parameter 'xxx'" |
| D (삭제) | - | U 실패로 미진행 |

## 에러 상세
### U 실패
- Logs 내용: ...
- Response: ...

## 스크린샷
(있으면 screenshots/ 폴더에 저장)
```

---

## 5. 다오의 위임 요청 문구

ido에게 전달할 메시지 템플릿:

```
모리에게 [{폼ID}] {테스트종류} 테스트 위임합니다.
지시서: /mnt/z/Obsidian/browser-test/request/{폼ID}_{open|crud}.md
(Windows: Z:\Obsidian\browser-test\request\{폼ID}_{open|crud}.md)
```

---

## 6. 결과 기반 판단 (복귀 라우팅)

복귀 단계는 frameweb-verify 라이프사이클 명칭으로 통일한다:
`frameweb-require → frameweb-canvas → frameweb-bindchain → frameweb-binding → frameweb-load → frameweb-deploy → frameweb-verify`.

### 폼 열기 테스트 결과 → 다음 액션
| 결과 | 다음 |
|------|------|
| 전부 성공 | → CRUD 테스트 진행 |
| SQL 에러 / 바인딩 GET 실패 | → `frameweb-bindchain` 복귀 (바인딩 체인·쿼리 수정) |
| 파라미터 에러 / bound_fcmp_ids 오류 | → `frameweb-binding` 복귀 (바인딩 매핑 수정) |
| 0건 조회 (세션/데이터) | → 테스트 데이터 확인 후 재의뢰 |

### CRUD 테스트 결과 → 다음 액션
| 결과 | 다음 |
|------|------|
| CRUD 전부 성공 | → 완료 |
| SET 매핑 에러 ("bind parameter") | → `frameweb-binding` 복귀 (저장 매핑 수정) |
| SQL 에러 (500) | → `frameweb-binding` 복귀 |
| "dirty 없음" / "삭제 대상 없음" / new_yn 문제 | → `frameweb-bindchain` 복귀 |
