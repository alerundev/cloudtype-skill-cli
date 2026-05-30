---
name: cloudtype-cli
description: >
  Deploy and operate services on Cloudtype using the official `ctype` CLI:
  create projects, deploy apps and databases from GitHub repos, manage
  environment variables and secrets, stream logs, exec into containers,
  and troubleshoot failures. Use this skill whenever the user mentions
  Cloudtype, `ctype`, deploying to Korea/Seoul, the `cloudtype.app`
  domain, or asks to deploy a service / database / Redis / Postgres /
  MongoDB on a Korean PaaS — even if they don't say "Cloudtype" explicitly.
allowed-tools: Bash(ctype:*), Bash(npm:*), Bash(npx:*), Bash(which:*), Bash(curl:*), Bash(git:*), Bash(gh:*)
---

# Cloudtype CLI

GitHub 저장소를 [Cloudtype](https://cloudtype.io) 에 배포하고, 같은 deployment 의
로그·설정·셸 을 활용해 문제 해결을 시도하는 스킬입니다. 모든 작업은 공식 CLI (`ctype`) 로
수행하며, 별도의 HTTP 클라이언트 / API 직접 호출 / SDK 가 필요하지 않습니다.

배포 자체는 본질적으로 *"`.cloudtype/app.yaml` 작성 → `ctype apply`"* 로 끝납니다.
실패 시에도 다른 preset 으로 갈아타거나 새 서비스를 만들지 않고, 동일 deployment 의
로그를 보고 `app.yaml` 을 수정한 뒤 `ctype apply` 를 다시 호출하는 흐름으로 처리합니다.

---

## 🎯 목표

> GitHub 저장소 → Cloudtype 에서 정상 동작하는 서비스로 배포.

이 스킬의 책임은 **배포 부분에 한정**됩니다.
시스템 설계, 코드 작성, 푸시 같은 상위 작업은 상위 에이전트가 담당합니다.

---

## 🚫 절대 하지 않는 것

사용자가 명시적으로 요청하지 않은 한 다음은 직접 수행하지 않습니다.

- **소스코드 직접 수정** — 위치와 수정 방향만 안내합니다.
- **다른 preset 으로 갈아타기** — 예: `web` 실패 → `dockerfile` 로 재배포 (금지)
- **새 deployment 이름으로 별도 서비스 생성하여 우회** — 예: `web` 실패 → `docker-web` 새로 만들기 (금지)
- **Dockerfile 자동 생성 또는 인라인 주입** — 사용자가 명시적으로 요청한 경우에만 사용합니다.
- **리소스 사양 자동 조정** — `cpu` / `memory` / `disk` / `replicas` 같은 세부 사양은 사용자가 명시한 경우에만 `app.yaml` 의 `resources` 에 포함합니다. 자동 증설/축소 금지.
- **시크릿 조회** — 필요 시 Cloudtype 콘솔 사용을 안내합니다.
- **삭제 (서비스 / 프로젝트 / 스테이지)** — `ctype remove` 는 사용자 명시 확인 후에만 실행합니다.
- **GitHub 연동 설정 변경** — 이미 연결된 상태를 활용만 합니다.
- **구독 / 결제 / 리소스 풀 구매** — 사용자가 콘솔에서 직접 처리합니다.

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
| repo | 사용자가 정확한 URL 을 주지 않은 경우 `gh repo list` 또는 사용자 확인 후 결정 |
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

이 디폴트는 마찰 감소용입니다. 사용자 명시 (예: "mysql 로 해줘", "python flask 백엔드") 가 있으면 **그 선택이 절대 우선**이며 디폴트로 덮어쓰지 않습니다. **디폴트를 적용했다면 완료 보고에 반드시 명시**합니다.

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
ctype project create <name>                # 새 프로젝트 생성 (클러스터 자동 배치)
```

Cloudtype 이 사용자 계정에 맞는 클러스터를 자동으로 선택합니다. 일반적으로 사용자가 클러스터를 지정할 일은 없습니다. 만일 yaml 에 클러스터 명시가 필요한 특수 상황이라면 해당 계정에서 사용 가능한 클러스터를 먼저 조회하여 그 값을 쓵니다 (일반적으로 노출되는 클러스터는 한 계정 기준 1개).

### 2. `app.yaml` 작성

기본 위치: `.cloudtype/app.yaml`. 다른 경로 가능하면 `-f` 로 지정.

**Node.js 앱 예시 (GitHub 연동):**

```yaml
name: my-api
app: node@24
options:
  ports: "3000"
  start: npm start
  install: npm ci
  buildenv: []
  env:
    - name: NODE_ENV
      value: production
    - name: DB_PASSWORD
      secret: DB_PASSWORD
  healthz: /
context:
  git:
    url: https://github.com/<owner>/<repo>
    ref: main
  preset: node
```

**PostgreSQL 예시:**

```yaml
name: postgresql
app: postgresql@16
options:
  rootusername: root
  rootpassword: "<생성한-평문-패스워드>"
```

> ⚠️ `rootpassword` 자리에는 **plain 문자열만**. `{secret: ...}` 같은 객체 안 됨.

**Redis 예시:**

```yaml
name: redis
app: redis@7
options:
  password: ""        # 인증 없음 (내부망에서만 호출 시)
```

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

| 증상 | 점검 포인트 |
|---|---|
| `[ServiceError] secret value must be a string` | `app.yaml` 의 `options.*` 에 객체가 들어갔는지 확인. 시크릿 참조는 `env[]` 안에서만. |
| `OOMKilled` | 메모리 부족. 사용자에게 `resources.memory` 증설 또는 코드 메모리 사용 조정 옵션 제시. |
| 빌드 실패 (`npm install` 등) | install 명령, 의존성, Node 버전 확인. `app.yaml` 의 `app:` (예: `node@24`) 와 repo 의 `engines.node` 일치 여부. |
| 헬스체크 실패 | `healthz` 경로가 실제 서버 라우트와 일치하는지. 시작 시간이 길면 `initialDelaySeconds` 또는 healthz 비활성화 옵션. |
| `X-Forwarded-For` validation 에러 (Express) | `app.set('trust proxy', 1)` 필요. Cloudtype 은 ingress 뒤에 있음. |

### 3. 셸 진입 (디버깅의 결정타)

```bash
ctype terminal <deployment>                # 실행 중 컨테이너에 셸 진입
```

- 환경변수 확인 (`env`)
- DB 연결 테스트 (`psql $DATABASE_URL -c "SELECT 1"`)
- 파일 시스템 / 로그 디렉토리 직접 확인

### 4. 수정 후 재배포

`app.yaml` 또는 stage secret/variable 만 수정 후:
```bash
ctype apply                                # 같은 deployment 에 재배포
```

코드 자체에 문제가 있으면 — repo 에 수정 푸시 후 `ctype update <deployment>`.

**3회 시도 한도** — 같은 종류 실패가 3회 반복되면 자동 재시도 중단하고 사용자에게 보고.

---

## ⛔ 자동으로 분기하지 않는 결정

다음 결정은 사용자 확인 후에만 실행합니다.

- 다른 preset 으로 갈아타기
- 새 deployment 이름으로 별도 서비스 생성
- 리소스 사양 조정 (`cpu` / `memory` / `disk` / `replicas`)
- Dockerfile 자동 생성
- 시크릿 덮어쓰기 (이미 존재하는 키)
- 풀 종류 변경 (`spot: true` ↔ `spot: false`)
- 서비스 / 프로젝트 / 스테이지 삭제

---

## 🧰 GitHub 연동

이미 연동되어 있는 상태를 활용만 합니다. 설치/해제는 사용자가 콘솔에서.

`app.yaml` 의 `context.git.url` 에 GitHub URL 을 주면 Cloudtype 이 webhook 으로 푸시 감지하고 자동 빌드합니다 (연동된 repo 에 한해).

```yaml
context:
  git:
    url: https://github.com/<owner>/<repo>
    ref: main
```

비공개 repo 라면 콘솔에서 GitHub 연동 설치가 먼저 필요합니다.

---

## 📚 참조 자료

일반 배포는 본 문서 (SKILL.md) 만으로 완결됩니다. 아래 자료는 **해당 상황이 실제로 발생했을 때만** 열어봅니다.

- [`reference/cli-quickref.md`](reference/cli-quickref.md) — 자주 쓰는 명령 압축 모음
- [`reference/yaml-schema.md`](reference/yaml-schema.md) — `app.yaml` 의 전체 필드 가이드

---

## 🔐 인증

이 스킬은 다음 환경에서 동작하도록 설계됐습니다.

- **`CLOUDTYPE_APIKEY` 환경변수가 주입된 경우** (포터 같은 통합 환경): `ctype login -t "$CLOUDTYPE_APIKEY"` 한 번이면 끝
- **로컬 개발 환경**: 사용자가 콘솔에서 API 키 발급 후 같은 방법
- **공유/임시 환경**: `ctype login` 의 username/password 흐름

`whoami` 로 항상 인증 상태 먼저 확인합니다.
