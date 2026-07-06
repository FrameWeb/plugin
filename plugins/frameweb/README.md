# frameweb 플러그인

Epic Cloud 메타데이터 폼을 만들고 유지보수하는 스킬 플러그인. 폼 요구사항 수집부터 화면 배치·바인딩·스크립트·상태 제어·적재·배포·런타임 검증까지 폼 개발 전 과정을 스킬로 담는다.

폼 스킬 14종을 담는다. 각 스킬은 트리거 키워드로 자동 활성화된다.

## 담은 스킬

### 라이프사이클 (폼 개발 순서)

| 스킬 | 설명 |
|------|------|
| `frameweb-require` | 요구사항 수집 — 자연어·스크린샷·참조 자료에서 구조화된 폼 스펙을 만든다 |
| `frameweb-canvas` | 화면 배치 — 캔버스/레이아웃/컴포넌트(FRMCMP). 새 폼을 0에서 만드는 진입점 |
| `frameweb-load` | DB 적재 — forms.sql을 시스템 DB에 실행하고 결과를 검증한다 |
| `frameweb-deploy` | 배포 — 폼 정의를 스냅샷(FRMMST)으로 저장해 런타임에 반영한다 |
| `frameweb-verify` | 런타임 검증 — 실제 화면 동작을 확인하는 테스트 의뢰서를 만든다 |

### 폼 디자이너 탭 (1:1)

| 스킬 | 설명 |
|------|------|
| `frameweb-bindchain` | 컴포넌트와 바인딩을 연결하고 체인(Observer Chain) 순서를 구성한다 |
| `frameweb-binding` | 조회·입력 SQL(bndquery)과 CRUD 매핑을 만들고 디버깅한다 |
| `frameweb-script` | 폼 스크립트 — Web Worker 샌드박스에서 실행되는 이벤트 핸들러를 작성한다 |
| `frameweb-sandbox` | 폼 전용 AI 도우미 — 특정 폼에 붙는 챗 에이전트의 작동 규칙을 설계한다 |
| `frameweb-state` | 상태 매트릭스 — 상태별로 화면의 각 칸을 잠그거나 숨기는 규칙을 선언한다 |

### 보조

| 스킬 | 설명 |
|------|------|
| `frameweb-analyze` | 폼 분석 — 기존 폼의 메타데이터를 읽고 구조를 설명한다 (읽기 전용) |
| `frameweb-multilanguage` | 폼 다국어 — 라벨·컬럼 헤더·메뉴·본문의 번역을 추출·등록·반영한다 |
| `frameweb-user-manual` | 사용자 매뉴얼 — ERP 화면 사용 설명서를 작성해 문서관리 모듈에 게시한다 |
| `frameweb-menu` | ⚠ 폼이 아니라 **개발자 플랫폼 사이드바 메뉴**(PLTMNU) 스킬. 폼·도구를 좌측 메뉴에 노출하고 권한을 부여한다 |

## 환경 설정 (.env)

DB 비밀번호·로그인 계정·접속 좌표 같은 값은 스킬 본문에 넣지 않고 환경변수로 참조한다. `.env.example` 을 `.env` 로 복사해 값을 채운 뒤, 스킬을 쓰기 전에 로드한다.

```shell
cp .env.example .env      # 값을 채운다 (특히 비밀번호)
set -a; . .env; set +a    # 예시 명령의 ${FRAMEWEB_DB_PASSWORD} 등이 채워진다
```

`.env` 는 `.gitignore` 로 저장소에 올라가지 않는다. `frameweb-state`·`frameweb-require` 처럼 DB 접속이 없는 스킬은 `.env` 없이도 동작한다.

## 설치

```shell
# 1) 이 저장소를 마켓플레이스로 추가
claude plugin marketplace add FrameWeb/plugin

# 2) frameweb 플러그인 설치
claude plugin install frameweb@frameweb-marketplace
```

로컬 디렉토리로 설치하려면 1단계 경로를 `/home/ido/frameweb-plugin` 로 바꾼다.

## 사용법

스킬은 트리거 키워드로 자동 활성화된다. 예를 들어 "상태 매트릭스", "칸 잠금" 은 `frameweb-state` 를, "폼 요구사항", "새 폼 기획" 은 `frameweb-require` 를 부른다. 상세 트리거는 각 `skills/frameweb-*/SKILL.md` 의 description 에 있다.
