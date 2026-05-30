# Cloudtype Skill (CLI)

[Anthropic Agent Skills](https://docs.anthropic.com/en/docs/build-with-claude/agents-and-tools/agent-skills) 표준을 따르는 [Cloudtype](https://cloudtype.io) 배포 스킬입니다.

다른 에이전트 (Claude Code, Cursor, OpenAI Codex, [Porter](https://getporter.ai) 등) 에 얹어 **GitHub 저장소를 Cloudtype 에 배포하고, 실패 시 같은 deployment 의 로그·셸·설정만으로 해결을 시도**합니다.

모든 작업은 공식 CLI (`ctype`) 로 수행하며, 별도의 HTTP 클라이언트 / SDK / API 직접 호출이 필요하지 않습니다.

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

API 키를 환경변수로 주입하는 것을 권장합니다 (예: 통합 환경, CI, Porter 같은 샌드박스).

## 다른 에이전트에 적용

### Claude Code

```bash
git clone https://github.com/alerundev/cloudtype-skill-cli ~/.claude/skills/cloudtype-cli
```

또는 Anthropic 의 plugin marketplace 가 등록된 시점에 `/plugin install cloudtype-cli` 형태로 설치.

### Cursor / Codex / 그 외

대상 에이전트가 [Agent Skills 표준](https://docs.anthropic.com/en/docs/build-with-claude/agents-and-tools/agent-skills) 을 지원하면 동일 폴더 구조로 적용 가능합니다.

### 일반 LLM 에이전트

이 폴더를 작업 디렉토리에 두고, 에이전트의 시스템 프롬프트에 다음 한 줄을 추가합니다:

> "Cloudtype 관련 작업 시 `cloudtype-skill-cli/SKILL.md` 정책을 따른다."

## 사용 예 (에이전트한테 던지는 자연어)

- "URL 단축 서비스 만들어줘. Redis 캐싱, DB 는 PostgreSQL. Cloudtype 에 배포까지."
- "방금 푸시한 내 GitHub repo `<owner>/<name>` 을 Cloudtype 에 배포해줘. 메인 브랜치."
- "Cloudtype 에 mariadb 하나 띄워줘."
- "마지막 배포가 stopped 됐어. 로그 보고 고쳐줘."

스킬이 알아서 `app.yaml` 작성 → `ctype apply` → 결과 확인까지 진행합니다. 디폴트 적용 시 (예: DB 종류 미명시 → PostgreSQL) 완료 보고에 명시합니다.

## API 트랙 vs CLI 트랙

이 스킬은 **CLI 트랙** (`ctype`) 입니다.

| 트랙 | 사용처 | 강점 |
|---|---|---|
| **CLI** (이 스킬) | 에이전트가 bash 가능한 환경. Claude Code, Cursor, Porter 샌드박스 등. | 짧은 SKILL.md, 셸 진입 (`ctype terminal`), 스트리밍 로그 (`ctype logs -f`), 토큰 풋프린트 작음 |
| **API** | HTTP 만 가능한 임베디드/제한 환경 | 머신 무관, JSON 응답 안정성, 미세 옵션 제어 |

대부분의 현대 에이전트 환경에서는 CLI 트랙을 권장합니다.

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

## 관련

- API 트랙 스킬: [`alerundev/cloudtype-skill`](https://github.com/alerundev/cloudtype-skill) — 같은 정책, HTTP API 기반 구현
- Cloudtype 문서: [docs.cloudtype.io](https://docs.cloudtype.io)
- Cloudtype CLI 문서: [docs.cloudtype.io/guide/cli/installation](https://docs.cloudtype.io/guide/cli/installation)
