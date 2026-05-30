# Cloudtype Skill (CLI)

[Anthropic Agent Skills](https://docs.anthropic.com/en/docs/build-with-claude/agents-and-tools/agent-skills) 표준을 따르는 [Cloudtype](https://cloudtype.io) 배포 스킬입니다.

상위 에이전트 (Claude Code, Cursor, OpenAI Codex, [Porter AI](https://getporter.ai) 등) 에 얹어 **GitHub 저장소를 Cloudtype 에 배포하고, 실패 시 같은 deployment 의 로그·셸·설정만으로 해결을 시도**합니다.

## 설계 원칙

- 배포는 본질적으로 `.cloudtype/app.yaml` 작성 + `ctype apply` 한 번
- 실패하면 **같은 deployment** 의 로그·셸을 사용해 옵션만 조정
- 다른 preset 으로 갈아타거나 새 서비스 만드는 우회는 사용자 승인 후에만
- 소스코드 수정은 위치/방향만 안내, 직접 수행하지 않음
- 리소스 사양 (`cpu` / `memory` / `disk`) 임의 박지 않음 — preset 디폴트 배분 존중

자세한 정책과 흐름은 [`SKILL.md`](./SKILL.md) 를 참조합니다.

## 구성

```
cloudtype-skill-cli/
├── SKILL.md                       # 에이전트가 따르는 정책/흐름 (진입점)
└── reference/
    ├── cli-quickref.md            # 자주 쓰는 ctype 명령 압축 모음
    └── yaml-schema.md             # .cloudtype/app.yaml 필드 가이드
```

## 사전 요구사항

- Node.js v12 이상 (CLI 가 npm 패키지)
- Cloudtype API 키 — [발급 가이드](https://docs.cloudtype.io/guide/references/apikey)

## 설치 + 인증

```bash
npm i -g @cloudtype/cli
ctype login -t "$CLOUDTYPE_APIKEY"
ctype whoami      # 인증 확인
```

API 키를 환경변수로 주입하는 것을 권장합니다 (예: 통합 환경, CI, Porter AI 같은 샌드박스).

## 사용 예 (에이전트한테 던지는 자연어)

이 스킬이 진입하는 두 가지 모드:

### 1. 배포된 서비스에 대한 작업 (코드는 이미 GitHub 에 있음)

상위 에이전트는 repo URL 만 넘기고, 스킬은 `app.yaml` 작성 → `ctype apply` → 결과 확인.

- "내 GitHub repo `<owner>/<name>` 을 Cloudtype 에 배포해줘. 메인 브랜치."
- "마지막 배포가 stopped 됐어. 로그 보고 고쳐줘."
- "DB 비밀번호를 회전했어. 앱 서비스 환경변수 갱신하고 재배포."
- "Cloudtype 에 mariadb 하나 띄워줘."

### 2. 상위 에이전트가 코드 생성 + push 까지 한 뒤 배포까지 위임

상위 에이전트가 코드를 만들고 자기 GitHub 인증으로 push 한 뒤, 스킬을 호출하여 배포까지 한 번에.

- "URL 단축 서비스 만들어줘. Redis 캐싱, DB 는 PostgreSQL. Cloudtype 에 배포까지."
- "방문자 카운터 페이지 만들어서 Cloudtype 에 배포해줘."

> 코드 생성 / GitHub repo 생성 / `git push` 는 **이 스킬의 책임이 아닙니다.** 상위 에이전트와 그 에이전트가 가진 GitHub 인증 (PAT 또는 OAuth) 의 책임입니다. 이 스킬은 push 가 끝난 시점부터 진입합니다.

스킬이 알아서 `app.yaml` 작성 → `ctype apply` → 결과 확인까지 진행합니다. 디폴트 적용 시 (예: DB 종류 미명시 → PostgreSQL) 완료 보고에 명시합니다.

## 정책 요약

- **사용자 명시는 절대 우선**. 명시 없을 때만 디폴트 (DB = PostgreSQL, 백엔드 = Node.js).
- **같은 이름의 deployment 가 이미 존재**하면 배포 중단 + 사용자 확인.
- **빌드/실행 로그**로 원인이 명확한 수정은 보고 후 같은 `ctype apply` 재호출.
- **자동 재시도 = 같은 처방 최대 3회**.
- **시크릿 조회 / 서비스 삭제는 콘솔에서**.
- **리소스 자동 조정 금지**. preset 디폴트 배분 존중.
- **Dockerfile 자동 생성 금지**. 사용자가 dockerfile preset 을 명시한 경우에만.

자세한 내용은 [`SKILL.md`](./SKILL.md) 참조.

## 환경변수

| 이름 | 용도 |
|---|---|
| `CLOUDTYPE_APIKEY` | API 키 (`ctype login -t` 에 전달) — 필수 |

## 라이선스

MIT. [LICENSE](./LICENSE) 참조.
