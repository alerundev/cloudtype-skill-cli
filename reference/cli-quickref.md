# CLI Quick Reference

자주 쓰는 `ctype` 명령 압축 모음. 본 문서는 SKILL.md 가 다루지 않는 자잘한 옵션과 결과 형식만 다룹니다.

## 인증 / 컨텍스트

```bash
ctype login -t "$CLOUDTYPE_APIKEY"          # API 키 인증 (권장)
ctype login                                 # username/password 인터랙티브
ctype login -u <user> -p <pass>             # 비인터랙티브
ctype logout

ctype whoami                                # 현재 로그인 계정 + 서버 URL
ctype use                                   # 현재 stage (project:stage on cluster)
ctype use myproject                         # stage 전환 (default stage = main)
ctype use myproject:development             # 특정 stage 로
ctype use :development                      # 같은 project 의 다른 stage 로
ctype use @teamname/myproject:main          # 팀 스코프 명시
```

## 프로젝트 / 클러스터

```bash
ctype projects                              # 프로젝트 목록 (현재 scope)
ctype project create <name>                 # 새 프로젝트 생성
ctype project remove <name>                 # 프로젝트 삭제 (확인 후)
ctype project connect <giturl>              # git repo 연결
ctype project connect <giturl> -r           # 읽기 전용 연결
ctype project key <name>                    # SSH 키 생성
```

## 배포

```bash
ctype apply                                 # .cloudtype/app.yaml 으로 배포
ctype apply -f path/to/file.yaml            # 다른 파일 사용
ctype apply -t @scope/project:stage         # target stage 명시
ctype apply -a                              # 모든 stage 에 적용
ctype apply -p                              # prepare only (dry-run 비슷)
ctype apply -r <git-revision>               # 특정 git revision 으로 배포

ctype update <deployment>                   # repo 최신 커밋으로 재배포
ctype update -a <deployment>                # 모든 stage 에서 재배포

ctype remove <deployment>                   # deployment 삭제
ctype remove <name1> <name2>                # 여러 개 동시 삭제
```

## 조회

```bash
ctype list                                  # stage 의 모든 deployment + status
ctype services                              # 서비스 풀 사용 상태 + replica 수
ctype routes                                # HTTP/TCP 라우트 + URL + 연결 상태
ctype presets                               # 사용 가능한 preset 마스터 목록 (100+)
```

## 로그 / 디버깅

```bash
ctype logs <deployment>                     # follow 모드 (기본 -f true)
ctype logs <deployment> -f false            # 원샷 (현재까지의 로그만)
ctype logs <deployment> -l 100              # 최근 N 줄 (기본 250)
ctype logs <deployment> -s 600              # 최근 N 초
ctype logs <deployment> -m                  # 타임스탬프 포함
ctype logs <deployment> -p                  # 이전 컨테이너 로그 (재시작 직전 / crashloop 진단)

ctype terminal <deployment>                 # 컨테이너 셸 진입 (실시간)
```

## 환경변수 / 시크릿

```bash
ctype stage variable LOG_LEVEL info         # 평문 환경변수 설정
ctype stage variable LOG_LEVEL -r           # 환경변수 삭제
ctype stage variable                        # 현재 stage 의 변수 목록

ctype stage secret DB_PASSWORD "dFhDn..."   # 시크릿 설정
ctype stage secret DB_PASSWORD -r           # 시크릿 삭제
ctype stage secret                          # 시크릿 키 목록 (값은 안 보임)
```

## 자주 쓰는 옵션 패턴

- `-g, --verbose` — 자세한 출력 (디버깅 시)
- `-t, --target` — target stage 명시 (현재 컨텍스트와 다른 곳 작업할 때)
- `-h, --help` — 어느 명령이든 `--help` 로 옵션 확인

## 종료 코드

CLI 는 표준 종료 코드를 사용합니다 — 실패 시 non-zero. 스크립트/에이전트가 결과 판정 시:

```bash
ctype apply && echo "OK" || echo "FAILED"
```

## 출력 파싱

CLI 출력은 사람 친화 표 형식입니다. JSON 모드 플래그는 현재 노출돼 있지 않습니다.
파싱이 필요한 경우 출력을 `awk` / `grep` 으로 다듬거나, 그 작업만큼은 API 트랙으로
보조 호출하는 것을 고려합니다.
