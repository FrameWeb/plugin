# frameweb-marketplace

Epic Cloud 폼 개발·유지보수 스킬을 Claude Code 플러그인으로 배포하는 마켓플레이스다.

이 저장소는 마켓플레이스 1개(`frameweb-marketplace`)와 플러그인 1개(`frameweb`)를 담는다.

```
.
├── .claude-plugin/
│   └── marketplace.json      ← 마켓플레이스 정의 (플러그인 목록)
├── plugins/
│   └── frameweb/             ← frameweb 플러그인 (상세는 plugins/frameweb/README.md)
│       ├── .claude-plugin/plugin.json
│       ├── README.md
│       └── skills/           ← 폼 스킬 (첫 샘플: frameweb-state)
├── .env.example              ← 스킬용 환경변수 카탈로그 (.env 로 복사해 사용)
└── README.md                 ← 이 파일
```

## 담긴 플러그인

| 플러그인 | 설명 |
|----------|------|
| `frameweb` | Epic Cloud 폼 개발·유지보수 스킬. 첫 샘플로 폼 상태 매트릭스 스킬(`frameweb-state`)을 담고 폼 라이프사이클 스킬로 확장 |

## 로컬 설치 절차

```shell
# 1) 이 디렉토리를 마켓플레이스로 추가
claude plugin marketplace add /home/ido/frameweb-plugin

# 2) frameweb 플러그인 설치
claude plugin install frameweb@frameweb-marketplace
```

## GitHub 설치

이 저장소가 GitHub(`FrameWeb/plugin`)에 올라간 뒤에는 repo 참조로 바로 추가할 수 있다.

```shell
claude plugin marketplace add FrameWeb/plugin
claude plugin install frameweb@frameweb-marketplace
```

## 민감정보 정책

이 저장소는 공개다. DB 비밀번호·로그인 계정 같은 값은 스킬 본문에 넣지 않고 환경변수로 참조하며, 실제 값은 `.env`(git 제외)에 둔다. 변수 목록은 `.env.example` 에 있다.

## 설치 전 검증

```shell
cd /home/ido/frameweb-plugin
claude plugin validate .
```
