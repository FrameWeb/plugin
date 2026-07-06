---
name: frameweb-user-manual
description: ERP 화면의 사용자 매뉴얼을 작성해 문서관리 모듈에 등록하고 게시한다. 폼을 만든 뒤(frameweb-canvas→…→frameweb-verify) 사용자에게 안내하는 단계로, 폼 라이프사이클 보조 스킬이다. "사용자 매뉴얼 만들어줘/등록해줘", "매뉴얼을 문서관리에 추가", "X 화면 사용 설명서", "엔드유저 가이드 작성" 같은 요청이나, 특정 폼(frm_id)의 사용법 문서가 필요할 때 사용한다. 대상 비즈니스에 문서관리 모듈이 없으면 복제 설치하는 절차도 포함한다. Trigger: frameweb-user-manual, 폼 매뉴얼, 사용 설명서, 사용자 매뉴얼, 엔드유저 가이드.
user-invocable: true
---

# frameweb-user-manual — 사용자 매뉴얼 등록·게시

## 이 스킬이 하는 것

ERP 화면 하나의 사용자 매뉴얼을 만들어 그 비즈니스의 **문서관리**(시스템관리 > 문서 > 문서 관리)에 등록하고 게시한다. 다섯 단계로 진행한다.

```
① 문서관리 모듈 확인/설치 → ② 화면 사실 조사 → ③ 매뉴얼 작성 → ④ 등록·게시 → ⑤ 검증 → ⑥ 폼 도움말 버튼 연결
```

매뉴얼의 독자는 코드를 모르는 업무 담당자다. 이 전제가 모든 단계의 판단 기준이 된다.

이 스킬은 폼을 만든 뒤(frameweb-canvas → … → frameweb-verify) 사용자에게 그 화면의 사용법을 안내하는 단계로, 폼 라이프사이클 보조 스킬이다.

## 시작 전 확인 (사용자에게 물을 것)

| 항목 | 기본값 | 비고 |
|------|--------|------|
| 대상 화면 (frm_id) | — (필수) | 예: SMS400 |
| 비즈니스 (bis_id) | — (필수) | bismst 실값, 대소문자 구분 |
| 게시 여부 | 게시함 | 초안만 원하면 draft로 두기 |
| 공개 범위 | internal (조직 내) | private / internal / public |

기본값으로 충분하면 다시 묻지 않고 진행한다.

## ① 문서관리 모듈 확인 / 설치

문서관리는 비즈 DB 단위 모듈이다. 대상 비즈에 없으면 먼저 설치한다.

```bash
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {비즈DB} -t \
  -c "SELECT to_regclass('public.documents');"
```

- 결과가 비어 있으면 미설치 → 마이그레이션 적용:
  ```bash
  PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {비즈DB} \
    -f database/migrations/20260612_documents_table.sql
  ```
- 적용 전에 시드 호환을 확인한다. 마이그레이션은 역할 `ADMIN`/`USER`와 상위 메뉴 `SYS`(시스템관리)를 전제한다:
  ```bash
  psql ... -c "SELECT rol_cd FROM rolmst;"
  psql ... -c "SELECT menu_cd, title FROM mnumst WHERE menu_cd='SYS';"
  ```
  역할 체계가 다른 비즈라면 mnupms 시드 부분을 그 비즈의 역할에 맞게 고쳐서 적용한다.
- 적용 후 검증: documents 테이블 + mnumst DOC/DOC001 + mnupms 행이 생겼는지 SELECT로 확인한다.

## ② 화면 사실 조사 — 추측 금지

매뉴얼의 내용은 전부 실측에서 나온다. 확인되지 않은 동작은 쓰지 않는다.

| 원천 | 조회 | 얻는 것 |
|------|------|---------|
| bisfrm (시스템 DB ${FRAMEWEB_DB_NAME}) | `SELECT frm_id, frm_nm FROM bisfrm WHERE bis_id='{bis}' AND frm_id='{frm}'` | 화면 공식 이름 |
| frm_description (시스템 DB) | `SELECT fcmp_id, title, description FROM frm_description WHERE frm_id='{frm}' ORDER BY sort_order` | 폼 전체 설명(`fcmp_id='__form__'` 행) + 컴포넌트별 설명 — 매뉴얼의 1차 원천 |
| mnumst (비즈 DB) | 메뉴 트리에서 해당 frm_cd의 경로 | "화면 여는 방법" 메뉴 경로 |
| 도메인 문서 | `Grep "{frm}" /home/ido/epic/docs/architecture/domains/` | 비용 계산·정산·상태값 같은 업무 규칙, 사용자 영향 있는 주의점 |

조사량이 많으면(도메인 문서 + DB 다건) 에이전트에 위임해도 된다. 위임할 때 위 표의 조회문과 작성 규칙(아래 ③)을 프롬프트에 그대로 포함한다.

## ③ 매뉴얼 작성

### 구조 (이 순서를 유지)

```markdown
# {화면 공식 이름}({frm_id}) — 사용자 매뉴얼

## 이 화면은 무엇인가          (2~3문장 — 업무 흐름에서의 위치 포함)
## 화면 여는 방법              (메뉴 경로: **상위 > 메뉴명**)
## 화면 구성                   (영역별 표 — frm_description 컴포넌트 설명 기반)
## 주요 작업                   (절차 단위, 번호 매긴 단계)
## 용어와 숫자 읽는 법          (도메인 문서에서 확인된 것만 — 비용/상태/집계 시점 등)
## 주의사항                    (사용자에게 실제 영향 있는 것만 3~5개, 굵은 글씨 한 줄 + 보충)
## 관련 화면                   (같은 업무 흐름의 앞뒤 화면 한 줄씩)
```

### 작성 규칙

- 화면에서 보고 누르는 단위로만 설명한다. 테이블명·컬럼명·API 경로·bnd_id 같은 내부 식별자는 쓰지 않는다 — 독자가 코드를 모르기 때문이다.
- 폼·화면을 지칭할 때는 항상 `이름(코드)` 순서로 쓴다(예: `발송 내역(SMS400)`). 독자는 폼코드를 모르므로 이름이 먼저고 코드는 괄호 보조다.
- 기술 문서 톤의 평범한 한국어. 분량 600~1200단어.
- 조사에서 확인되지 않은 영역은 "화면에서 확인" 수준으로 짧게 두거나 생략한다. 추측으로 채우면 매뉴얼이 화면과 어긋나는 순간 신뢰를 전부 잃는다.
- 숫자의 의미(예상 vs 확정, 집계 시점)는 사용자가 가장 자주 오해하는 부분이므로 도메인 문서에 근거가 있으면 반드시 넣는다.

## ④ 등록·게시 (ERP API)

문서 생성은 DB INSERT가 아니라 실제 API로 한다 — 권한·검증 경로를 그대로 타는 것이 안전하다.

```bash
# 1. ERP 로그인 (토큰은 JSON body로 반환 — set-cookie 없음)
TOK=$(curl -s -X POST http://localhost:8181/api/auth/erp-login \
  -H 'Content-Type: application/json' \
  -d '{"username":"${FRAMEWEB_LOGIN_USER}","password":"${FRAMEWEB_LOGIN_PW}","bis_id":"{bis}"}' \
  | python3 -c "import json,sys;d=json.load(sys.stdin);print(d.get('access_token') or d.get('erp_access_token'))")

# 2. 문서 생성 (본문은 jq/python으로 JSON 인코딩 — 따옴표 깨짐 방지)
python3 -c "
import json
print(json.dumps({'title': '{제목}', 'content_md': open('/tmp/manual.md').read(), 'visibility': 'internal'}, ensure_ascii=False))" > /tmp/doc_payload.json
curl -s -X POST http://localhost:8181/api/documents \
  -H "Authorization: Bearer $TOK" -H "Cookie: erp_access_token=$TOK" -H "X-Bis-Id: {bis}" \
  -H 'Content-Type: application/json' -d @/tmp/doc_payload.json
# 응답의 doc_id 확보

# 3. 게시 (게시 여부 '게시함'일 때)
curl -s -X POST http://localhost:8181/api/documents/{doc_id}/publish \
  -H "Authorization: Bearer $TOK" -H "Cookie: erp_access_token=$TOK" -H "X-Bis-Id: {bis}"
```

- 제목 형식: `{화면 이름}({frm_id}) — 사용자 매뉴얼`
- Bearer와 Cookie를 둘 다 붙인다. localhost에서는 Bearer가 우선이고, 쿠키는 ERP 서브도메인 환경 대비다.

## ⑤ 검증

```bash
# 목록에 보이는지 (status=published 확인)
curl -s http://localhost:8181/api/documents -H "Authorization: Bearer $TOK" \
  -H "Cookie: erp_access_token=$TOK" -H "X-Bis-Id: {bis}"
# DB 교차 확인 1회
psql ... -d {비즈DB} -c "SELECT doc_id, status, visibility, length(content_md), slug FROM documents ORDER BY doc_id DESC LIMIT 3;"
```

사용자 보고에는 **확인 방법**을 포함한다: `${FRAMEWEB_APP_URL}?bis_id={bis}` → ${FRAMEWEB_LOGIN_USER}/${FRAMEWEB_LOGIN_PW} → 시스템관리 > 문서 > 문서 관리.

## ⑥ 폼 도움말 버튼 연결 (게시한 매뉴얼을 해당 화면의 ? 버튼에)

폼에는 도움말 URL 속성이 있다 — 값이 있으면 ERP 화면 우상단에 ? 버튼이 나타나고, 누르면 새 탭으로 그 주소가 열린다. 게시 주소를 여기 등록해 매뉴얼을 화면에서 바로 열 수 있게 한다.

```bash
# 1. help_url 등록 — 시스템 DB bisfrm.form_property_config(jsonb)에 root 병합
#    (통째 덮어쓰기 금지 — ai_chat, chain_action_button 같은 다른 키 보존. 메모리 jsonb-config-merge)
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d ${FRAMEWEB_DB_NAME} -c "
UPDATE bisfrm
SET form_property_config = COALESCE(form_property_config,'{}'::jsonb)
    || jsonb_build_object('help_url', '/docs/{slug}?bis_id={bis}')
WHERE bis_id='{bis}' AND frm_id='{frm}';"

# 2. 폼 재배포 — ERP 화면은 FRMMST 배포 스냅샷을 읽으므로 재배포 없이는 ? 버튼이 안 나타난다
TOK_DEV=$(curl -s -X POST http://localhost:8181/api/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"${FRAMEWEB_LOGIN_USER}","password":"${FRAMEWEB_LOGIN_PW}"}' | python3 -c "import json,sys;print(json.load(sys.stdin)['access_token'])")
# version은 필수 입력 — 현재 최신 버전을 조회해 같은 버전으로 덮어쓴다 (UPSERT)
#   SELECT MAX(version) FROM frmmst WHERE frm_id='{frm}';  (비즈 DB)
curl -s -X POST http://localhost:8181/api/forms/deploy/ \
  -H "Authorization: Bearer $TOK_DEV" -H "X-Bis-Id: {bis}" \
  -H 'Content-Type: application/json' -d '{"frm_id": "{frm}", "version": "{현재버전}"}'

# 3. 검증 — 배포 스냅샷에 help_url이 들어갔는지
PGPASSWORD=${FRAMEWEB_DB_PASSWORD} psql -h ${FRAMEWEB_DB_HOST} -p ${FRAMEWEB_DB_PORT} -U ${FRAMEWEB_DB_USER} -d {비즈DB} -t -c "
SELECT frm_id, form_data::jsonb #>> '{form,form_property_config,help_url}'
FROM frmmst WHERE frm_id='{frm}' ORDER BY version DESC LIMIT 1;"
```

- 주소 형식: `/docs/{slug}?bis_id={bis}` (게시 응답의 slug 사용). 같은 사이트 상대 경로라 로컬/프로덕션 양쪽에서 동작한다.
- ? 버튼 노출 조건: 도움말 URL 값 존재 + 우상단 버튼 설정(Chain Action Button)에서 Help 체크가 꺼져 있지 않을 것. 기본은 켜짐이므로 보통 URL만 넣으면 된다.
- 게시 문서가 internal(조직 내)이어도 ERP에 로그인한 사용자는 열람된다(열람 페이지가 로그인 정보를 함께 보낸다).

## 주의사항

- 이 절차는 **로컬 환경**에 반영된다. 프로덕션에 올리려면 별도 배포(모듈 마이그레이션 + 문서 이전)가 필요하다 — 보고에 이 사실을 명시하고, 배포는 epic-deploy 스킬로 진행한다.
- 매뉴얼에 화면 캡처 이미지를 넣으려면 모리에게 캡처를 의뢰한 뒤 문서 편집 화면에서 Ctrl+V로 붙여넣는 후속 작업이 필요하다(이미지 업로드는 편집 화면 경유가 표준).
- bis_id는 대소문자를 구분한다. 사용자 표기와 bismst 실값이 다를 수 있으니 SELECT로 확인 후 사용한다.
- 같은 화면의 매뉴얼이 이미 있는지 목록을 먼저 확인한다. 있으면 새 문서 대신 기존 문서 수정(PUT)을 제안한다.
