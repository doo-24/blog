---
title: "[Docker & 컨테이너] 3편 — Docker Compose와 로컬 개발 환경"
date: 2026-03-20T15:00:00+09:00
draft: false
tags: ["Docker Compose", "로컬 개발", "컨테이너 네트워크", "볼륨", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "Docker Compose 핵심 문법과 서비스 정의, 컨테이너 네트워크(bridge, overlay)와 서비스 디스커버리, 볼륨과 데이터 영속성, 로컬 개발 환경 구성 실전, 프로필과 환경별 설정 분리까지"
---

서버 하나로 완결되는 서비스는 없다. API 서버, 데이터베이스, 캐시, 메시지 큐 — 로컬에서 이 모든 것을 각각 설치하고 버전 맞추는 건 신입 개발자의 첫날을 지옥으로 만든다.

팀 내에서 누군가는 PostgreSQL 15를 쓰고, 다른 누군가는 14를 쓰고, 또 다른 누군가는 MySQL을 쓰고 있다면 "내 로컬에서는 되는데"라는 말이 매일 반복된다.

Docker Compose는 `docker compose up` 한 줄로 전체 개발 환경을 정의하고 띄우는 도구다. 이 편에서는 Compose 핵심 문법부터 컨테이너 네트워크, 볼륨, 로컬 개발 환경 실전 구성, 그리고 프로필과 환경 분리까지 실제로 써먹을 수 있는 수준으로 다룬다.

---

## 1. Docker Compose 핵심 문법

### 1-1. compose.yaml — 파일 이름과 버전

오래된 튜토리얼을 보면 `docker-compose.yml` 파일에 `version: "3.9"` 같은 줄이 맨 위에 있다. 현재 Docker Compose V2에서는 이 `version` 필드가 더 이상 필요하지 않다. 무시되거나 경고만 출력된다.

권장 파일 이름은 `compose.yaml`이다 (`compose.yml`도 동작한다). Docker CLI는 현재 디렉터리에서 다음 순서로 파일을 탐색한다.

```
compose.yaml
compose.yml
docker-compose.yaml
docker-compose.yml
```

앞에 위치한 파일이 우선한다. 새 프로젝트라면 `compose.yaml`을 쓰는 것이 표준이다.

```yaml
# compose.yaml — version 필드 없음 (V2 방식)
services:
  app:
    build: .
    ports:
      - "8080:8080"
```

### 1-2. 최상위 키 구조

Compose 파일은 네 가지 최상위 키로 구성된다.

| 키 | 역할 |
|----|------|
| `services` | 실행할 컨테이너 정의 (필수) |
| `networks` | 커스텀 네트워크 정의 |
| `volumes` | Named Volume 정의 |
| `configs` / `secrets` | 설정값·민감 정보 주입 (주로 Swarm) |

대부분의 로컬 개발은 `services`와 `volumes`, `networks` 세 가지로 충분하다.

### 1-3. 서비스 정의 핵심 필드

#### image vs build

```yaml
services:
  # 이미 빌드된 이미지를 직접 사용
  redis:
    image: redis:7-alpine

  # 로컬 Dockerfile로 빌드
  app:
    build:
      context: .          # Dockerfile이 있는 디렉터리
      dockerfile: Dockerfile.dev   # 기본값은 Dockerfile
      args:
        BUILD_ENV: development
```

#### ports vs expose

```yaml
services:
  app:
    ports:
      - "8080:8080"   # 호스트포트:컨테이너포트 — 호스트에서 접근 가능
      - "5005:5005"   # 디버거 포트도 함께 열기

  db:
    image: postgres:16-alpine
    expose:
      - "5432"        # 같은 네트워크 컨테이너끼리만 접근, 호스트에는 노출 안 됨
```

`ports`는 호스트 머신에서도 접근할 수 있게 열어준다. `expose`는 같은 Docker 네트워크 내 다른 컨테이너에만 포트를 알려주는 메타데이터다 — 실제로 방화벽을 여는 게 아니라 선언적 문서 역할에 가깝다. DB는 앱 컨테이너만 접근하면 되므로 `expose`로 충분하다. 로컬에서 DB 클라이언트로 직접 접속하고 싶다면 `ports`로 열어야 한다.

#### environment와 env_file

```yaml
services:
  app:
    environment:
      # 키=값 방식
      - SPRING_PROFILES_ACTIVE=local
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/myapp
      # 맵 방식 (가독성이 더 좋음)
    # 또는:
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/myapp
      LOG_LEVEL: DEBUG

    # .env 파일에서 읽기
    env_file:
      - .env
      - .env.local    # 개발자별 로컬 오버라이드
```

`.env` 파일은 Compose 파일 내에서 `${VARIABLE}` 형태로 변수 치환에도 쓰인다.

```bash
# .env 파일 예시
POSTGRES_VERSION=16-alpine
APP_PORT=8080
DB_PASSWORD=localdev_secret
```

```yaml
# compose.yaml에서 .env 변수 치환
services:
  db:
    image: postgres:${POSTGRES_VERSION}  # postgres:16-alpine으로 치환됨
```

#### depends_on과 헬스체크

단순히 컨테이너 시작 순서만 보장하는 `depends_on`은 DB가 실제로 준비됐는지 보장하지 않는다. 컨테이너가 시작됐다는 것과 DB가 연결을 받을 준비가 됐다는 것은 다르다. PostgreSQL은 컨테이너 시작 후 내부 초기화가 완료되기까지 수 초가 걸리는데, 그 사이에 앱이 연결을 시도하면 "Connection refused" 오류로 실패한다. 헬스체크와 함께 써야 한다.

```yaml
services:
  app:
    build: .
    depends_on:
      db:
        condition: service_healthy   # 헬스체크 통과 후 시작
      redis:
        condition: service_started   # 단순 시작 확인 (기본값)

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app} -d ${POSTGRES_DB:-myapp}"]
      interval: 5s       # 5초마다 체크
      timeout: 5s        # 타임아웃
      retries: 5         # 5번 실패하면 unhealthy
      start_period: 10s  # 컨테이너 시작 후 10초는 실패 무시
```

`condition: service_healthy`를 쓰려면 대상 서비스에 반드시 `healthcheck`가 정의되어 있어야 한다. 없으면 오류가 난다.

### 1-4. 전형적인 웹 앱 구성 — app + DB + Redis

실제 프로젝트에서 가장 많이 쓰는 세 가지 서비스 조합 전체를 보여준다.

```yaml
# compose.yaml

services:
  # ── 애플리케이션 서버 ──────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
      - "5005:5005"   # 원격 디버거
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/myapp
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      # 소스 코드 실시간 반영 (핫 리로드용)
      - ./src:/workspace/src

  # ── 데이터베이스 ───────────────────────────────────────
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"   # 로컬 DB 클라이언트(DBeaver 등) 접속용
    volumes:
      - postgres-data:/var/lib/postgresql/data    # 데이터 영속성
      - ./db/init:/docker-entrypoint-initdb.d     # 초기화 SQL
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── 캐시 ───────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes    # AOF 영속성 활성화
    volumes:
      - redis-data:/data

# ── Named Volume 선언 ──────────────────────────────────
volumes:
  postgres-data:
  redis-data:
```

이 구성 하나면 팀원 누구든 `git clone` 후 `docker compose up`으로 동일한 개발 환경을 얻는다.

---

## 2. 컨테이너 네트워크 — 서비스 간 통신

### 2-1. Docker 네트워크 드라이버 종류

Docker는 여러 네트워크 드라이버를 제공한다. 로컬 개발에서는 대부분 bridge만 쓰지만, 전체 그림을 알아야 한다.

| 드라이버 | 범위 | 주요 용도 |
|----------|------|----------|
| `bridge` | 단일 호스트 | 기본값. 같은 호스트 컨테이너 간 통신 |
| `host` | 단일 호스트 | 컨테이너가 호스트 네트워크를 직접 사용 (Linux 전용) |
| `none` | 단일 호스트 | 네트워크 완전 격리 |
| `overlay` | 멀티 호스트 | Docker Swarm, 여러 호스트 간 컨테이너 통신 |
| `macvlan` | 단일 호스트 | 컨테이너에 물리 MAC 주소 부여 |

로컬 개발은 `bridge`, 프로덕션 Swarm 클러스터는 `overlay`가 기본이다.

### 2-2. Compose 기본 네트워크

별도로 `networks`를 정의하지 않으면 Compose가 자동으로 기본 네트워크를 만든다. 이름은 `{프로젝트명}_default`다.

```bash
# 프로젝트명은 compose.yaml이 있는 디렉터리명 (또는 -p 옵션)
# myapp 디렉터리에서 실행하면:
$ docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
a1b2c3d4e5f6   myapp_default     bridge    local
```

같은 `_default` 네트워크에 속한 모든 컨테이너는 서로 통신할 수 있다. 그리고 서비스명이 DNS 이름으로 자동 등록된다.

```
app 컨테이너 → db:5432 로 연결 가능
app 컨테이너 → redis:6379 로 연결 가능
db 컨테이너 → app:8080 으로 연결 가능
```

이게 `SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/myapp`처럼 서비스명을 호스트명으로 쓸 수 있는 이유다. Docker의 내장 DNS가 서비스명을 컨테이너 IP로 해석해준다.

### 2-3. 커스텀 네트워크로 서비스 격리

기본 네트워크에 모든 서비스를 넣으면 서비스끼리 불필요하게 서로를 볼 수 있다. 예를 들어 프론트엔드 컨테이너가 DB에 직접 접근하는 것을 막고 싶다면 네트워크를 나눠야 한다.

```yaml
services:
  # 프론트엔드 — frontend 네트워크만 접근
  frontend:
    image: nginx:alpine
    networks:
      - frontend

  # API 서버 — frontend와 backend 모두 접근 (브릿지 역할)
  app:
    build: .
    networks:
      - frontend
      - backend

  # DB — backend 네트워크만, 프론트엔드에서 직접 접근 불가
  db:
    image: postgres:16-alpine
    networks:
      - backend

  # 캐시 — backend 네트워크만
  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    # 내부 전용 (호스트에서 접근 불가)
    internal: true
```

`internal: true`를 설정하면 해당 네트워크는 외부 인터넷 접근이 차단된다. DB나 캐시처럼 외부와 통신할 필요가 없는 서비스에 적합하다.

### 2-4. 네트워크 별칭 (aliases)

같은 서비스를 여러 이름으로 접근해야 할 때는 `aliases`를 쓴다.

```yaml
services:
  db:
    image: postgres:16-alpine
    networks:
      backend:
        aliases:
          - database     # db 또는 database 둘 다 접근 가능
          - postgres-primary

networks:
  backend:
```

### 2-5. 외부 네트워크 연결

이미 실행 중인 다른 Compose 프로젝트의 네트워크에 연결할 수 있다.

```yaml
networks:
  # 이미 존재하는 외부 네트워크 사용
  shared-infra:
    external: true
    name: infra_default    # 외부 네트워크의 실제 이름
```

마이크로서비스처럼 여러 Compose 파일로 나뉜 환경에서 공통 인프라(DB, 메시지 큐)를 공유할 때 유용하다.

---

## 3. 볼륨과 데이터 영속성

### 3-1. 세 가지 볼륨 유형

컨테이너는 기본적으로 stateless다. 컨테이너를 삭제하면 내부 데이터도 사라진다. 데이터를 유지하려면 볼륨이 필요하다.

| 유형 | 선언 방식 | 관리 주체 | 성능 | 이식성 |
|------|-----------|-----------|------|--------|
| Named Volume | `volumes:` 키 선언 | Docker | 높음 | 높음 (Docker가 관리) |
| Bind Mount | 호스트 경로 직접 지정 | 호스트 파일시스템 | OS 의존 | 낮음 (경로 의존) |
| tmpfs | 메모리에 마운트 | 메모리 | 매우 높음 | 없음 (비영속) |

### 3-2. Named Volume — DB 데이터 보존

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      # named volume: postgres-data라는 이름의 볼륨을 컨테이너 경로에 마운트
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

# 최상위에서 볼륨 선언
volumes:
  postgres-data:
    # driver: local  (기본값, 생략 가능)
  redis-data:
```

Named Volume은 `docker volume ls`로 확인하고 `docker volume inspect`로 실제 저장 경로를 볼 수 있다.

```bash
$ docker volume ls
DRIVER    VOLUME NAME
local     myapp_postgres-data
local     myapp_redis-data

$ docker volume inspect myapp_postgres-data
[
    {
        "Name": "myapp_postgres-data",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/myapp_postgres-data/_data",
        ...
    }
]
```

### 3-3. Bind Mount — 소스 코드 실시간 반영

개발 중에는 소스 코드 변경이 즉시 컨테이너에 반영돼야 한다. Bind Mount를 쓰면 호스트의 디렉터리가 컨테이너 내부에 그대로 마운트된다.

```yaml
services:
  app:
    build: .
    volumes:
      # 짧은 문법: 호스트경로:컨테이너경로
      - ./src:/app/src

      # 긴 문법 (옵션 지정 가능)
      - type: bind
        source: ./src
        target: /app/src
        read_only: false    # 컨테이너에서 쓰기 허용 여부

      # node_modules는 호스트 것을 쓰지 않고 컨테이너 것을 유지
      # macOS 호스트에서 npm install한 node_modules에는 macOS용 네이티브 바이너리가
      # 포함될 수 있는데, 이를 Linux 컨테이너에서 실행하면 오류가 난다.
      # 익명 볼륨으로 마스킹하면 호스트 node_modules가 컨테이너 경로를 덮어쓰지 않는다.
      - /app/node_modules   # 익명 볼륨으로 마스킹
```

**주의:** macOS에서 Bind Mount는 성능이 느릴 수 있다. Docker Desktop의 VirtioFS를 사용하거나 `compose watch` (아래에서 설명)를 쓰는 것이 더 빠른 대안이다.

### 3-4. tmpfs — 임시 데이터, 민감 정보

```yaml
services:
  app:
    image: myapp:latest
    tmpfs:
      - /tmp             # 짧은 문법
      - /run/secrets     # 민감 정보를 메모리에만 유지

    # 긴 문법으로 옵션 지정
    volumes:
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100m     # 최대 100MB
```

tmpfs는 컨테이너가 중지되면 데이터가 사라진다. 테스트용 임시 파일, 세션 토큰 등 디스크에 남기면 안 되는 데이터에 적합하다.

### 3-5. 볼륨 선택 기준 정리

```
어떤 볼륨을 써야 할까?

DB 데이터, 영속적인 파일
  → Named Volume

소스 코드, 설정 파일, 실시간 반영 필요
  → Bind Mount

임시 파일, 빠른 I/O, 민감 정보
  → tmpfs
```

---

## 4. 로컬 개발 환경 구성 실전

### 4-1. compose.override.yaml — 개발용 오버라이드

`docker compose up`을 실행하면 Compose는 자동으로 `compose.yaml`과 `compose.override.yaml`을 병합한다. 이 특성을 이용해 기본 정의와 개발 전용 설정을 분리한다.

```yaml
# compose.yaml — 기본 정의 (팀 공통, Git에 커밋)
services:
  app:
    build: .
    environment:
      SPRING_PROFILES_ACTIVE: local

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine

volumes:
  postgres-data:
```

```yaml
# compose.override.yaml — 개발 전용 (Git에 커밋, 선택적으로 .gitignore 처리)
services:
  app:
    ports:
      - "8080:8080"
      - "5005:5005"    # 디버거 포트
    volumes:
      - ./src:/app/src # 소스 코드 실시간 마운트
    environment:
      JAVA_TOOL_OPTIONS: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"

  db:
    ports:
      - "5432:5432"    # 로컬에서 DB 클라이언트로 직접 접속

  redis:
    ports:
      - "6379:6379"
```

`compose.override.yaml`은 자동으로 병합되므로 별도 `-f` 플래그 없이 `docker compose up`만 치면 된다.

### 4-2. 프로덕션 설정 분리

프로덕션 환경에서는 오버라이드 없이 기본 파일만, 또는 별도 파일을 명시적으로 지정한다.

```yaml
# compose.prod.yaml — 프로덕션 전용
services:
  app:
    image: registry.example.com/myapp:${IMAGE_TAG}  # 빌드 대신 이미지 직접 사용
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512m
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

```bash
# 프로덕션 실행: 오버라이드 파일 없이 명시적으로 지정
$ docker compose -f compose.yaml -f compose.prod.yaml up -d

# 개발 실행: 자동으로 compose.override.yaml 병합
$ docker compose up
```

### 4-3. 핫 리로드 설정

#### Spring Boot + DevTools

```yaml
# compose.override.yaml
services:
  app:
    environment:
      # Spring DevTools 활성화
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
      SPRING_DEVTOOLS_LIVERELOAD_ENABLED: "true"
    volumes:
      - ./build/classes:/app/classes  # 컴파일된 클래스 파일 마운트
```

```bash
# 호스트에서 지속적 컴파일 (별도 터미널)
$ ./gradlew compileJava --continuous
```

#### Node.js + nodemon

```dockerfile
# Dockerfile.dev
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
# 소스는 볼륨으로 마운트 (COPY 안 함)
CMD ["npx", "nodemon", "--watch", "src", "src/index.js"]
```

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - ./src:/app/src
      - /app/node_modules    # 컨테이너의 node_modules 보존
    ports:
      - "3000:3000"
```

#### Python + Flask/FastAPI

```yaml
services:
  api:
    build: .
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - ./app:/app
    ports:
      - "8000:8000"
```

### 4-4. 원격 디버거 연결

#### JVM 원격 디버깅

```yaml
services:
  app:
    environment:
      # JVM 원격 디버거 에이전트 설정
      JAVA_TOOL_OPTIONS: >-
        -agentlib:jdwp=transport=dt_socket,
        server=y,
        suspend=n,
        address=*:5005
    ports:
      - "5005:5005"    # 디버거 포트 노출
```

IntelliJ IDEA에서 **Run → Edit Configurations → Remote JVM Debug** 추가:
- Host: `localhost`
- Port: `5005`

#### Node.js 디버거

```yaml
services:
  api:
    command: node --inspect=0.0.0.0:9229 src/index.js
    ports:
      - "9229:9229"
```

VS Code `.vscode/launch.json`:

```json
{
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Docker: Attach to Node",
      "remoteRoot": "/app",
      "localRoot": "${workspaceFolder}",
      "port": 9229,
      "address": "localhost"
    }
  ]
}
```

### 4-5. 로그 확인과 트러블슈팅

```bash
# 전체 서비스 로그 스트리밍
$ docker compose logs -f

# 특정 서비스만
$ docker compose logs -f app

# 마지막 100줄만
$ docker compose logs --tail=100 app

# 타임스탬프 포함
$ docker compose logs -f -t app
```

```bash
# 컨테이너 상태 확인
$ docker compose ps

# 헬스체크 상태 포함
$ docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# 실행 중인 컨테이너에 명령 실행
$ docker compose exec db psql -U app -d myapp

# 새 컨테이너로 명령 실행 (서비스 실행 안 해도 됨)
$ docker compose run --rm app ./gradlew test
```

```bash
# 컨테이너 재시작 없이 환경 변수 확인
$ docker compose exec app env | grep SPRING

# 네트워크 확인
$ docker network inspect myapp_default
```

### 4-6. 테스트용 DB 분리

통합 테스트용 DB를 별도로 띄우면 개발 DB 데이터를 오염시키지 않는다.

```yaml
# compose.test.yaml
services:
  db-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"    # 개발 DB(5432)와 포트 충돌 방지
    tmpfs:
      - /var/lib/postgresql/data    # 테스트는 영속성 불필요, tmpfs로 빠르게
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 2s
      retries: 10
```

```bash
# 테스트용 DB만 실행
$ docker compose -f compose.test.yaml up -d db-test

# 통합 테스트 실행
$ SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5433/myapp_test \
  ./gradlew integrationTest

# 테스트 후 정리
$ docker compose -f compose.test.yaml down
```

### 4-7. 좋은 패턴 vs 나쁜 패턴

#### 나쁜 패턴 — 비밀번호 하드코딩, 볼륨 없음

```yaml
# 나쁜 예시
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: supersecretpassword123  # Git에 노출됨
    # volumes 없음 → docker compose down 하면 데이터 소멸
```

#### 좋은 패턴 — 환경 변수 분리, Named Volume

```yaml
# 좋은 예시
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}    # .env에서 주입
    volumes:
      - postgres-data:/var/lib/postgresql/data   # 데이터 보존

volumes:
  postgres-data:
```

```bash
# .env 파일 (Git에 커밋하지 않음 — .gitignore에 추가)
DB_PASSWORD=localdev_secret

# .env.example 파일 (Git에 커밋 — 팀원에게 가이드)
DB_PASSWORD=your_local_password_here
```

#### 나쁜 패턴 — 불필요한 루트 실행, latest 태그

```yaml
# 나쁜 예시
services:
  app:
    image: myapp:latest    # latest는 재현 불가, CI에서 다른 버전 가져올 수 있음
    # user 지정 없음 → root로 실행됨 (보안 위험)
```

#### 좋은 패턴 — 버전 고정, non-root 사용자

```yaml
# 좋은 예시
services:
  app:
    image: myapp:1.2.3     # 명시적 버전
    user: "1000:1000"      # non-root 실행
```

---

## 5. 프로필과 환경 분리

### 5-1. Compose Profiles — 선택적 서비스 실행

모든 서비스를 항상 띄울 필요는 없다. 모니터링 스택, 메일 서버 목업, 관리자 도구 같은 것은 필요할 때만 띄우고 싶다. `profiles`를 쓰면 선택적으로 서비스를 활성화할 수 있다.

```yaml
services:
  # 항상 실행되는 핵심 서비스 (profiles 없음)
  app:
    build: .
    ports:
      - "8080:8080"

  db:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine

  # monitoring 프로필로만 실행
  prometheus:
    image: prom/prometheus:latest
    profiles: ["monitoring"]
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    profiles: ["monitoring"]
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  # 메일 서버 목업 (개발 중 이메일 확인용)
  mailhog:
    image: mailhog/mailhog:latest
    profiles: ["mail"]
    ports:
      - "1025:1025"    # SMTP
      - "8025:8025"    # 웹 UI

  # 데이터베이스 관리 UI
  adminer:
    image: adminer:latest
    profiles: ["tools"]
    ports:
      - "8888:8080"
    depends_on:
      - db
```

```bash
# 핵심 서비스만 (profiles 없는 것만)
$ docker compose up -d

# 모니터링 스택 포함
$ docker compose --profile monitoring up -d

# 여러 프로필 동시 활성화
$ docker compose --profile monitoring --profile tools up -d

# 환경 변수로 프로필 지정
$ COMPOSE_PROFILES=monitoring,mail docker compose up -d

# 특정 프로필의 서비스 중단
$ docker compose --profile monitoring down
```

### 5-2. .env 파일 계층 구조

Compose는 여러 `.env` 파일 소스를 지원한다. 우선순위는 다음과 같다.

```
우선순위 높음
  1. 셸 환경 변수 (export로 설정)
  2. compose.yaml 내 environment 키
  3. env_file에 명시된 파일
  4. .env 파일 (자동 로드)
우선순위 낮음
```

```bash
# 프로젝트 루트 .env (자동 로드, 공통 기본값)
POSTGRES_VERSION=16-alpine
REDIS_VERSION=7-alpine
APP_PORT=8080
LOG_LEVEL=INFO

# .env.local (개발자별, .gitignore에 추가)
LOG_LEVEL=DEBUG
APP_PORT=9090    # 포트 충돌 시 개인 설정
```

```yaml
services:
  app:
    env_file:
      - .env           # 공통
      - .env.local     # 개인 오버라이드 (없으면 무시됨)
      # path 방식 (Compose 2.24+):
      # - path: .env.local
      #   required: false   # 파일 없어도 오류 안 남
```

### 5-3. docker compose config — 최종 설정 검증

여러 파일이 병합된 실제 최종 설정을 확인한다.

```bash
# 병합된 최종 compose 설정 출력
$ docker compose config

# 특정 서비스만 확인
$ docker compose config --services

# 볼륨 목록만
$ docker compose config --volumes

# 오버라이드 파일 포함
$ docker compose -f compose.yaml -f compose.prod.yaml config
```

변수 치환이 제대로 됐는지, 파일 병합이 의도대로 됐는지 배포 전에 반드시 확인하는 습관을 들이자.

### 5-4. docker compose watch — 파일 변경 감지 자동 동기화

Docker Compose 2.22+ (Docker Desktop 4.24+)에서 도입된 `watch` 명령은 Bind Mount보다 정교한 파일 동기화를 제공한다.

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    develop:
      watch:
        # 소스 변경 시 컨테이너 내부로 동기화 (재빌드 없음)
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - "**/*.test.ts"    # 테스트 파일 제외

        # 의존성 변경 시 이미지 재빌드
        - action: rebuild
          path: package.json

        # 설정 파일 변경 시 서비스 재시작
        - action: sync+restart
          path: ./config
          target: /app/config
```

```bash
# watch 모드로 실행
$ docker compose watch

# up과 watch를 함께
$ docker compose up --watch
```

`watch`의 세 가지 액션:

| 액션 | 동작 | 속도 | 용도 |
|------|------|------|------|
| `sync` | 파일만 복사 | 빠름 | 소스 코드 변경 |
| `rebuild` | 이미지 재빌드 + 컨테이너 재생성 | 느림 | 의존성 변경 |
| `sync+restart` | 파일 복사 후 컨테이너 재시작 | 중간 | 설정 파일 변경 |

macOS에서 Bind Mount 성능 문제의 대안으로 `sync` 액션이 특히 효과적이다.

### 5-5. 전체 환경 관리 명령어 정리

```bash
# ── 시작 / 중지 ──────────────────────────────────────────

# 백그라운드 실행
$ docker compose up -d

# 특정 서비스만 실행
$ docker compose up -d app redis

# 강제 재빌드 후 실행
$ docker compose up -d --build

# 서비스 중지 (컨테이너 삭제, 볼륨 유지)
$ docker compose down

# 볼륨까지 삭제 (데이터 초기화)
$ docker compose down -v

# 이미지까지 삭제
$ docker compose down --rmi all

# ── 개별 서비스 제어 ─────────────────────────────────────

# 서비스 재시작
$ docker compose restart app

# 서비스 확장 (같은 이미지로 N개 실행)
$ docker compose up -d --scale app=3

# ── 빌드 ─────────────────────────────────────────────────

# 이미지 빌드만 (실행 안 함)
$ docker compose build

# 캐시 없이 재빌드
$ docker compose build --no-cache app

# ── 정보 확인 ─────────────────────────────────────────────

# 실행 중인 서비스 상태
$ docker compose ps

# 서비스 내부에서 명령 실행
$ docker compose exec app bash

# 임시 컨테이너로 명령 실행
$ docker compose run --rm app ./gradlew clean test

# 리소스 사용량
$ docker compose stats
```

### 5-6. 흔한 실수와 해결법

#### restart 정책

```yaml
services:
  app:
    restart: unless-stopped   # 수동 중지 제외하고 항상 재시작
    # 옵션:
    # no            — 재시작 안 함 (기본값)
    # always        — 항상 재시작 (Docker 재시작 후에도)
    # on-failure    — 오류 종료 시만
    # unless-stopped — 수동 중지 제외하고 항상
```

개발 환경에서는 `restart: unless-stopped`, 프로덕션 단독 Docker 환경에서는 `restart: always`가 일반적이다. Compose를 K8s 앞 단계로만 쓴다면 `restart`는 의미가 없다.

#### depends_on이 충분하지 않은 경우

```yaml
# 나쁜 예시 — DB가 준비 안 됐는데 앱이 연결 시도
services:
  app:
    depends_on:
      - db    # DB 컨테이너가 시작됐다는 것만 보장, 실제 준비는 모름

# 좋은 예시 — 헬스체크로 실제 준비 상태 확인
services:
  app:
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5
```

그래도 앱 쪽에 재시도 로직(connection retry, backoff)을 두는 것이 더 견고하다. `depends_on`은 첫 연결 실패를 줄여주지만, 네트워크 순단 같은 이후의 장애는 막지 못한다.

#### 볼륨 권한 문제

```yaml
# 문제: 호스트 사용자(1000)와 컨테이너 사용자(root 또는 다른 UID)가 달라 파일 권한 충돌
services:
  app:
    volumes:
      - ./data:/app/data    # 권한 오류 발생 가능

# 해결 1: user 지정으로 UID 맞추기
services:
  app:
    user: "1000:1000"
    volumes:
      - ./data:/app/data

# 해결 2: Dockerfile에서 사용자 생성
```

```dockerfile
# Dockerfile
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser
USER appuser
```

---

## 마무리

Docker Compose는 로컬 개발 환경의 사실상 표준이다. `docker compose up` 한 줄로 DB, 캐시, 메시지 큐까지 동일한 환경이 올라오면, "내 로컬에서는 되는데"라는 말이 팀에서 사라진다.

이 편에서 다룬 핵심을 정리하면:

- **Compose 문법**: `version` 필드 불필요, `depends_on` + `healthcheck` 조합으로 시작 순서 보장
- **네트워크**: 서비스명이 DNS로 자동 등록, 커스텀 네트워크로 서비스 격리
- **볼륨**: DB는 Named Volume, 소스 코드는 Bind Mount, 임시 데이터는 tmpfs
- **환경 분리**: `compose.override.yaml`로 개발/프로덕션 설정 분리, `profiles`로 선택적 서비스 실행
- **검증**: `docker compose config`로 병합 결과 확인 후 배포

한 가지 한계를 짚어두자. Compose는 **단일 호스트**에서 동작한다. 여러 서버에 걸쳐 컨테이너를 오케스트레이션하고, 자동 스케일링과 자가 복구가 필요한 프로덕션 환경에서는 Kubernetes가 필요하다. 다음 편에서는 Kubernetes의 핵심 개념 — Pod, Deployment, Service, ConfigMap — 을 다룬다. Compose로 익힌 개념들이 K8s에서 어떻게 대응되는지 함께 볼 것이다.

---

## 참고 자료

1. **Docker Compose Specification** — [docs.docker.com/compose/compose-file](https://docs.docker.com/compose/compose-file/)
   Compose 파일 전체 스펙. `services`, `networks`, `volumes`의 모든 필드와 옵션을 공식 문서로 확인한다.

2. **Awesome Compose** — [github.com/docker/awesome-compose](https://github.com/docker/awesome-compose)
   Docker 공식이 관리하는 실전 예제 모음. Spring Boot + PostgreSQL, React + Node + MongoDB 등 다양한 스택의 완성된 Compose 파일을 참고할 수 있다.

3. **Docker 네트워크 공식 문서** — [docs.docker.com/network](https://docs.docker.com/network/)
   bridge, overlay, macvlan 등 네트워크 드라이버별 동작 원리와 사용 사례. 컨테이너 간 통신이 예상대로 안 될 때 반드시 읽어야 하는 문서다.

4. **Compose Watch 공식 가이드** — [docs.docker.com/compose/how-tos/file-watch](https://docs.docker.com/compose/how-tos/file-watch/)
   `docker compose watch`의 `sync`, `rebuild`, `sync+restart` 액션 상세 설명. macOS Bind Mount 성능 문제의 대안으로 적극 검토할 만하다.

5. **Production-ready Docker packaging** — [hynek.me/articles/docker-uv](https://hynek.me/articles/docker-uv/)
   단순 `docker compose up`을 넘어, 프로덕션 수준의 Dockerfile과 Compose 구성을 위한 실전 가이드. 캐시 레이어 최적화, non-root 실행, 헬스체크 설계 등을 깊이 다룬다.
