---
name: cloudtype-cli
description: >
  Cloudtype 에 서비스를 배포하고 운영합니다 (프로젝트 생성, GitHub repo 에서
  앱·DB 배포, 환경변수·시크릿 관리, 로그 스트리밍, 컨테이너 셸 접근, 장애 진단).
  사용자가 Cloudtype, ctype, cloudtype.app 도메인을 언급하거나 서비스/데이터베이스/
  Redis/Postgres/MongoDB 배포를 요청할 때 사용합니다 — "Cloudtype" 단어를 명시적으로
  말하지 않더라도 동일하게 적용됩니다.
allowed-tools: Bash(ctype:*), Bash(curl:*), Bash(npm:*), Bash(which:*)
---

# Cloudtype CLI

GitHub 저장소를 [Cloudtype](https://cloudtype.io) 에 배포하고, 같은 deployment 의 로그·설정·셸을 활용해 문제 해결을 시도합니다. 배포·로그·셸·환경변수 등 인프라 작업은 공식 CLI (`ctype`) 로 수행하며, CLI 가 노출하지 않는 일부 조회 기능 (예: GitHub repo 목록) 은 동일한 `CLOUDTYPE_APIKEY` 로 Cloudtype HTTP API 를 호출합니다.

배포는 본질적으로 `.cloudtype/app.yaml` 작성 후 `ctype apply` 한 번으로 끝납니다. 실패 시에도 다른 preset 으로 갈아타거나 새 서비스를 만들지 않고, 동일 deployment 의 로그를 보고 `app.yaml` 을 수정한 뒤 다시 `ctype apply` 합니다.

---

## 🎯 목표

> GitHub 저장소 → Cloudtype 에서 정상 동작하는 서비스로 배포.

이 스킬의 책임은 **배포 부분에 한정**됩니다.
시스템 설계, 코드 작성, 푸시 같은 상위 작업은 상위 에이전트가 담당합니다.

---

## 🛂 사전 점검

배포 전 다음을 확인합니다.

```bash
which ctype                       # CLI 설치 여부
ctype whoami                      # 인증 + 로그인 계정
ctype use                         # 현재 컨텍스트 (project:stage on cluster)
```

CLI 미설치 시:
```bash
npm i -g @cloudtype/cli
```

미인증 시 (API 키가 환경변수로 주입되어 있다면 권장):
```bash
ctype login -t "$CLOUDTYPE_APIKEY"
```

API 키가 없다면 `ctype login` 으로 username/password 흐름 안내. 키 발급은 콘솔에서만 가능합니다.

---

## 🧭 입력 추론과 디폴트

사용자가 명시하지 않은 항목은 추론합니다. **세세한 사항을 일일이 묻지 않습니다.**

| 항목 | 디폴트 / 추론 방식 |
|---|---|
| repo | 사용자가 정확한 URL 을 주지 않은 경우 Cloudtype 에 연동된 GitHub 의 repo 목록을 조회하여 이름 매칭. 후보가 하나면 진행, 여러 개면 선택지 제시. ("GitHub 연동" 섹션 참고) |
| branch | 명시 없으면 `main` |
| project | 명시 없으면 repo 이름. 없으면 `ctype project create <name>` 으로 생성 |
| stage | 명시 없으면 `main` |
| deployment 이름 | 명시 없으면 repo 이름 (소문자 + 하이픈 정규화) |
| 옵션 | 사용자가 명시한 항목만 `app.yaml` 에 포함. 나머지는 서버 디폴트에 맡깁니다. |

### 추론 디폴트 (사용자 명시가 있으면 그게 절대 우선)

| 상황 | 디폴트 |
|---|---|
| DB 가 필요한데 종류 미명시 | `postgresql` |
| 백엔드 언어/런타임 미명시 (그리고 repo 신호도 없을 때) | `node` (Node.js) |
| 추가 인프라 (캐시/큐) 미명시 | 추가하지 않음 — 사용자가 명시한 서비스만 배포 |

사용자가 명시한 선택은 절대 우선이며 디폴트로 덮어쓰지 않습니다 (예: "mysql 로 해줘", "python flask 백엔드"). 디폴트를 적용한 경우 완료 보고에 어떤 디폴트가 적용됐는지 명시합니다.

⚠️ **사용자가 catalog 에 없는 것을 명시한 경우** (예: "mysql" → catalog 는 `mariadb` 만 있음) 는 디폴트 규칙이 아니라 "호환 대체 제안 + 사용자 확인" 경로로 이동합니다.

### Preset 결정

```bash
ctype presets                     # 사용 가능한 preset 마스터 목록 (100+ 종)
```

- 사용자 명시 ("Spring Boot", "MariaDB", "Rust", "Bun" 등) → preset 이름과 매칭
- repo 구조 신호 (사용자 명시 없을 때만):
  - `package.json` → `node` 계열 (`next.js`, `nest.js`, `remix` 등 더 구체적 후보 우선)
  - `requirements.txt` / `pyproject.toml` → `python-flask` / `python-django` / `python-fastapi`
  - `pom.xml` / `build.gradle` → `java-springboot`
  - `Cargo.toml` → `rust` (없으면 `dockerfile` 후보)
  - `Dockerfile` → `dockerfile`
  - 정적 프론트엔드 → `react` / `vue` / `html` 등

후보가 마스터 목록에 없으면 사용자에게 알리고 임의 대체 금지. 호환 가능한 대체가 있으면 (예: `mysql` → `mariadb`) **제안 후 사용자 확인을 받은 뒤** 진행합니다.

---

## 🚀 배포 절차

### 1. 컨텍스트 준비

```bash
ctype use                                  # 현재 컨텍스트 확인
ctype use @<scope>/<project>:<stage>       # 다른 프로젝트/스테이지로 전환
```

프로젝트가 없으면:
```bash
ctype project create <name>                # 새 프로젝트 생성
```

### 2. `app.yaml` 작성

기본 위치는 `.cloudtype/app.yaml` 이며, 다른 경로는 `-f` 로 지정합니다.

각 preset 의 필수·선택 필드, 시크릿 참조 규칙, 멀티 서비스 묶기 같은 작성 가이드는 [`reference/yaml-schema.md`](reference/yaml-schema.md) 를 참고합니다. 새 yaml 을 작성하거나 기존 yaml 을 수정할 때 먼저 이 reference 를 읽습니다.

### 3. 배포

```bash
ctype apply                                # .cloudtype/app.yaml 사용
ctype apply -f path/to/file.yaml           # 다른 파일 사용
ctype apply -a                             # 모든 stage 에 적용
```

### 4. 결과 확인

```bash
ctype list                                 # stage 의 모든 deployment + 상태
ctype routes                               # HTTP/TCP 라우트 + URL
ctype services                             # 서비스 풀 사용 상태
```

배포 직후 status 가 `Running` 으로 안정될 때까지 몇 십 초 걸릴 수 있습니다.

### 5. 재배포 / 업데이트

같은 `app.yaml` 을 다시 `ctype apply` 하면 기존 deployment 가 재배포됩니다 (새 빌드 세션 생성). spec 이 같으면 단순 재배포, 변경되면 업데이트 + 재배포.

```bash
ctype update <deployment>                  # GitHub 의 최신 커밋으로 재배포
```

---

## 🔐 시크릿과 환경변수

stage 단위 시크릿 / 환경변수는 별도 명령으로 관리합니다.

```bash
ctype stage secret DB_PASSWORD "dFhDn..."  # 시크릿 저장
ctype stage variable LOG_LEVEL info        # 평문 환경변수 저장
ctype stage secret DB_PASSWORD -r          # 시크릿 삭제
```

`app.yaml` 의 `options.env[]` 에서 두 가지 형태 사용 가능:

```yaml
env:
  - name: NODE_ENV
    value: production           # 평문 (app.yaml 에 그대로 저장됨)
  - name: DB_PASSWORD
    secret: DB_PASSWORD         # 시크릿 참조 (stage secret store 의 키 이름)
```

**시크릿 참조 규칙**: `secret:` 형태의 참조는 **오직 `options.env[]` / `options.buildenv[]` 항목 안에서만** 동작합니다. 그 외 모든 `options.*` 필드 (예: DB preset 의 `rootpassword`) 에는 **plain 문자열만** 넣으세요.

**DB 배포 권장 패턴**:
1. 강한 패스워드 생성 → DB preset 의 `rootpassword` 에 plain 으로 넣고 배포
2. 같은 패스워드를 `ctype stage secret DB_PASSWORD <값>` 로 저장
3. 앱 서비스의 `app.yaml` 의 `env[]` 에서 `{ name: DB_PASSWORD, secret: DB_PASSWORD }` 로 참조

---

## 🎟️ 리소스 정책

`app.yaml` 의 `resources` 에는 **풀 종류만** 명시합니다. `cpu` / `memory` / `disk` 는 사용자가 명시한 경우에만 포함합니다. LLM 상식 추측 금지 — preset 별 디폴트 배분 (특히 프리티어 1GB 한도) 을 깨고 후속 배포를 막을 수 있습니다.

```yaml
resources:
  spot: true        # 프리티어 풀
  # spot: false     # 구독 풀
```

풀 선택:
- 사용자가 명시한 풀을 그대로 사용
- 명시 없으면 구독 풀(`spot: false`) 우선, 없으면 프리티어 풀(`spot: true`)
- 자동 선택 결과가 프리티어 풀인 경우 완료 보고에 운영 특성 안내 (주기적 자동 중지 등)

---

## 🌐 멀티 서비스 (frontend + backend + DB 등)

여러 서비스가 필요하면 각각 `.cloudtype/<name>.yaml` 로 분리해서 작성하고 순서대로 `apply` 합니다.

```bash
ctype apply -f .cloudtype/postgres.yaml    # DB 먼저
ctype apply -f .cloudtype/redis.yaml       # 캐시
ctype apply -f .cloudtype/api.yaml         # 백엔드
ctype apply -f .cloudtype/web.yaml         # 프론트
```

내부 서비스 간 호출은 deployment 이름이 곧 호스트:
- `postgres:5432`, `redis:6379`, `api:3000` 같은 식
- 같은 stage 의 서비스끼리는 자동으로 사내 DNS 로 연결됨

---

## 🔁 실패 대응

배포 실패 / `stopped` 상태 / 빌드 실패 시 흐름:

### 1. 로그 확인 (스트리밍 가능)

```bash
ctype logs <deployment>                    # 기본 follow 모드
ctype logs <deployment> -f false -l 100    # 원샷, 최근 100줄
ctype logs <deployment> -m                 # 타임스탬프 포함
ctype logs <deployment> -p                 # 이전 컨테이너 로그 (재시작 직전)
```

### 2. 진단

로그를 읽고 원인을 추론합니다. 일반적인 빌드/실행 오류는 로그가 충분한 단서를 줍니다.

Cloudtype 환경 컨벤션 중 자주 놓치는 것: Cloudtype 은 ingress 뒤에 있으므로 `X-Forwarded-For` 헤더가 들어옵니다. Express `trust proxy` 같은 프록시 인지 옵션이 꺼져 있으면 rate limiter 등에서 validation 에러가 발생할 수 있습니다.

### 3. 컨테이너 안 직접 확인

`ctype terminal <deployment>` 로 실행 중인 컨테이너 안에 들어가 상태를 직접 확인합니다. 컨테이너는 격리된 환경이며 패키지 설치나 시스템 권한 변경은 불가능합니다. 환경변수·파일·내부 연결 확인 같은 **조회/검증 작업**에 사용합니다.

확인할 만한 것:

- 환경변수가 의도대로 들어왔는지 (`env` 등)
- DB / 캐시 호스트로 네트워크가 닿는지
- 빌드 산출물의 파일 구조가 예상과 맞는지
- 로그 파일이 컨테이너 내부 어디에 쌓이는지

### 4. 수정 후 재배포

`app.yaml` 또는 stage secret/variable 만 수정 후:
```bash
ctype apply                                # 같은 deployment 에 재배포
```

코드 자체에 문제가 있으면 — repo 에 수정 푸시 후 `ctype update <deployment>`.

**3회 시도 한도** — 같은 종류 실패가 3회 반복되면 자동 재시도 중단하고 사용자에게 보고.

---

## ⛔ 사용자 확인이 필요한 결정

다음 동작은 자동으로 수행하지 않고 사용자 명시 확인 후에만 실행합니다.

- 다른 preset 으로 갈아타기 (예: `web` 실패 → `dockerfile` 로 재배포)
- 새 deployment 이름으로 별도 서비스 생성 (예: `web` 실패 → `docker-web` 새로 만들기)
- 리소스 사양 조정 (`cpu` / `memory` / `disk` / `replicas` — 자세한 정책은 "리소스 정책" 섹션)
- Dockerfile 자동 생성 또는 인라인 주입
- 시크릿 덮어쓰기 (이미 존재하는 키)
- 풀 종류 변경 (`spot: true` ↔ `spot: false`)
- 삭제 (`ctype remove`, 프로젝트 / 스테이지 삭제)

---

## 🧰 GitHub 연동

Cloudtype 콘솔에서 사용자가 한 번 GitHub OAuth 연동을 해두면, 이후 이 스킬은 동일한 `CLOUDTYPE_APIKEY` 로 GitHub repo 목록·브랜치를 조회할 수 있고, `app.yaml` 의 `context.git.url` 에 박힌 repo 를 Cloudtype 이 자동으로 클론·빌드합니다. push 시 webhook 으로 자동 재빌드됩니다.

### Repo 조회

사용자가 정확한 URL 대신 대략적인 이름만 말한 경우 (예: "내 주소 축약기 배포") 다음 흐름으로 repo 를 확정합니다.

```bash
curl -sS -H "Authorization: Bearer $CLOUDTYPE_APIKEY" \
  https://api.cloudtype.io/oauth/github/accounts
# → [{ "installationid": <ID>, "name": "<github-username>", ... }]

curl -sS -H "Authorization: Bearer $CLOUDTYPE_APIKEY" \
  "https://api.cloudtype.io/oauth/github/repository/<installationid>"
# → [{ "name": ..., "url": ..., "defaultbranch": ..., ... }, ...]
```

결과 목록에서 사용자 발화의 키워드와 이름·설명을 매칭하여 후보를 도출합니다. 후보가 하나면 그대로 진행, 여러 개면 선택지를 제시합니다. 후보가 없거나 `/oauth/github/accounts` 가 빈 결과인 경우 콘솔에서 해당 repo 를 GitHub 연동에 추가하도록 안내합니다.

확정된 `url` 을 `app.yaml` 의 `context.git.url` 에 사용합니다.

```yaml
context:
  git:
    url: https://github.com/<owner>/<repo>.git
    ref: main
```

### 책임 경계

이 스킬은 push 가 끝난 시점부터 진입합니다. 코드 작성, 새 GitHub repo 생성, `git push` 는 호출하는 상위 에이전트와 그 에이전트의 GitHub 인증 (PAT 또는 OAuth) 영역입니다.

GitHub 연동 설치/해제는 사용자가 콘솔에서 수행합니다. 연동되지 않은 repo 라면 콘솔에서 추가하도록 안내합니다.

---

## 📚 참조 자료

일반 배포는 본 문서 (SKILL.md) 만으로 완결됩니다. 아래 자료는 **해당 상황이 실제로 발생했을 때만** 열어봅니다.

- [`reference/cli-quickref.md`](reference/cli-quickref.md) — 자주 쓰는 명령 압축 모음
- [`reference/yaml-schema.md`](reference/yaml-schema.md) — `app.yaml` 의 전체 필드 가이드

---

## 🔐 인증

`CLOUDTYPE_APIKEY` 환경변수 하나로 CLI 와 보조 API 호출 둘 다 인증합니다.

- **CLI 인증**: `ctype login -t "$CLOUDTYPE_APIKEY"` 한 번 (이후 `ctype whoami` 로 확인)
- **API 보조 호출**: `Authorization: Bearer $CLOUDTYPE_APIKEY` 헤더로 `curl` 직접 호출 (GitHub repo 조회 등)

환경별 권장:

- **통합 환경** (Porter AI 같은 샌드박스): API 키가 환경변수로 미리 주입됨 — 위 한 줄이면 끝
- **로컬 개발**: 사용자가 콘솔에서 API 키 발급 후 환경변수로 export
- **공유/임시 환경**: `ctype login` 의 username/password 인터랙티브 흐름

매 작업 전에 `ctype whoami` 로 인증 상태 먼저 확인합니다.
