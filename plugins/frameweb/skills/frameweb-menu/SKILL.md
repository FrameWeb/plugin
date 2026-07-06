---
name: frameweb-menu
description: Epic Cloud 개발자(Dev) 플랫폼 사이드바 메뉴(PLTMNU)에 항목을 추가하고 어드민/역할 권한을 부여한다. 새 어드민 페이지나 도구를 좌측 사이드바에 노출시킬 때 사용. 항상 "어디에 둘지(그룹/위치) + 어떤 권한에 줄지"를 먼저 ido에게 묻고 진행한다. PLTMNU 행 INSERT + MNUPMS 권한 시드 + LANGTXT 메뉴명 다국어(KO/EN/LA) 등록 + 프론트 라우트 매핑(FORGE_ROUTE_MAP) + 아이콘 등록 + 마이그레이션 적용 + 검증을 한 번에 처리한다. 트리거 — frameweb-menu, 메뉴 추가, 사이드바에 추가, Dev 메뉴 추가, 플랫폼 메뉴, PLTMNU, 메뉴 노출, 메뉴 다국어, i18n 언어 추가, "X 페이지를 사이드바에 넣어줘", "어드민 메뉴 추가". 비즈니스(ERP) 메뉴(MNUMST/비즈 DB)는 본 스킬 범위 밖.
---

# frameweb-menu

> 개발자 플랫폼 사이드바(PLTMNU)에 메뉴 한 건을 추가하고 권한을 부여하는 운영 스킬.

## 1. 본질 — 개발자가 dev에서 보는 좌측 사이드바 메뉴

`pltmnu`는 개발자(dev.mupai.studio / ${FRAMEWEB_APP_URL})가 보는 **좌측 사이드바 메뉴 트리**다. 시스템 DB(`${FRAMEWEB_DB_NAME}`)의 단일 트리이며 모든 개발자가 공유한다. 그룹(`parent_id IS NULL`) 아래 메뉴 항목이 달리는 자기참조 구조.

**비즈니스(ERP) 메뉴와 혼동 금지.** ERP 사용자가 보는 메뉴는 `mnumst`(비즈니스 DB, 폼 연결 + 역할별 권한)로 완전히 다른 시스템이다. 본 스킬은 PLTMNU(개발자 사이드바)만 다룬다.

## 2. 항상 먼저 묻는다 — 위치 + 권한 + 메뉴명 라벨

이 스킬의 핵심은 **추측하지 않고 세 가지를 먼저 확인**하는 것이다. ido가 페이지 경로만 줘도 아래 셋은 반드시 합의한다.

1. **어디에 둘지** — 어느 그룹(`parent_id`)의 어느 위치(맨 위/맨 아래/특정 항목 다음)인가.
   - 그룹 예: `grp_platform_admin`(플랫폼 관리) / `grp_development`(개발) / `grp_organization`(공통) / `grp_status`(현황) / `grp_test`(테스트) / `grp_forge`(Forge).
   - "맨 아래" = 그 그룹 자식 중 가장 큰 `sort_order` + 1.
2. **어떤 권한에 줄지** — 어느 `min_level`(접근 레벨)인가. 이 값이 `mnupms` 권한 행을 시드한다.
   - 어드민 권한 = `min_level=100` → 역할 `platform_admin`.
3. **메뉴명을 각 언어로 어떻게 표시할지** — 사이드바 메뉴는 `mnu_nm`에 담긴 i18n 키(`core.sidebar.{key}`)를 `langtxt` 번역으로 화면에 그린다. 활성 언어 **KO / EN / LA** 세 가지 라벨을 등록해야 모든 언어 화면에서 제대로 보인다(§4.2 langtxt 단계).
   - **KO**(한국어) = `mnu_nm_fallback`과 같은 라벨. 필수.
   - **EN**(영어, 기본 언어) = ido가 함께 주는 것이 가장 정확하다. 안 주면 다오가 번역안을 제시해 합의한다.
   - **LA**(라오어) = 다오가 번역안을 제시하거나, 우선 EN 라벨로 채우고 이후 LanguageManager에서 보정한다.

이미 셋 다 받았으면(예: "플랫폼 관리 맨 아래, 어드민 권한, 메뉴명 KO '포지AI관리' / EN 'Forge AI'") 다시 묻지 않고 진행한다.

## 3. 두 가지 메뉴 연결 방식

PLTMNU에는 **경로(route) 컬럼이 없다.** 메뉴 클릭 시 동작은 프론트가 `mnu_id`로 결정한다(`DashboardPage.tsx`).

- **방식 A — 별도 페이지로 이동** (대상 라우트가 이미 독립 페이지로 존재할 때, 권장):
  - 프론트 `FORGE_ROUTE_MAP`에 `'{mnu_id}': '{경로}'` 한 줄 추가 → 클릭 시 `navigate('{경로}')`.
- **방식 B — 대시보드 안 내용 패널** (별도 라우트 없이 대시보드 안에서 내용 전환):
  - `DashboardPage.tsx` 렌더 분기에 `{activeMenu === '{mnu_id}' && <콘텐츠 />}` 추가 + 임베드 컴포넌트 필요.
  - 코드가 더 들어가므로, 독립 라우트가 있으면 방식 A를 쓴다.

## 4. 추가 절차 (4단계)

### 4.1 현재 DB 상태 확인 (추측 금지)
시스템 DB `${FRAMEWEB_DB_NAME}`에서 대상 그룹의 자식과 정렬값, 중복 여부를 실조회한다.
```bash
export PGPASSWORD=${FRAMEWEB_DB_PASSWORD}
# 대상 그룹 자식 + sort_order
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c \
 "SELECT mnu_id, mnu_nm_fallback, sort_order, is_visible FROM pltmnu WHERE parent_id='grp_platform_admin' ORDER BY sort_order;"
# mnu_id 중복 확인
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c \
 "SELECT mnu_id FROM pltmnu WHERE mnu_id='{새 mnu_id}';"
```
- `mnu_id`는 케밥 케이스, 라우트와 맞추면 명확하다(예: 경로 `/page-context-manager` → `mnu_id='page-context-manager'`).
- 맨 아래 위치 = 조회된 최대 `sort_order` + 1.

### 4.2 마이그레이션 작성
`sync_pltmnu_missing_menus.py`를 표준 모델로 답습한다. 새 파일 `backend/migrations/versions/pltmnu_{topic}_001.py`, `down_revision`은 현재 alembic head(`psql ... "SELECT version_num FROM alembic_version;"`로 확인).
- **pltmnu INSERT**: `(mnu_id, mnu_nm, mnu_nm_fallback, parent_id, icon, min_level, sort_order, is_visible, is_system)` + `ON CONFLICT (mnu_id) DO NOTHING`.
  - `mnu_nm` = i18n 키(`core.sidebar.{key}`), `mnu_nm_fallback` = 화면에 보일 한국어 라벨.
- **mnupms INSERT**: `min_level → 역할` 매핑(`LEVEL_ROLE_MAP`)대로 `(mnu_id, target_type, target_value, is_allowed=true)` + `ON CONFLICT (mnu_id, target_type, target_value) DO NOTHING`.
- **langtxt INSERT (메뉴명 다국어)**: `mnu_nm`의 i18n 키(`core.sidebar.{key}`)에 대해 활성 언어 3행(KO/EN/LA)을 **같은 마이그레이션 안에서** 시드한다. 이게 빠지면 EN/LA 화면에서 메뉴명이 한국어 fallback으로만 나온다.
  ```sql
  INSERT INTO langtxt (txt_key, txt_category, lang_cd, txt_value, page_id, c_user, c_dt, m_user, m_dt)
  VALUES
    ('core.sidebar.{key}', 'MENU', 'KO', '{한국어 라벨}', 'common:system.sidebar', 'frameweb-menu', NOW(), 'frameweb-menu', NOW()),
    ('core.sidebar.{key}', 'MENU', 'EN', '{영어 라벨}',   'common:system.sidebar', 'frameweb-menu', NOW(), 'frameweb-menu', NOW()),
    ('core.sidebar.{key}', 'MENU', 'LA', '{라오어 라벨}', 'common:system.sidebar', 'frameweb-menu', NOW(), 'frameweb-menu', NOW())
  ON CONFLICT (txt_key, lang_cd) DO NOTHING;
  ```
  - `txt_id`는 시퀀스(`langtxt_txt_id_seq`) 자동 생성이라 넣지 않는다. UNIQUE 제약은 `(txt_key, lang_cd)`.
  - `txt_category='MENU'`, `page_id='common:system.sidebar'`는 기존 사이드바 키의 표준값이다. `c_user='frameweb-menu'`로 출처를 남긴다.
  - 활성 언어는 발행 전 실조회로 확인한다: `SELECT lang_cd FROM langmst WHERE use_yn='t' ORDER BY sort_order;` (현재 KO/EN/LA 3개). 언어가 늘면 행도 함께 늘린다.

`LEVEL_ROLE_MAP` (min_level → 허용 role / org_role):
| min_level | role | org_role |
|-----------|------|----------|
| 0, 10, 30 | platform_admin, project_manager, developer | owner, pm, developer(, viewer) |
| 50 | platform_admin, project_manager | owner, pm |
| 100 | platform_admin | (없음) |

### 4.3 프론트 연결 (방식 A 기준)
- `frontend/src/pages/DashboardPage.tsx`의 `FORGE_ROUTE_MAP`에 `'{mnu_id}': '{경로}',` 한 줄 추가(주석에 "새 메뉴 라우트 추가 시 갱신 의무" 명시되어 있음).
- 아이콘은 `frontend/src/utils/iconMap.ts`에 등록된 lucide 이름만 제대로 그려진다. 미등록 이름은 `Box`로 폴백되므로, 원하는 아이콘이 없으면 import 목록과 `ICON_MAP` 둘 다에 추가한다.

### 4.4 적용 + 검증
```bash
cd /home/ido/epic/epic-cloud/backend && . venv/bin/activate && alembic upgrade head
# 적용 확인
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "SELECT * FROM pltmnu WHERE mnu_id='{mnu_id}';"
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "SELECT * FROM mnupms WHERE mnu_id='{mnu_id}';"
# 메뉴명 다국어 — KO/EN/LA 3행이 채워졌는지 확인
psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "SELECT lang_cd, txt_value FROM langtxt WHERE txt_key='core.sidebar.{key}' ORDER BY lang_cd;"
```
- 사이드바 메뉴는 새 세션/새로고침 시 `menuPermissionApi.getMyMenus()`로 다시 로드된다. 화면에서 메뉴가 보이고 클릭 시 대상 페이지로 이동하는지는 **모리에게 UI 확인 위임**(`admin-test-request`).

## 5. 핵심 사실 (검증된 메커니즘, 2026-06-06)

| 항목 | 값 |
|------|-----|
| 메뉴 테이블 | `pltmnu` (시스템 DB `${FRAMEWEB_DB_NAME}`, public) — `mnu_id` PK, 자기참조 트리 |
| 권한 테이블 | `mnupms` (시스템 DB) — `(mnu_id, target_type, target_value)` UK, `is_allowed` bool. **비즈 DB의 mnupms와 다른 테이블** |
| 조회 EP | `GET /api/platform/menus` / `GET /api/platform/menus/visible` |
| 권한 계산 EP | `menuPermissionApi.getMyMenus()` → 사용자 역할 × mnupms |
| 수정 EP | `PUT /api/platform/menus/{mnu_id}` (순서/숨김/레벨만). **생성 POST 없음 → 마이그레이션 전용** |
| 라우트 연결 | pltmnu에 route 컬럼 없음 → `DashboardPage.tsx`의 `FORGE_ROUTE_MAP[mnu_id]` |
| 메뉴명 번역 | `langtxt` (시스템 DB) — `(txt_key, lang_cd)` UK, `txt_id` 시퀀스 자동. `mnu_nm`의 키 `core.sidebar.{key}`에 KO/EN/LA 3행. 표준 `txt_category='MENU'` / `page_id='common:system.sidebar'`. 누락 시 비-KO 화면에서 fallback만 노출 |
| 어드민 안전장치 | `platform_admin` 역할 사용자는 mnupms와 무관하게 모든 `is_visible` 메뉴 자동 노출 |

## 6. Gotcha

- **route 컬럼 없음** — 별도 페이지 이동은 `FORGE_ROUTE_MAP` 갱신이 필수. 빠뜨리면 클릭 시 빈 내용(존재하지 않는 `activeMenu` 패널)만 뜬다.
- **생성 API 없음** — pltmnu는 마이그레이션 INSERT로만 추가한다. admin API는 수정(순서/숨김/레벨)만 가능.
- **아이콘 미등록 시 Box 폴백** — `iconMap.ts`에 없는 lucide 이름은 조용히 일반 박스로 나온다. 원하는 아이콘은 두 곳(import + ICON_MAP)에 등록.
- **min_level은 권한의 시드값** — 런타임 접근 제어는 `mnupms`가 한다. min_level만 넣고 mnupms 행을 빠뜨리면, platform_admin 외 역할에는 메뉴가 안 보인다.
- **시스템 DB mnupms ≠ 비즈 DB mnupms** — 이름은 같지만 컬럼과 DB가 다른 별개 테이블이다. 비즈(ERP) 메뉴 권한은 본 스킬 밖.
- **메뉴명 i18n 키만 넣고 langtxt 누락** — `mnu_nm`은 i18n 키(`core.sidebar.{key}`)다. 키에 대한 `langtxt` 번역(KO/EN/LA)을 안 채우면 EN/LA 화면에서 한국어 `mnu_nm_fallback`만 나오거나 키가 그대로 노출된다. 메뉴 추가 시 langtxt 3행을 같은 마이그레이션에 반드시 함께 시드한다(§4.2).

## 7. 첫 적용 사례 (2026-06-06)

`/page-context-manager`(포지AI관리)를 사이드바 맨 아래(Forge 그룹 끝)·어드민 권한으로 추가:
- pltmnu: `('page-context-manager', 'core.sidebar.page_context_manager', '포지AI관리', 'grp_forge', 'Bot', 100, 6, true, true)`
- 참고: "맨 아래"는 그룹 안 맨 아래가 아니라 사이드바 전체 맨 아래일 수 있다 — 그룹 순서를 먼저 확인(플랫폼 관리 그룹은 맨 위). ido 1차 의도 확인 사례.
- mnupms: `('page-context-manager', 'role', 'platform_admin', true)`
- 프론트: `FORGE_ROUTE_MAP`에 `'page-context-manager': '/page-context-manager'`, `iconMap`에 `Bot` 등록.
- 마이그레이션: `backend/migrations/versions/pltmnu_add_page_context_001.py` (`down_revision='bndsend_001'`).
- **langtxt 누락(보강 필요)**: 이 첫 사례는 `core.sidebar.page_context_manager` 키의 langtxt 번역을 등록하지 않아, EN/LA 화면에서 한국어 fallback만 나온다. 본 스킬에 §4.2 langtxt 단계를 추가한 계기다. 보강 시 KO '포지AI관리' / EN 'Forge AI' / LA 번역 3행을 위 INSERT 패턴으로 시드한다.
