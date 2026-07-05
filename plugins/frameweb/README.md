# frameweb 플러그인

Epic Cloud 메타데이터 폼을 만들고 유지보수하는 스킬 플러그인. 폼 요구사항 수집부터 화면 배치·바인딩·스크립트·상태 제어·적재·배포·런타임 검증까지를 스킬로 담는다.

지금은 **첫 샘플**로 폼 상태 매트릭스 스킬 1종(`frameweb-state`)을 담았다. 배포 방식이 확인되면 나머지 폼 스킬을 같은 방식으로 확장한다.

## 담은 스킬

| 스킬 | 설명 |
|------|------|
| `frameweb-state` | 폼 디자이너 "상태 매트릭스" 탭 — 상태별로 화면의 각 칸을 잠그거나 숨기는 규칙을 메타데이터로 선언하고 디버깅한다 |

### 확장 예정 (같은 방식으로 이어서 담는다)

- 라이프사이클: `frameweb-require` · `frameweb-canvas` · `frameweb-load` · `frameweb-deploy` · `frameweb-verify`
- 폼 디자이너 탭: `frameweb-bindchain` · `frameweb-binding` · `frameweb-script` · `frameweb-sandbox`
- 보조: `frameweb-analyze` · `frameweb-multilanguage` · `frameweb-user-manual` · `frameweb-menu`

## 환경 설정 (.env)

DB 비밀번호·로그인 계정 같은 민감정보를 쓰는 스킬은 값을 스킬 본문에 넣지 않고 환경변수로 참조한다. `.env.example` 을 `.env` 로 복사해 값을 채운 뒤, 스킬을 쓰기 전에 로드한다.

```shell
cp .env.example .env      # 값을 채운다 (특히 비밀번호)
set -a; . .env; set +a    # 예시 명령의 ${FRAMEWEB_DB_PASSWORD} 등이 채워진다
```

`.env` 는 `.gitignore` 로 저장소에 올라가지 않는다. 첫 샘플 `frameweb-state` 는 민감값을 쓰지 않아 `.env` 없이도 동작한다.

## 설치

```shell
# 1) 이 저장소를 마켓플레이스로 추가
claude plugin marketplace add FrameWeb/plugin

# 2) frameweb 플러그인 설치
claude plugin install frameweb@frameweb-marketplace
```

로컬 디렉토리로 설치하려면 1단계 경로를 `/home/ido/frameweb-plugin` 로 바꾼다.

## 사용법

스킬은 트리거 키워드로 자동 활성화된다. 예를 들어 "상태 매트릭스", "칸 잠금", "state-matrix" 같은 요청이 `frameweb-state` 를 부른다. 상세 트리거는 `skills/frameweb-state/SKILL.md` 의 description 에 있다.
