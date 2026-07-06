---
name: frameweb-multilanguage
description: >
  폼 화면의 라벨·컬럼 헤더·메뉴·본문이 여러 저장소에 흩어져 있어, 다국어화하려면 위치마다
  추출·번역·등록·재배포 절차가 다른 문제를 한 곳에서 다룬다.
  저장 위치 카탈로그(5위치) + 표준 절차(추출→번역→등록→반영→검증) + 라벨 변경 시 다국어 재생성 연쇄 규칙을 제공한다.
  폼 디자이너 탭 스킬 6종과 별개인 폼 라이프사이클 보조 스킬이다.
  Trigger: 폼 다국어, 다국어 정리, 라벨 다국어, 메뉴 번역, langtxt, form_label,
  컬럼 헤더 번역, 코드 노출 라벨, 영어 라오어 번역, i18n 폼, 다국어 번역 관리, frameweb-multilanguage.
user-invocable: true
---

# Form Multilanguage — 폼 다국어 추출·번역·등록·재배포

폼 화면의 라벨·제목·메뉴·본문은 다섯 저장 위치에 나뉘어 있다. 위치마다 저장 DB가 다르고(시스템 DB / 비즈 DB), 다국어를 거는 방식이 다르며, 일부는 값을 바꾸면 재배포가 필요하고 일부는 새로고침으로 반영된다. 이 스킬은 그 위치 카탈로그와 표준 절차, 그리고 라벨을 바꿀 때 다국어를 다시 떠야 하는 연쇄 규칙을 정리한다.

## 폼 라이프사이클 보조 스킬에서의 위치

이 스킬은 폼 디자이너 탭 스킬 6종(`frameweb-canvas` / `frameweb-bindchain` / `frameweb-binding` / `frameweb-script` / `frameweb-sandbox` / `frameweb-state`)과는 별개인 **폼 라이프사이클 보조 스킬**이다. 탭 스킬이 폼을 만들고 바인딩을 잇는 단계라면, 이 스킬은 만들어진 폼을 다국어로 운영하는 단계를 담당한다.

`frameweb-canvas`의 "라벨 빈 컴포넌트 점검" 절과 연계한다. frameweb-canvas는 비어 있는 라벨을 채워 코드 노출을 막는 데까지 책임지고, 채운 라벨을 EN/LA로 번역해 등록하는 단계가 이 스킬이다. frameweb-canvas 본문도 "폼 라벨 다국어는 별도 단계"라고 이 스킬로 넘긴다.

---

## 1. 폼 다국어 저장 위치 카탈로그

폼 화면 텍스트는 다섯 위치에 저장된다. 위치마다 DB·키 형식·렌더 경로·다국어 방식이 다르다.

| # | 무엇 | 저장 테이블 · DB | 키 형식 | 화면 렌더 경로 | 다국어 방식 |
|---|------|-----------------|---------|----------------|-------------|
| 1 | 컴포넌트·패널 라벨 | `frmcmp.label` · 시스템 DB `${FRAMEWEB_DB_NAME}` | (라벨은 컬럼 직접 값) | DynamicFormRenderer `t('{frm_id}.{fcmp_id}', {defaultValue: label})` | langtxt 키(3번)로. KO 라벨이 defaultValue 폴백 |
| 2 | 그리드 컬럼 헤더 | `bndcmp.label` · 시스템 DB `${FRAMEWEB_DB_NAME}` | (헤더는 컬럼 직접 값) | 그리드 렌더러 (label 비면 필드명 노출) | langtxt 키 `{frm_id}.{fcmp_id}`(3번)로 |
| 3 | 폼 라벨 다국어 | `langtxt` · **비즈 DB(bis_id별, 예: bnk)** | `{frm_id}.{fcmp_id}`, 언어 KO/SRC/EN/LA | 런타임 `GET /api/core/translations/bundle`(시스템 DB 기본 + X-Bis-Id 비즈 DB merge, 비즈 우선) → i18next `t()`가 EN/LA 적용 | langtxt KO/SRC/EN/LA 행. **재배포 불필요**(새로고침 반영) |
| 4 | 메뉴 번역 | `langtxt` · **비즈 DB(bis_id별, 예: bnk)** | `menu.{menu_cd}`, 언어 KO/SRC/EN/LA(SRC=원문) | I18N100 "다국어 번역 관리" Menu 탭(key_prefix `menu.%`, 컴포넌트 inp_prefix_menu) | langtxt EN/LA 행. KO/SRC 보존 |
| 5 | 폼 데이터 본문 다국어 | 비즈 DB 테이블의 언어별 컬럼 (예: `bnk_sms_template.tpl_body_ko/_en/_zh/_lo`) | (레코드 1행에 언어 N컬럼) | 발송·표시 시 언어 선택으로 해당 컬럼 읽음 | 언어별 컬럼 분리. 코드 라벨 다국어와 별개 |

### 핵심 구분 — 키 1개에 언어 N행 vs 레코드 1행에 언어 N컬럼

- 1~4번은 **코드 라벨 다국어** — UI 텍스트를 키 단위로 언어별 번역. langtxt 키-번역 체계.
- 5번은 **데이터 레코드 본문 다국어** — 사용자가 입력한 콘텐츠 자체를 언어별 컬럼으로 저장. 번역 카탈로그 대상이 아니다.

### langtxt 정본은 비즈 DB — 런타임 bundle은 비즈 우선 merge

폼 라벨(위치 3)과 메뉴(위치 4)의 다국어 정본은 둘 다 **비즈 DB(bis_id별, 예: bnk)의 langtxt**다. 폼 라벨 키는 `{frm_id}.{fcmp_id}`, 메뉴 키는 `menu.{menu_cd}`.

런타임 번역 로드 EP `GET /api/core/translations/bundle`(`core_admin_router.py` L670~)은 **시스템 DB langtxt를 기본으로 읽고, `X-Bis-Id` 헤더가 있으면 그 비즈 DB langtxt를 merge하되 비즈 값을 우선**한다. 같은 키가 시스템·비즈 양쪽에 있으면 화면에는 비즈 값이 나온다.

따라서 폼 라벨 다국어는 **비즈 DB langtxt에 넣어야 화면에 반영된다.** 시스템 DB에만 넣으면, 비즈 DB에 같은 키가 (extract로) 이미 있을 때 비즈의 미완성 값(EN 빈/코드)에 가려진다.

### I18N100 다국어 번역 관리 화면 — 3탭

I18N100 화면은 Form / Menu / Code 3탭이며, 각 탭이 다른 소스를 본다.

| 탭 | 소스 | 대상 |
|----|------|------|
| Form | langtxt `{frm_id}.%`(비즈 DB) | 폼 라벨(위치 3) |
| Menu | langtxt `menu.%`(비즈 DB) | 메뉴 번역(위치 4) |
| Code | 코드 번역 | (메커니즘 상세 미파악 — "후속 보강" 참조) |

---

## 2. 코드 노출 판정 (라벨 빈 컴포넌트)

라벨이 비면 화면에 컴포넌트 코드(fcmp_id)가 노출되는 경우와, 비어도 안 보이는 경우를 가른다. 노출 여부는 `exposes_code` 기준으로 판정한다.

| 판정 | 컴포넌트 | 근거 |
|------|----------|------|
| 코드 노출됨 | 제목 표시형 — cmp_id에 `PANEL` / `CONTAINER` / `TABS` / `HANDSONTABLE` / `BUTTON` 포함 | label 비면 렌더러가 fcmp_id로 폴백 (`transformer.ts` `label\|\|fcmp_id`, `rectText.tsx`, `bnk_split_container.tsx`) |
| 안 보임 | 순수 레이아웃 컨테이너 — 헤더 미표시 (예: `SPLIT_CONTAINER`) | 헤더 자리가 없어 label 비어도 화면에 안 나옴 |
| 제외 | 입력 필드 | 입력칸만 표시, 코드 노출 대상 아님 |

cmp_id 패턴만으로 단정하지 말고, 헤더를 실제로 표시하는지 렌더러로 확인해 `exposes_code`를 가른다.

---

## 3. 표준 절차 (extract → 추출 확인 → 번역 → 등록 → 반영 → 검증)

신규 컴포넌트·그리드 컬럼·메뉴·코드는 추가해도 langtxt에 자동으로 안 들어간다. 그래서 다국어 작업의 첫 단계는 항상 **extract(키 가져오기)** 다. extract 전에는 번역할 소스 자체가 없다.

### 3.0 신규 소스 확보 (extract)

`POST /api/erp/i18n/extract`(= I18N100 화면 "키 가져오기" 버튼). Bearer 토큰 + `X-Bis-Id` 헤더로 호출한다. 비즈 DB langtxt에 키+원문(SRC/KO 두 행)을 등록한다. 이미 있는 키는 DO NOTHING, EN/LA는 만들지 않는다.

| target_type | 추가 인자 | 소스 | 비즈 DB langtxt 등록 키 |
|-------------|-----------|------|--------------------------|
| `form` | `target_id=frm_id` | 시스템 DB frmcmp + bndcmp 라벨 | `{frm_id}.{fcmp_id}` SRC/KO |
| `menu` | - | 비즈 DB mnumst.title | `menu.{menu_cd}` SRC/KO |
| `code` | - | 비즈 DB popcod | 코드 키 SRC/KO |

- 응답: `{status:'ok', extracted(신규 KO 등록수), total}`.
- 라벨·컬럼을 한국어로 고쳤다면(코드 노출 수정 후) 그 SRC/KO 원문이 갱신되도록 extract를 다시 부른다. SRC는 `ON CONFLICT UPDATE`, KO는 최초 1회만 시드한다.

### 3.1 추출 확인

비즈 DB langtxt에서 대상 키의 KO 값과 EN/LA 빈 여부를 점검한다. **값 기준** — 빈 문자열도 누락으로 본다.

### 3.2 번역

- EN/LA가 빈 키만 번역한다. 고유 라벨만 추려 중복 제거로 번역량을 줄인다.
- 상태 접두 대괄호(`[준비]` / `[보고]`) 형식을 유지한다.
- 라오어(LA)는 정확도 한계가 있어 운영자 검토를 권장한다.

### 3.3 등록

| 등록 대상 | 위치 | 키 / 컬럼 | 재배포 |
|-----------|------|-----------|--------|
| 라벨 자체(코드 노출 해소) | 시스템 DB | `frmcmp.label` / `bndcmp.label` UPDATE | **필요** |
| 폼 라벨 다국어 | 비즈 DB langtxt | `{frm_id}.{fcmp_id}` EN/LA (KO/SRC 보존) | 불필요 |
| 메뉴 번역 | 비즈 DB langtxt | `menu.{menu_cd}` EN/LA (KO/SRC 보존) | 불필요 |

- 비즈 DB langtxt에 EN/LA를 `ON CONFLICT (txt_key, lang_cd) DO UPDATE`로 UPSERT한다. KO/SRC는 보존한다.
- `txt_category`는 extract가 NULL로 두므로 그에 맞춘다. `form_label` 같은 임의 카테고리를 시스템 DB에 따로 넣지 않는다(비즈 우선 merge로 가려지거나 정본이 둘로 갈린다).
- 라벨 자체 변경 후 재배포: `POST /api/forms/deploy/`(`X-Bis-Id` 헤더 + Bearer 토큰).

### 3.4 반영

| 변경 종류 | 반영 방식 |
|-----------|-----------|
| langtxt 다국어 | 재배포 불필요. 런타임 i18next가 bundle로 로드, 새로고침 반영 |
| 라벨·컬럼 텍스트 자체 (frmcmp / bndcmp.label) | 폼 재배포 필요(별개). ERP 스냅샷(비즈 DB `frmmst`) 갱신 |

### 3.5 검증

- **비즈 DB 기준**으로 KO는 있는데 EN/LA가 빈 키 0건을 확인한다. 빈 문자열도 누락으로 본다. 시스템 DB가 아니라 비즈 DB를 조회한다.
- 화면에서 언어를 전환(영어/라오어)했을 때 라벨·컬럼 헤더·메뉴가 번역되는지 확인한다.

---

## 4. 연쇄 규칙 (변경 시 신규 다국어 조달)

라벨이나 내용을 바꾸면 그 위치의 다국어도 다시 떠야 한다.

```
라벨/컬럼 변경
  → extract 재호출 (비즈 DB langtxt의 SRC/KO 원문 갱신)
  → 그 키의 비즈 DB langtxt EN/LA 재번역·재등록 (값이 바뀌었으므로)
  → 라벨·컬럼 텍스트 자체를 바꿨으면 폼 재배포
```

예: 컬럼 헤더 코드를 한국어로 바꾸면(`bndcmp.label`), extract를 다시 불러 SRC/KO를 갱신하고 그 키의 langtxt EN/LA를 새로 번역·등록해야 화면 언어 전환이 맞는다. 라벨만 바꾸고 다국어를 안 새로 뜨면, 언어 전환 시 옛 번역이나 KO 폴백이 그대로 나온다.

---

## 5. 자주 만나는 함정

- **폼 라벨 다국어 정본은 비즈 DB다.** 시스템 DB langtxt에 넣으면, 비즈 DB에 같은 키가 있을 때 런타임 bundle의 비즈 우선 merge로 가려진다. 폼 라벨·메뉴 모두 비즈 DB langtxt에 넣는다.
- **신규 컴포넌트·라벨은 extract 전에는 langtxt에 없다.** 번역 작업 전 반드시 extract를 먼저 부른다. extract 없이 비즈 DB만 조회하면 "키 없음"으로 잘못 진단한다.
- **라벨을 한국어로 고친 뒤(코드 노출 수정)에는 extract를 다시 불러야** SRC/KO 원문이 갱신된다. 안 부르면 옛 원문이 남아 번역 대조가 어긋난다.
- **메뉴·폼 라벨 번역을 시스템 DB langtxt에서 찾으면 0건일 수 있다.** 정본은 비즈 DB langtxt다. 시스템 DB만 보고 "누락 0"으로 진단하면 틀린다. 반드시 비즈 DB를 조회한다.
- **번역 누락 진단은 "lang_cd 행 존재"가 아니라 "txt_value 값 존재"로 한다.** 행은 있고 값이 빈 경우가 있다.
- **langtxt는 전역(시스템 DB)과 비즈별(비즈 DB) 둘 다 존재한다.** 폼 라벨·메뉴 정본은 비즈 DB. 시스템 DB는 런타임 bundle의 기본값일 뿐 같은 키는 비즈 값이 우선한다.
- **로컬↔프로덕션은 양쪽 DB를 맞춘다.** 시스템 DB(`${FRAMEWEB_DB_NAME}` ↔ `${FRAMEWEB_DB_NAME}`)와 비즈 DB(`bnk` ↔ 프로덕션 `bnk`) 둘 다 반영한다.

---

## 6. 작업 흐름 (전형)

```
대상 폼/비즈 확인
  → extract (POST /api/erp/i18n/extract — 비즈 DB langtxt에 SRC/KO 시드)
  → 비즈 DB langtxt에서 KO 있고 EN/LA 빈 키 추출 확인
  → 고유 라벨 번역 (KO 기준 EN/LA)
  → 비즈 DB langtxt에 EN/LA UPSERT (KO/SRC 보존)
  → langtxt는 재배포 불필요(새로고침) / 라벨 텍스트 자체 변경이면 폼 재배포
  → 로컬 검증 (비즈 DB 기준 빈 키 0 + 언어 전환)
  → 프로덕션 반영
  → 프로덕션 검증
```

---

## 후속 보강

아래는 본 스킬 작성 시점(2026-06-16)에 코드/DB로 확정하지 못한 항목이다. 확인 후 채운다.

- **I18N100 Code 탭 상세 메커니즘** — 코드 번역 탭이 보는 소스 테이블·키 형식·등록 경로 미파악.
- **본문 다국어 폼 메타 자동 생성** — 위치 5번(언어별 컬럼)의 폼 메타(panel 자식 입력 컴포넌트 N건 + bndcmp 매핑 + bndquery C/R/U 언어 컬럼 처리)를 수기 작성이 아니라 자동 생성하는 절차 미확립.

## 관련 문서

- `docs/architecture/domains/i18n.md` — LANGMST/LANGTXT, LanguageManager 3탭, 데이터 본문 다국어 vs 코드 라벨 다국어 구분
- `docs/architecture/domains/forms.md` "다국어 본문 컬럼 패턴" 절 — 위치 5번 폼 메타 구조
- `~/.claude/skills/frameweb-canvas/SKILL.md` "라벨 빈 컴포넌트 점검" 절 — 코드 노출 라벨 채우기(이 스킬 진입 직전 단계)

<!-- writer-check: 위 원칙 전원 준수. -->
