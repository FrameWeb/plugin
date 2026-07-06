# 폼 유형 + trigger 설계 판단

bnd-setup 스킬의 참조 문서. Task 1(유형 판단), Task 2(frmbnd 설정)에서 사용.

---

## 1. 폼 유형 분류

### A. 단순 그리드 (44%)
- 구조: GridSet 1개, 탭 없음, FieldSet 없음
- 셋업 난이도: 쉬움
- 폼스크립트: 불필요
- bnd-setup: Task 1~5만 (6,7은 CUD 있을 때만)

### B. 트리+그리드 (~5%)
- 구조: TreeView(DataSet) + GridSet 1~2개
- 셋업 난이도: 중간
- 폼스크립트: expand_yn 체크 정도
- 주의: TreeView는 bnd_type=DataSet이지만 GridSet으로 분류

### C. 코드 마스터 (~10%)
- 구조: FieldSet(상세) + GridSet(목록) + GridSet(다국어)
- 셋업 난이도: 중간
- 폼스크립트: 필드잠금(Lock)
- 주의: 다국어 테이블은 U 없이 D+C 패턴 (DELETE 후 INSERT)

### D. 마스터-디테일 (~25%)
- 구조: 마스터 GridSet + 다수 탭/GridSet + FieldSet(new_yn=Y)
- 셋업 난이도: 높음
- 폼스크립트: DisableFields, CheckRequired
- 주의:
  - 하위 GridSet PK/FK는 반드시 마스터 참조 (bndsave)
  - Lazy mount로 미방문 탭의 handler 미등록은 정상
  - FieldSet의 new_yn=Y 설정 필수

### E. 종합 업무 (~16%)
- 구조: FieldSet+GridSet 혼합, FTP, 채번, 결재, 외부연동
- 셋업 난이도: 매우 높음
- 폼스크립트: 다수 (5개+)
- 주의:
  - FTP GridSet 제외
  - 채번/결재 → 별도 구현 (ido 상의)

---

## 2. trigger_yn x open_trigger 조합

워크셋의 실행 타이밍을 결정하는 두 설정. **반드시 조합으로 이해**.

- **trigger_yn**: 폼 진입 시 자동 실행 여부
- **open_trigger**: 실행 트리거 출처

### 조합별 용도

| 조합 | trigger_yn | open_trigger | 용도 |
|------|-----------|-------------|------|
| **A** | true | 'OPEN' | 폼 열면 바로 데이터 표시 (메인 Grid) |
| **B** | true | NULL | 조건 없이 전체 즉시 표시 (단일 Grid) |
| **C** | true | '{bnd_id}' | 체인 + 독립 로드 (드묾) |
| **D** | false | 'OPEN' | 조건 입력 후 Open 버튼으로 조회 |
| **E** | false | '{bnd_id}' | 마스터 행 클릭 시 연쇄 조회 |
| **F** | false | NULL | Script/버튼으로 직접 호출 |

### 설계 판단 플로우

```
1. 이 워크셋은 다른 워크셋에 종속되는가?
   ├─ YES → open_trigger = '{마스터_bnd_id}'
   │   └─ 폼 진입 시에도 독립적으로 보여야 하나?
   │       ├─ YES → trigger_yn = true  (C) ← 드묾
   │       └─ NO  → trigger_yn = false (E) ← 일반적
   │
   └─ NO → 독립 워크셋
       │
       2. 폼 열자마자 자동 실행해야 하나?
          ├─ YES → trigger_yn = true
          │   └─ 다른 OPEN 워크셋과 함께 체인 실행?
          │       ├─ YES → open_trigger = 'OPEN' (A)
          │       └─ NO  → open_trigger = NULL   (B)
          │
          └─ NO → trigger_yn = false
              └─ Open 버튼으로 실행?
                  ├─ YES → open_trigger = 'OPEN'  (D)
                  └─ NO  → open_trigger = NULL     (F)
```

### ★ 기본값 정책
- **trigger_yn = false** (ido 명시 요청 없으면 false)
- 마스터-디테일: 마스터=D(false/OPEN), 디테일=E(false/{마스터})
- 단일 그리드: B(true/NULL) 또는 D(false/OPEN)

### 실무 패턴 3가지

**패턴 1: 마스터-디테일 (가장 흔함)**
- 마스터 → D (trigger_yn=false, open_trigger='OPEN')
- 디테일 → E (trigger_yn=false, open_trigger='{마스터_bnd_id}')
- 유틸리티 → F (trigger_yn=false, open_trigger=NULL)

**패턴 2: 단일 Grid 관리 화면**
- 메인 → B (trigger_yn=true, open_trigger=NULL) — 폼 열면 전체 즉시

**패턴 3: 조건 검색 폼**
- 검색결과 → D (trigger_yn=false, open_trigger='OPEN') — Open 버튼 필요

---

## 3. Observer Chain (마스터-디테일)

### chain_events 필수 전제조건

그리드 컴포넌트의 CMPMST.chain_events가 없으면 Observer Chain 미동작:
```sql
-- CMPMST에 chain_events 확인
SELECT cmp_id, chain_events FROM cmpmst
WHERE cmp_id IN ('EPIC_HANDSONTABLE', 'EMAX_HANDSONTABLE');
```

경로: 행 선택 → `hasChainEvents` 체크 → `onChange` → `handleFieldChange` → `executeChainFrom` → 자식 R 실행.

### open_sq 규칙

| 역할 | open_sq |
|------|---------|
| 마스터 (루트) | 1 |
| 디테일 (1차) | 2 |
| 서브디테일 (2차) | 3 |
| ... | 순차 증가 |

★ open_sq = 0이면 Observer Chain에서 필터링됨 (미동작)

### save_sq 규칙

| 역할 | save_sq |
|------|---------|
| 마스터 | 1 |
| 디테일 | 2, 3... |
| 조회 전용 | 0 (Save 스킵) |

---

## 4. Lazy Open (탭 내부 워크셋)

탭 안에 있는 워크셋은 해당 탭을 방문하기 전까지 로드되지 않는다 (정상 동작).

### Open 테스트에서 판단

| 상황 | 판단 | 대응 |
|------|------|------|
| Open 후 로그에 워크셋 없음 | 레이지 오픈 (정상) | 탭 클릭 후 GET 확인 |
| 탭 클릭 후 GET 성공 | 정상 | 통과 |
| 탭 클릭 후 GET 실패 | SQL/파라미터 에러 | Task 3/4로 복귀 |

### 레이지 vs 체인 미발동 구분
- **레이지 오픈**: open_trigger 설정됨 + 탭 안. 탭 클릭하면 실행됨.
- **체인 미발동**: open_trigger 미설정(NULL). 탭 클릭해도 실행 안 됨 → Task 2에서 설정 필요.

---

## 5. new_yn / FieldSet 규칙

| bnd_type | new_yn | 이유 |
|----------|--------|------|
| FieldSet (CUD) | 'Y' | 폼 상단 NEW 버튼 대상 |
| GridSet | 'N' | 그리드 자체 Add(+) 버튼 사용 |
| DataSet | 'N' | 보조 데이터 |

- **폼당 1개**의 바인딩셋만 new_yn='Y' 가능
- GridSet 행 추가 = 그리드 툴바 Add(+) 버튼 (new_yn 무관)
