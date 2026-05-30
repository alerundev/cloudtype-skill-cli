# `.cloudtype/app.yaml` 스키마 가이드

`ctype apply` 가 받는 deployment 파일의 필드 가이드. 이 문서는 SKILL.md 의 배포 절차 예시에서 다루지 않는 옵션을 설명합니다.

## 최소 필수 필드

```yaml
name: <deployment-name>           # 필수. stage 안에서 고유
app: <preset>@<version>           # 필수. 예: node@24, postgresql@16, redis@7
```

## 전체 구조

```yaml
name: my-api
app: node@24

options:                          # preset 별 옵션 (필드는 preset 마다 다름)
  ports: "3000"                   # 노출 포트 (string 으로 권장)
  install: npm ci                 # 빌드 명령
  start: npm start                # 시작 명령
  buildenv: []                    # 빌드 시점 환경변수
  env:                            # 런타임 환경변수
    - name: NODE_ENV
      value: production
    - name: DB_PASSWORD
      secret: DB_PASSWORD         # stage secret store 의 키 참조
  healthz: /                      # 헬스체크 경로
  initialDelaySeconds: 30         # 헬스체크 시작 전 대기 (기본 0)

resources:                        # 옵션. 풀 종류만 명시 권장.
  spot: true                      # true=프리티어, false=구독 풀
  # cpu / memory / disk / replicas 는 사용자가 명시한 경우에만

context:                          # repo 연결 / preset 메타
  git:
    url: https://github.com/<owner>/<repo>
    ref: main                     # branch 또는 commit SHA
  preset: node                    # preset 이름
```

## Preset 별 핵심 옵션

### Node.js / Python / Go 등 framework preset

```yaml
options:
  ports: "3000"
  install: npm ci                 # 또는 pip install -r requirements.txt
  start: npm start                # 또는 python app.py
  healthz: /health
  env:
    - name: NODE_ENV
      value: production
```

### PostgreSQL / MariaDB / MySQL / MongoDB

```yaml
options:
  rootusername: root              # plain string only
  rootpassword: "<plain-password>"  # plain string only — secret 객체 X
  database: mydb                  # 초기 DB 이름 (선택)
  # tz: Asia/Seoul                # 시간대 (선택)
```

> ⚠️ `rootpassword` 같은 preset 옵션 필드는 **반드시 plain 문자열**. `{secret: ...}` 객체를 넣으면 배포 후 `[ServiceError] secret value must be a string` 으로 stopped 됩니다.

### Redis

```yaml
options:
  password: ""                    # 비우면 인증 없음 (내부망 전용 권장)
  # password: "<plain>"           # 인증 사용 시 plain string
```

### Dockerfile

```yaml
options:
  ports: "8080"
  # dockerfile: ./Dockerfile      # 위치 명시 (기본은 repo root)
context:
  preset: dockerfile
```

### Container (외부 이미지)

```yaml
app: container
options:
  image: nginx:1.25
  ports: "80"
context:
  preset: container
```

## 시크릿 참조 규칙

`{secret: <KEY>}` 형태의 참조는 **오직 다음 자리에서만** 동작합니다:

- `options.env[]` 항목
- `options.buildenv[]` 항목

그 외 모든 `options.*` 필드에는 plain 값 (string / number / boolean) 만 허용됩니다.

올바른 예:
```yaml
options:
  rootpassword: "Lp7zXq..."       # plain ✅
  env:
    - name: DB_PASSWORD
      secret: DB_PASSWORD         # ✅ env[] 안에서 참조
```

잘못된 예:
```yaml
options:
  rootpassword:                   # ❌ object in plain-only field
    secret: DB_PASSWORD
```

## 리소스 명시

생략 권장. 명시할 경우:

```yaml
resources:
  spot: true                      # 풀 종류만이 기본
  cpu: 1                          # 사용자 명시 시
  memory: 0.5                     # GB 단위
  disk: 1                         # GB 단위. DB 는 보통 명시 필요.
  replicas: 1
```

## healthz / initialDelaySeconds

```yaml
options:
  healthz: /                      # 응답 시 서비스 정상으로 판정
  initialDelaySeconds: 60         # 시작 시간 긴 앱 (Spring 등)
```

healthz 가 빈 문자열이면 헬스체크 비활성화.

## 멀티 서비스를 한 파일에

여러 서비스를 한 파일에 묶을 수 있습니다 (각 항목이 deployment 하나).

```yaml
- name: postgresql
  app: postgresql@16
  options:
    rootusername: root
    rootpassword: "..."

- name: api
  app: node@24
  options:
    ports: "3000"
    start: npm start
    env:
      - name: DATABASE_URL
        value: postgresql://root:.../postgres
  context:
    git:
      url: https://github.com/...
      ref: main
```

또는 파일을 나누고 순서대로 `apply` (권장 — 더 명확).

## 자주 헷갈리는 것

- `ports` 는 **string 으로** (`"3000"`). 숫자도 받지만 일관성 위해 string 권장.
- `app:` 의 버전 (`node@24`) 은 `ctype presets` 으로 확인. 명시 안 하면 최신 안정 버전.
- `env[].value` 는 항상 string. 숫자라도 `"production"` 같이 따옴표.
