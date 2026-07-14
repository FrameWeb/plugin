# frameweb-migration 플러그인

FRAME9(EMAX 레거시, MSSQL) 폼 자산을 FrameWeb으로 이관하는 **12단계 마이그레이션 마법사**의 단계별 스킬 모음.

대상 코드: FrameWeb 저장소 `backend/migration/frame9/` + `frontend/src/migration/frame9/MigrationPage.tsx` (API prefix `/api/migration9`).

## 운영 방식 (전제)

1. **UnSkein PLANNER가 분석** — 이관 대상 폼을 스콥으로 내리고, 단계를 UnSkein 카드(WBS)로 정의한다
2. **UnSkein을 통해 작업 개시** — 실행기가 카드를 선점해 한 단계씩 실행한다
3. **한 단계 = 한 카드 = 데이터 확인까지** — 각 단계의 list API로 건수·경고를 확인해야 카드 완료
4. **스킬은 살아있는 문서** — 진행 중 발견한 함정·수정 사항은 해당 단계 스킬(SKILL.md)에 반영하고 이 플러그인으로 배포한다

단계→카드 매핑 규약은 `skills/migration-frame9/SKILL.md`(전체 지도)에 있다.

## 담긴 스킬

| 스킬 | 단계 | 산출물 |
|------|------|--------|
| `migration-frame9` | 전체 지도 | 단계 순서·의존 관계·방언 결정·UnSkein 진행 절차 |
| `migration-step00-mapping` | Step 0 | migmap (레거시 타입→컴포넌트 사전) |
| `migration-step01-bisfrm` | Step 1 | BISFRM + MIGCMP (.Designer.vb 임포트) |
| `migration-step02-frmcmp` | Step 2 | FRMCMP (컴포넌트 트리) |
| `migration-step03-frmwrk` | Step 3 | FRMWRK (워크셋 정의) |
| `migration-step04-wrksql` | Step 4 | WRKSQL (쿼리 변환 — PG/MSSQL 방언 분기) |
| `migration-step05-wrkget` | Step 5 | WRKGET (조회 파라미터) |
| `migration-step06-wrkcmp` | Step 6 | WRKCMP (컬럼-필드 매핑) |
| `migration-step07-bispop` | Step 7 | BISPOP 계열 (입력보조) |
| `migration-step08-biztbl` | Step 8 | 비즈 테이블 구조+데이터 (PG/MSSQL 방언 분기) |
| `migration-step09-setref` | Step 9 | WRKSET + WRKREF (저장 바인딩·참조 관계) |
| `migration-step10-coding` | Step 10 | BISFRM.vbcode (원본 VB 로직 보관) |
| `migration-step11-rename` | Step 11 | migrename (리네임 사전) |

스킬 이름의 번호가 실행 순서다. 설치 후에는 `frameweb-migration:migration-step04-wrksql` 형태로 호출된다.

## 설치

```shell
# 마켓플레이스가 이미 등록돼 있으면 플러그인만 설치
claude plugin install frameweb-migration@frameweb-marketplace
```
