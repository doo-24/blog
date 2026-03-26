---
title: "[Docker & 컨테이너] 2편 — Dockerfile 실전: 잘 만든 이미지의 조건"
date: 2026-03-20T14:30:00+09:00
draft: false
tags: ["Docker", "Dockerfile", "멀티스테이지 빌드", "이미지 최적화", "컨테이너 보안", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "Dockerfile 핵심 명령어(FROM, RUN, COPY, ENTRYPOINT vs CMD), 레이어 캐시 최적화, 멀티스테이지 빌드로 이미지 크기 90% 줄이기, .dockerignore, 보안 베스트 프랙티스(non-root, 시크릿 관리)까지"
---

Dockerfile은 누구나 쓸 수 있지만, 잘 쓰는 건 다른 문제다.

처음 Docker를 접한 개발자가 작성하는 Dockerfile은 대개 이렇게 생겼다. `FROM ubuntu:latest`, `RUN apt-get install` 몇 줄, `COPY . /app`, `CMD python app.py`.

이미지는 빌드된다. 컨테이너도 실행된다. 그런데 이미지 크기가 1.2GB다. 빌드할 때마다 패키지를 새로 다운로드한다. root 권한으로 프로세스가 실행된다.

소스코드에 적어둔 API 키가 이미지 레이어에 그대로 남아 있다.

동작은 하지만, 프로덕션에 올릴 수 없는 이미지다.

잘 만든 이미지는 다르다. 크기가 작아 레지스트리 전송이 빠르고, 레이어 캐시를 최대한 활용해 빌드가 수십 초 안에 끝나며, 최소 권한으로 실행되고, 시크릿이 레이어에 노출되지 않는다. 이런 이미지는 우연히 만들어지지 않는다. Dockerfile 명령어의 동작 원리를 이해하고, 의도를 가지고 설계해야 나온다.

이 글에서는 Dockerfile의 핵심 명령어부터 레이어 캐시 전략, 멀티스테이지 빌드, 보안 베스트 프랙티스까지 실전 코드와 함께 살펴본다.

---

## 섹션 1: Dockerfile 핵심 명령어 완전 정리

### FROM: 모든 것의 시작

`FROM`은 베이스 이미지를 지정한다. 단순해 보이지만 이 한 줄이 최종 이미지 크기, 보안, 빌드 속도에 가장 큰 영향을 미친다.

#### 베이스 이미지 계열 비교

같은 언어 런타임도 어떤 베이스 이미지를 쓰느냐에 따라 이미지 크기가 극적으로 달라진다.

| 태그 계열 | 예시 | 크기 (압축 전) | 특징 |
|-----------|------|----------------|------|
| `ubuntu` / `debian` | `python:3.12` | 900MB+ | 개발 도구 포함, 가장 큰 크기 |
| `slim` | `python:3.12-slim` | 130MB | 불필요한 패키지 제거, 범용적 |
| `alpine` | `python:3.12-alpine` | 25MB | musl libc 기반, 최소 크기 |
| `distroless` | `gcr.io/distroless/python3` | 20MB | 셸/패키지 매니저 없음, 최고 보안 |

**alpine**은 크기가 작지만 musl libc를 사용하기 때문에 glibc에 의존하는 네이티브 확장 모듈(예: numpy, cryptography)을 컴파일할 때 문제가 생길 수 있다. 패키지 설치도 `apk`를 사용하며, 일부 패키지는 alpine 저장소에 없거나 버전이 다르다.

**distroless**는 구글이 만든 이미지로 애플리케이션 실행에 필요한 최소한만 포함한다. 셸이 없어서 `docker exec`로 접속해 디버깅하는 게 불가능하다. 보안 관점에서는 공격 표면이 가장 작지만, 디버깅이 어려워 운영 성숙도가 높은 팀에 적합하다.

```dockerfile
# 단계별 선택 가이드

# 개발 단계: 도구가 많이 필요할 때
FROM python:3.12-slim

# 프로덕션 범용: 안정적이고 작은 크기
FROM python:3.12-slim

# 최소 크기가 중요할 때 (musl 호환 확인 후)
FROM python:3.12-alpine

# 최고 보안이 필요할 때 (멀티스테이지 빌드의 최종 스테이지)
FROM gcr.io/distroless/python3-debian12
```

#### 버전 태그 고정: `latest` 사용 금지

`FROM ubuntu:latest`는 오늘 빌드한 이미지와 6개월 후 빌드한 이미지가 다를 수 있다는 뜻이다. CI/CD 파이프라인에서 어느 날 갑자기 빌드가 깨지거나, 동일한 Dockerfile로 만든 이미지가 서로 다른 동작을 보이는 원인이 된다.

```dockerfile
# 나쁜 예 - latest는 비결정적
FROM node:latest
FROM python:latest
FROM ubuntu:latest

# 좋은 예 - 버전을 명시하고 다이제스트로 고정
FROM node:20.11.0-slim
FROM python:3.12.2-slim

# 가장 엄격한 고정: 이미지 다이제스트 사용
# (이미지가 교체되어도 내용이 동일함을 보장)
FROM node:20.11.0-slim@sha256:a1b2c3d4e5f6...
```

실제 프로젝트에서는 major.minor.patch까지 명시하고, Dependabot이나 Renovate 같은 도구로 버전 업데이트를 자동화하는 방식이 현실적이다.

---

### RUN: 레이어를 이해하고 명령을 구성하라

`RUN`은 명령을 실행하고 그 결과를 새 레이어로 커밋한다. 레이어는 이미지의 기본 단위이며, 한 번 생성된 레이어는 변경할 수 없다. 이 특성을 이해하지 못하면 이미지가 비효율적으로 커진다.

#### 레이어 최소화: 명령 체이닝

```dockerfile
# 나쁜 예 - 레이어가 3개 생성되고, 캐시도 중간에 남음
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# 나쁜 예 - 캐시를 정리하지 않아 apt 캐시가 이미지에 포함됨
RUN apt-get update && apt-get install -y curl git

# 좋은 예 - 단일 레이어, --no-install-recommends로 추가 패키지 방지
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

`rm -rf /var/lib/apt/lists/*`가 핵심이다. `apt-get update`는 패키지 목록 캐시를 `/var/lib/apt/lists/`에 저장하는데, 이 캐시가 이미지 레이어에 그대로 포함되면 수십 MB가 낭비된다. **같은 `RUN` 명령 안에서** 캐시를 지워야만 해당 레이어에 캐시가 포함되지 않는다.

별도 `RUN`으로 캐시를 지우면 이미 이전 레이어에 캐시가 기록되어 있으므로 아무 효과가 없다.

#### Alpine에서의 패키지 설치

```dockerfile
FROM python:3.12-alpine

# alpine은 apk 패키지 매니저 사용
# --no-cache: 캐시 디렉터리 자체를 만들지 않음 (rm 불필요)
RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev \
    openssl-dev
```

alpine의 `apk`는 `--no-cache` 플래그가 있어 캐시 정리가 더 간단하다.

#### pip, npm, gradle 캐시 처리

```dockerfile
# Python: pip 캐시 비활성화
RUN pip install --no-cache-dir -r requirements.txt

# Node.js: npm 캐시 정리
RUN npm ci --production && npm cache clean --force

# 또는 pnpm 사용 시
RUN pnpm install --frozen-lockfile --prod
```

---

### COPY vs ADD: 단순함이 미덕

`ADD`는 `COPY`의 슈퍼셋처럼 보이지만 실제로는 부작용이 많다.

| 기능 | COPY | ADD |
|------|------|-----|
| 로컬 파일 복사 | O | O |
| URL에서 파일 다운로드 | X | O |
| tar 아카이브 자동 압축 해제 | X | O |
| 동작 예측 가능성 | 높음 | 낮음 |
| 권장 여부 | 권장 | 제한적 사용 |

```dockerfile
# 나쁜 예 - ADD를 단순 파일 복사에 사용
ADD ./app /app
ADD requirements.txt /app/

# 좋은 예 - 단순 복사는 COPY
COPY ./app /app
COPY requirements.txt /app/

# ADD가 유효한 유일한 경우: tar 자동 압축 해제가 필요할 때
ADD app-archive.tar.gz /app/
# 하지만 이 경우도 COPY + RUN tar로 명시적으로 처리하는 걸 권장
```

URL 다운로드는 `ADD` 대신 `RUN curl` 또는 `RUN wget`을 쓰는 게 낫다. 캐시 처리와 오류 핸들링을 더 명시적으로 제어할 수 있기 때문이다.

---

### WORKDIR: 절대 경로로 명확하게

```dockerfile
# 나쁜 예 - 암묵적인 경로, 어디서 실행되는지 불분명
RUN mkdir /app
RUN cd /app && npm install

# 나쁜 예 - RUN cd는 다음 RUN에서 유지되지 않음
RUN cd /app
RUN npm install  # 여전히 / 에서 실행됨!

# 좋은 예 - WORKDIR은 이후 모든 명령의 작업 디렉터리를 설정
WORKDIR /app
RUN npm install
```

`WORKDIR`은 디렉터리가 없으면 자동으로 생성하고, 이후 `RUN`, `COPY`, `CMD`, `ENTRYPOINT`, `ADD` 명령의 기준 경로가 된다. 항상 절대 경로를 사용하고, 여러 단계에서 경로가 바뀐다면 명시적으로 다시 선언한다.

---

### ENV vs ARG: 빌드 시점과 실행 시점의 차이

```dockerfile
# ARG: 빌드 시점에만 존재하는 변수
# docker build --build-arg APP_VERSION=1.2.3
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

# ENV: 빌드 시점 + 컨테이너 실행 시점 모두 존재
ENV NODE_ENV=production
ENV PORT=8080

# ARG로 받은 값을 ENV로 넘기기 (런타임에도 접근 필요할 때)
ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}
```

중요한 보안 원칙: **시크릿(비밀번호, API 키)을 ARG나 ENV에 절대 넣지 마라.** `docker history` 명령으로 ARG 값이 빌드 레이어에 노출될 수 있고, ENV는 `docker inspect`로 실행 중인 컨테이너에서 직접 조회된다. 시크릿 처리는 섹션 5에서 별도로 다룬다.

---

### EXPOSE: 문서화 목적

```dockerfile
# EXPOSE는 실제로 포트를 열지 않는다 - 문서화용
EXPOSE 8080
EXPOSE 8443

# 실제 포트 바인딩은 docker run -p 옵션으로
# docker run -p 80:8080 myapp
```

`EXPOSE`는 이미지를 사용하는 사람에게 "이 컨테이너는 이 포트를 사용합니다"라고 알리는 메타데이터다. 건물 안내도에 "이 방은 출입구가 여기 있습니다"라고 표시하는 것과 같다. 안내도에 표시한다고 문이 열리는 건 아니다. `docker run -P` (대문자 P)를 쓸 때 랜덤 포트로 매핑되는 포트를 결정하는 역할도 한다. 실제 방화벽 규칙이나 포트 바인딩에는 영향이 없다.

---

## 섹션 2: ENTRYPOINT vs CMD — 정확한 구분

가장 많이 혼동되는 두 명령어다. 잘못 사용하면 컨테이너가 이상하게 동작하거나 시그널 처리가 깨진다.

### 각각의 역할

| 명령어 | 역할 | docker run 인수로 덮어쓰기 |
|--------|------|--------------------------|
| `CMD` | 기본 실행 명령 또는 ENTRYPOINT의 기본 인수 | 가능 |
| `ENTRYPOINT` | 컨테이너의 고정 실행 명령 | `--entrypoint` 플래그로만 가능 |

```dockerfile
# CMD만 사용: 기본 명령을 실행하되 덮어쓰기 가능
CMD ["python", "app.py"]
# docker run myimage → python app.py 실행
# docker run myimage bash → bash 실행 (CMD 덮어쓰기)

# ENTRYPOINT만 사용: 항상 이 명령 실행
ENTRYPOINT ["python"]
# docker run myimage → python 실행 (인수 없음, 오류 가능성)
# docker run myimage app.py → python app.py 실행

# ENTRYPOINT + CMD 조합: 고정 실행 파일에 기본 인수 제공
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage → python app.py 실행
# docker run myimage other.py → python other.py 실행 (CMD만 덮어쓰기)
```

### exec form vs shell form: PID 1 문제

이것이 진짜 중요한 차이다.

```dockerfile
# shell form: /bin/sh -c "java -jar app.jar" 로 실행
# sh가 PID 1, java는 sh의 자식 프로세스
CMD java -jar app.jar
ENTRYPOINT java -jar app.jar

# exec form: java 프로세스가 직접 PID 1
CMD ["java", "-jar", "app.jar"]
ENTRYPOINT ["java", "-jar", "app.jar"]
```

shell form은 `/bin/sh -c`를 통해 명령을 실행하므로 `sh`가 PID 1이 된다. 문제는 `docker stop`이 SIGTERM을 보낼 때 이 시그널이 PID 1(`sh`)에게 전달되지만, `sh`는 대부분 SIGTERM을 자식 프로세스에 전달하지 않는다는 점이다.

결과적으로 Java, Node.js, Python 등의 애플리케이션이 그레이스풀 셧다운을 수행하지 못하고 Docker가 타임아웃 후 강제로 SIGKILL을 보내게 된다. 진행 중인 요청이 끊기고, 데이터 손실이 발생할 수 있다.

```dockerfile
# 나쁜 예: shell form 사용으로 시그널이 앱에 전달되지 않음
FROM eclipse-temurin:21-jre-alpine
COPY app.jar /app.jar
CMD java -Xmx512m -jar /app.jar          # sh가 PID 1

# 좋은 예: exec form으로 앱이 직접 PID 1
FROM eclipse-temurin:21-jre-alpine
COPY app.jar /app.jar
ENTRYPOINT ["java", "-Xmx512m", "-jar", "/app.jar"]  # java가 PID 1
```

환경변수 치환이 필요하다면 exec form으로도 가능하다.

```dockerfile
# 환경변수를 사용하면서 exec form 유지
ENTRYPOINT ["sh", "-c", "java ${JVM_OPTS} -jar /app.jar"]

# 또는 시작 스크립트를 별도 파일로 분리 (권장)
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["java", "-jar", "/app.jar"]
```

### ENTRYPOINT + CMD 조합 패턴

실전에서 가장 많이 쓰이는 패턴들이다.

```dockerfile
# 패턴 1: 고정 바이너리 + 기본 인수
# docker run myapp --port 9090 → 포트만 변경
ENTRYPOINT ["./myapp"]
CMD ["--port", "8080", "--config", "/etc/myapp/config.yaml"]

# 패턴 2: 래퍼 스크립트로 초기화 후 앱 실행
# (환경변수 처리, DB 마이그레이션 등)
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["server"]

# docker-entrypoint.sh 예시
#!/bin/sh
set -e
# 초기화 작업
echo "Waiting for database..."
# exec "$@" 로 CMD를 exec로 실행 (PID 1 유지)
exec "$@"

# 패턴 3: 개발 도구 이미지 (특정 CLI 도구를 컨테이너화)
# docker run --rm mytool --help
ENTRYPOINT ["mytool"]
CMD ["--help"]
```

---

## 섹션 3: 레이어 캐시 최적화

빌드 속도는 개발 경험과 CI/CD 비용에 직접적으로 영향을 미친다. Docker의 레이어 캐시를 이해하고 활용하면 수 분짜리 빌드를 수십 초로 줄일 수 있다.

### Docker 빌드 캐시 동작 원리

Docker는 각 명령어를 실행하기 전에 캐시에서 재사용 가능한 레이어가 있는지 확인한다. 캐시 히트 조건은 명령어 유형마다 다르다.

- `RUN`: 명령 문자열이 동일하고 이전 레이어가 캐시 히트인 경우
- `COPY` / `ADD`: 복사하는 파일의 내용(체크섬)이 동일한 경우
- 한 레이어라도 캐시 미스가 발생하면 그 이후 모든 레이어는 캐시를 무시하고 재실행

이 마지막 규칙이 핵심이다. **자주 변경되는 레이어를 뒤로 배치**해야 캐시 효율이 높아진다.

```
레이어 1: FROM → 거의 변경 없음 (캐시 유지)
레이어 2: RUN apt-get install → 가끔 변경 (캐시 유지)
레이어 3: COPY package.json → 의존성 변경 시만 (캐시 유지)
레이어 4: RUN npm install → package.json 변경 시만 (캐시 유지)
레이어 5: COPY . /app → 코드 변경마다 (캐시 미스, 이하 전부 재실행)
레이어 6: RUN npm run build → 항상 재실행
```

### Node.js 캐시 최적화

```dockerfile
# 나쁜 예 - 소스코드가 바뀔 때마다 npm install이 재실행됨
FROM node:20-slim
WORKDIR /app
COPY . /app          # 소스 변경 → 캐시 미스
RUN npm install      # 매번 재실행 (수분 소요)
RUN npm run build

# 좋은 예 - 의존성 파일만 먼저 복사하여 npm install 캐시 활용
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./   # 의존성 파일만
RUN npm ci --production                   # 캐시 히트 (의존성 변경 없으면)
COPY . .                                  # 소스 복사
RUN npm run build
```

`npm install` 대신 `npm ci`를 쓰는 이유: `npm ci`는 `package-lock.json`을 기준으로 정확히 재현 가능한 설치를 보장하며, CI 환경에 최적화되어 있다.

### Python 캐시 최적화

```dockerfile
# 나쁜 예
FROM python:3.12-slim
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt    # 코드 변경마다 재실행

# 좋은 예
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./               # requirements만 먼저
RUN pip install --no-cache-dir -r requirements.txt
COPY . .                               # 이후 소스 복사
```

더 정밀하게 관리하고 싶다면 의존성을 분리한다.

```dockerfile
# requirements를 역할별로 분리
COPY requirements-base.txt requirements-app.txt ./
RUN pip install --no-cache-dir -r requirements-base.txt  # 자주 안 바뀜
RUN pip install --no-cache-dir -r requirements-app.txt   # 가끔 바뀜
COPY . .
```

### Java/Gradle 캐시 최적화

```dockerfile
# 나쁜 예 - 소스 변경마다 모든 의존성을 다시 다운로드
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar

# 좋은 예 - 의존성 다운로드를 별도 레이어로 분리
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app

# 빌드 스크립트를 먼저 복사하여 의존성 캐시
COPY build.gradle settings.gradle gradlew ./
COPY gradle ./gradle/
# 의존성만 다운로드 (소스 없이)
RUN ./gradlew dependencies --no-daemon

# 소스 복사 후 실제 빌드
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test
```

### Maven 캐시 최적화

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app

# pom.xml 먼저 복사하여 의존성 캐시
COPY pom.xml ./
COPY .mvn .mvn/
COPY mvnw ./
RUN ./mvnw dependency:go-offline -B   # 의존성 오프라인 캐시

COPY src ./src
RUN ./mvnw package -DskipTests -B
```

### .dockerignore 필수 항목

`.dockerignore`는 `COPY . .` 시 제외할 파일을 지정한다. 이 파일이 없으면 `node_modules`(수백 MB), `.git` 디렉터리, 빌드 산출물 등이 전부 빌드 컨텍스트에 포함되어 빌드가 느려지고 레이어 캐시 효율이 떨어진다.

```plaintext
# .dockerignore

# 버전 관리
.git
.gitignore

# 의존성 (컨테이너 안에서 새로 설치)
node_modules/
vendor/
.venv/
__pycache__/
*.pyc

# 빌드 산출물
dist/
build/
target/
*.class
*.jar (빌드 결과 포함 시 주의)

# 개발 도구 설정
.idea/
.vscode/
*.iml

# 환경 설정 (보안)
.env
.env.local
.env.*.local
*.env

# OS 파일
.DS_Store
Thumbs.db

# 테스트 관련
coverage/
.nyc_output/
*.test.js
__tests__/

# Docker 파일 자체 (불필요)
Dockerfile
docker-compose.yml
.dockerignore

# 문서
README.md
docs/
```

`.dockerignore`를 잘 관리하면 빌드 컨텍스트 전송 시간이 크게 줄어든다. `node_modules`가 포함된 컨텍스트는 전송에만 수 분이 걸릴 수 있다.

---

## 섹션 4: 멀티스테이지 빌드

단일 스테이지 빌드의 가장 큰 문제는 빌드 도구가 최종 이미지에 포함된다는 점이다. Java 애플리케이션을 빌드하려면 JDK와 Gradle이 필요하지만, 실행하려면 JRE만 있으면 된다. Go 애플리케이션은 컴파일러가 필요하지만 실행 바이너리는 OS 의존성이 거의 없다.

멀티스테이지 빌드는 빌드 환경과 런타임 환경을 분리하여 최종 이미지에는 실행에 필요한 것만 포함시킨다.

```text
[단일 스테이지]                    [멀티스테이지]

FROM jdk:21                        Stage 1 (builder)
  + JDK (빌드 도구)                FROM jdk:21-jdk AS builder
  + Gradle                           COPY 소스코드
  + 소스코드                          RUN gradle build → app.jar
  + app.jar                                │
                                           │ COPY --from=builder app.jar
이미지: ~600MB                     Stage 2 (runtime)
                                   FROM jre:21-slim
                                     COPY app.jar
                                     CMD ["java", "-jar", "app.jar"]

                                   이미지: ~150MB (75% 감소)
```

### Java 멀티스테이지 빌드

```dockerfile
# ── 빌드 스테이지 ────────────────────────────────────────────
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app

# Gradle 래퍼와 빌드 스크립트 (의존성 캐시 레이어)
COPY build.gradle settings.gradle gradlew ./
COPY gradle ./gradle/
RUN ./gradlew dependencies --no-daemon

# 소스 빌드
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test

# ── 런타임 스테이지 ───────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# 빌드 스테이지의 결과물만 복사
COPY --from=builder /app/build/libs/*.jar app.jar

# JVM 튜닝 옵션 (환경변수로 재정의 가능)
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseContainerSupport"

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

JVM은 컨테이너 메모리 제한을 인식하는 `-XX:+UseContainerSupport` 플래그를 Java 10부터 지원한다. Java 11 이상에서는 기본 활성화되어 있지만, 명시적으로 선언하면 의도가 명확해진다.

### Node.js 멀티스테이지 빌드

```dockerfile
# ── 의존성 설치 스테이지 ──────────────────────────────────────
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production

# ── 빌드 스테이지 (TypeScript 컴파일 등) ─────────────────────
FROM node:20-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                         # devDependencies 포함
COPY . .
RUN npm run build                  # TypeScript 컴파일, 번들링 등

# ── 런타임 스테이지 ───────────────────────────────────────────
FROM node:20-slim AS runtime
WORKDIR /app

# 프로덕션 의존성만 복사
COPY --from=deps /app/node_modules ./node_modules
# 빌드 결과물만 복사
COPY --from=builder /app/dist ./dist
COPY package.json ./

ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Go 멀티스테이지 빌드

Go는 멀티스테이지 빌드의 효과가 가장 극적이다. 컴파일된 바이너리는 OS 의존성이 거의 없어 distroless나 scratch 이미지를 사용할 수 있다.

```dockerfile
# ── 빌드 스테이지 ────────────────────────────────────────────
FROM golang:1.22-alpine AS builder
WORKDIR /app

# 의존성 캐시
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# CGO 비활성화로 완전한 정적 바이너리 생성
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \   # 디버그 심볼 제거로 바이너리 크기 축소
    -o /app/server .

# ── 런타임 스테이지 ───────────────────────────────────────────
# distroless: 셸, 패키지 매니저 없음 (공격 표면 최소화)
FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

Go 바이너리 하나만 들어있는 최종 이미지 크기는 10~20MB 수준이다.

### Python 멀티스테이지 빌드

```dockerfile
# ── 빌드 스테이지 (컴파일이 필요한 패키지 처리) ──────────────
FROM python:3.12-slim AS builder
WORKDIR /app

# 빌드 도구 설치 (최종 이미지에는 불필요)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
# 가상환경에 설치하여 복사하기 쉽게 구성
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# ── 런타임 스테이지 ───────────────────────────────────────────
FROM python:3.12-slim
WORKDIR /app

# 런타임 의존성만 설치 (gcc 등 빌드 도구 불필요)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 && \
    rm -rf /var/lib/apt/lists/*

# 빌드된 가상환경만 복사
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY . .
CMD ["python", "app.py"]
```

### 이미지 크기 비교

실제 멀티스테이지 빌드 적용 전후 크기 변화다.

| 애플리케이션 | 단일 스테이지 | 멀티스테이지 | 감소율 |
|-------------|--------------|-------------|--------|
| Spring Boot (JDK 기반) | 680MB | 185MB | 73% |
| Spring Boot (JRE alpine) | 680MB | 95MB | 86% |
| Node.js TypeScript | 1.1GB | 180MB | 84% |
| Go 서버 | 800MB | 15MB | 98% |
| Python FastAPI | 1.0GB | 220MB | 78% |

특정 스테이지만 빌드하거나 디버깅할 때는 `--target` 플래그를 사용한다.

```bash
# builder 스테이지까지만 빌드 (디버깅용)
docker build --target builder -t myapp:debug .

# 특정 스테이지 결과물 확인
docker run --rm myapp:debug ls -la /app/build/libs/
```

---

## 섹션 5: 보안 베스트 프랙티스

컨테이너가 보안 경계가 되려면 Dockerfile 단계에서부터 보안을 고려해야 한다. 실행 시점의 권한, 시크릿 처리, 이미지 취약점이 모두 Dockerfile에서 결정된다.

### non-root 사용자 실행

컨테이너가 root로 실행되면 컨테이너 탈출 취약점이 발생했을 때 호스트 시스템까지 영향을 받을 수 있다. 가장 기본적이고 중요한 보안 설정이다.

```dockerfile
FROM node:20-slim
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --production

COPY . .

# 전용 사용자와 그룹 생성
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/false --create-home appuser

# 앱 파일 소유권 변경
RUN chown -R appuser:appgroup /app

# non-root 사용자로 전환
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

alpine 기반 이미지에서는 `addgroup`과 `adduser`를 사용한다.

```dockerfile
FROM node:20-alpine
WORKDIR /app

COPY --chown=node:node package.json package-lock.json ./
RUN npm ci --production

COPY --chown=node:node . .

# node 이미지는 이미 'node' 사용자를 제공함
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

`COPY --chown=user:group`을 쓰면 별도의 `RUN chown` 레이어 없이 파일 소유권을 설정할 수 있어 레이어 수를 줄일 수 있다.

실행 중인 컨테이너의 사용자 확인:

```bash
# 어떤 사용자로 실행 중인지 확인
docker exec mycontainer whoami
docker exec mycontainer id

# docker inspect로 확인
docker inspect mycontainer | grep -i user
```

### 시크릿 관리: BuildKit secret mount

API 키, 비밀번호, 인증서 같은 시크릿을 ARG나 ENV로 빌드에 넘기면 `docker history`에 그 값이 그대로 노출된다.

```bash
# 위험한 방식으로 빌드하면
docker build --build-arg GITHUB_TOKEN=ghp_xxxx .

# 이렇게 노출됨
docker history myimage
IMAGE          CREATED          CREATED BY                                      SIZE
abc123         2 minutes ago    |1 GITHUB_TOKEN=ghp_xxxx /bin/sh -c pip in...  45.2MB
```

BuildKit의 `--secret` 기능을 사용하면 시크릿이 레이어에 기록되지 않는다.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
WORKDIR /app

# --secret으로 마운트된 시크릿은 레이어에 저장되지 않음
# 해당 RUN 명령 실행 중에만 /run/secrets/에서 접근 가능
RUN --mount=type=secret,id=pip_token \
    pip install --no-cache-dir \
    --index-url https://$(cat /run/secrets/pip_token)@private.registry.example.com/simple/ \
    -r requirements.txt

COPY . .
CMD ["python", "app.py"]
```

빌드 시 시크릿을 파일로 전달한다.

```bash
# 환경변수에서 시크릿 파일 생성 후 전달
echo "$PIP_TOKEN" > /tmp/pip_token
docker build --secret id=pip_token,src=/tmp/pip_token .

# 또는 직접 환경변수에서
docker build --secret id=pip_token,env=PIP_TOKEN .
```

SSH 인증이 필요한 경우 (private git repository 클론 등):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-slim AS builder
WORKDIR /app

# SSH 소켓 마운트 (키가 이미지에 저장되지 않음)
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git && \
    cd private-repo && npm ci

COPY . .
RUN npm run build
```

```bash
# SSH 에이전트 포워딩으로 빌드
docker build --ssh default=$SSH_AUTH_SOCK .
```

### HEALTHCHECK 설정

HEALTHCHECK를 설정하면 Docker와 오케스트레이터(Kubernetes, ECS 등)가 컨테이너의 실제 헬스 상태를 파악할 수 있다.

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --production

# HTTP 헬스체크
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=60s \
            --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "server.js"]
```

옵션 설명:
- `--interval`: 헬스체크 실행 간격 (기본 30s)
- `--timeout`: 이 시간 내에 응답 없으면 실패 (기본 30s)
- `--start-period`: 초기 시작 시간 유예 기간 (이 동안의 실패는 카운트 안 함)
- `--retries`: 이 횟수 연속 실패 시 unhealthy 상태

`curl`이 없는 이미지(distroless, alpine minimal)에서는 wget이나 자체 헬스체크 바이너리를 사용한다.

```dockerfile
# wget 사용 (alpine)
HEALTHCHECK CMD wget -qO- http://localhost:8080/health || exit 1

# 자체 Go 헬스체크 바이너리 (distroless)
COPY --from=builder /app/healthcheck /healthcheck
HEALTHCHECK CMD ["/healthcheck"]
```

### 읽기 전용 파일시스템

런타임에서 파일시스템을 읽기 전용으로 설정하면 컨테이너가 침해되더라도 파일 변조를 방지할 수 있다.

```bash
# 실행 시 읽기 전용 파일시스템
docker run --read-only \
    --tmpfs /tmp \                 # 임시 파일만 tmpfs로
    --tmpfs /var/run \
    myapp
```

Dockerfile 수준에서는 직접 설정할 수 없지만, 애플리케이션이 임시 파일을 쓸 위치를 `/tmp`로 통일하도록 설계해야 한다.

```dockerfile
# 로그, 임시 파일 경로를 환경변수로 분리
ENV LOG_DIR=/tmp/logs
ENV TEMP_DIR=/tmp/work
RUN mkdir -p /tmp/logs /tmp/work
```

### 이미지 취약점 스캔

빌드된 이미지에 알려진 CVE 취약점이 있는 패키지가 포함되어 있는지 주기적으로 스캔해야 한다.

```bash
# Trivy 설치 (macOS)
brew install trivy

# 이미지 스캔
trivy image myapp:latest

# 심각도 필터링 (CRITICAL, HIGH만)
trivy image --severity CRITICAL,HIGH myapp:latest

# CI/CD에서 사용 (취약점 발견 시 빌드 실패)
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# 출력 예시
myapp:latest (debian 12.4)
=========================
Total: 3 (HIGH: 2, CRITICAL: 1)

┌──────────────┬────────────────┬──────────┬───────────────────┬──────────────────┬──────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Installed Version │  Fixed Version   │                Title                 │
├──────────────┼────────────────┼──────────┼───────────────────┼──────────────────┼──────────────────────────────────────┤
│ libssl3      │ CVE-2024-0727  │ HIGH     │ 3.0.11-1~deb12u2  │ 3.0.13-1~deb12u1 │ openssl: PKCS12 Decoding crashes     │
└──────────────┴────────────────┴──────────┴───────────────────┴──────────────────┴──────────────────────────────────────┘
```

CI/CD 파이프라인에 스캔을 통합하는 예시:

```yaml
# GitHub Actions
- name: 이미지 빌드
  run: docker build -t myapp:${{ github.sha }} .

- name: Trivy 취약점 스캔
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

### Hadolint로 Dockerfile 린팅

Dockerfile 작성 실수를 자동으로 잡아주는 린터다. `latest` 태그 사용, 체이닝 없는 `RUN`, `curl | bash` 같은 패턴을 경고해준다.

```bash
# 설치 (macOS)
brew install hadolint

# 린팅
hadolint Dockerfile

# 출력 예시
Dockerfile:3 DL3007 warning: Using latest is best avoided unless you are using a
Dockerfile:5 DL3008 warning: Pin versions in apt get install. Instead of...
Dockerfile:8 DL3059 info: Multiple consecutive `RUN` instructions. Consider...

# CI에서 사용
hadolint --failure-threshold warning Dockerfile
```

주요 규칙:
- `DL3007`: `latest` 태그 금지
- `DL3008`: apt-get 버전 고정
- `DL3009`: apt-get 캐시 정리
- `DL3059`: 연속된 `RUN` 병합 권장
- `SC2086`: 쉘 스크립트 변수 따옴표 처리

---

## 완성된 Dockerfile 예시: Spring Boot 프로덕션

지금까지 다룬 내용을 모두 적용한 실전 예시다.

```dockerfile
# syntax=docker/dockerfile:1
# Dockerfile for Spring Boot production image

# ── 1단계: 의존성 캐시 ────────────────────────────────────────
FROM eclipse-temurin:21-jdk AS dependencies
WORKDIR /app

# Gradle 래퍼만 먼저 복사 (캐시 최적화)
COPY gradlew build.gradle settings.gradle ./
COPY gradle ./gradle/

# 의존성 사전 다운로드 (소스 변경과 분리)
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew dependencies --no-daemon

# ── 2단계: 빌드 ──────────────────────────────────────────────
FROM dependencies AS builder

COPY src ./src

RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew bootJar --no-daemon -x test

# ── 3단계: 런타임 ─────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# 보안: 전용 사용자 생성
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser

# 빌드 결과물만 복사 (소유권 동시 설정)
COPY --from=builder --chown=appuser:appgroup \
    /app/build/libs/*.jar app.jar

# 메타데이터
LABEL maintainer="server-engineering-team" \
      version="1.0" \
      description="Spring Boot application"

# 환경 설정
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
ENV SPRING_PROFILES_ACTIVE=production

# non-root 사용자로 전환
USER appuser

# 헬스체크
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=120s \
            --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## 완성된 Dockerfile 예시: Node.js API 서버

```dockerfile
# syntax=docker/dockerfile:1

# ── 1단계: 의존성 설치 ────────────────────────────────────────
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production && npm cache clean --force

# ── 2단계: 빌드 (TypeScript) ──────────────────────────────────
FROM node:20-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# ── 3단계: 런타임 ─────────────────────────────────────────────
FROM node:20-slim AS runtime
WORKDIR /app

# 보안 패치 적용
RUN apt-get update && \
    apt-get upgrade -y && \
    rm -rf /var/lib/apt/lists/*

# 필요한 파일만 복사
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./

ENV NODE_ENV=production
ENV PORT=3000

USER node

HEALTHCHECK --interval=20s \
            --timeout=5s \
            --start-period=30s \
            --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Dockerfile 품질 체크리스트

빌드 전 스스로 확인할 수 있는 항목들이다.

| 항목 | 확인 내용 | 우선순위 |
|------|-----------|----------|
| 베이스 이미지 | `latest` 태그 사용하지 않음 | 필수 |
| 베이스 이미지 | slim/alpine 등 최소 이미지 사용 | 권장 |
| RUN 체이닝 | 연속된 RUN을 &&로 병합 | 필수 |
| 패키지 캐시 | apt/apk/pip 캐시 정리 | 필수 |
| 레이어 순서 | 자주 변경되는 것을 뒤에 배치 | 권장 |
| .dockerignore | node_modules, .git 등 제외 | 필수 |
| COPY vs ADD | 단순 복사에 COPY 사용 | 권장 |
| exec form | CMD/ENTRYPOINT에 exec form 사용 | 필수 |
| 멀티스테이지 | 빌드/런타임 스테이지 분리 | 권장 |
| non-root | 전용 사용자로 실행 | 필수 |
| 시크릿 처리 | ARG/ENV에 시크릿 없음 | 필수 |
| HEALTHCHECK | 헬스체크 정의 | 권장 |
| 취약점 스캔 | Trivy/Snyk으로 주기적 스캔 | 권장 |
| Hadolint | 린팅 통과 | 권장 |

---

## 마무리

잘 만든 Dockerfile은 세 가지 조건을 충족한다.

**작은 이미지**: 멀티스테이지 빌드와 최소 베이스 이미지로 이미지 크기를 최소화한다. 이미지가 작으면 레지스트리 전송이 빠르고, 공격 표면이 줄어들며, 보안 스캔 대상 패키지 수가 적어진다.

**빠른 빌드**: 레이어 캐시를 이해하고 자주 변경되는 레이어를 뒤에 배치한다. 의존성 설치와 소스 복사를 분리하면 코드 변경 시 수 분 걸리던 빌드가 수십 초로 줄어든다.

**안전한 실행**: non-root 사용자로 실행하고, 시크릿을 레이어에 남기지 않으며, 취약점 스캔을 CI/CD에 통합한다. 컨테이너 보안의 첫 번째 방어선은 Dockerfile이다.

이 세 가지는 서로 상충하지 않는다. 오히려 좋은 Dockerfile은 세 가지를 동시에 달성한다. 오늘 당장 기존 Dockerfile에 Hadolint를 돌려보는 것부터 시작해보자. 생각보다 많은 개선 포인트를 발견하게 될 것이다.

다음 편에서는 Docker Compose를 다룬다. 여러 컨테이너를 조합해 로컬 개발 환경을 구성하고, 서비스 간 네트워크와 볼륨을 관리하는 실전 패턴을 살펴본다.

---

## 참고 자료

1. **Docker 공식 Dockerfile 베스트 프랙티스**
   [https://docs.docker.com/build/building/best-practices/](https://docs.docker.com/build/building/best-practices/)
   Dockerfile 작성 공식 가이드. 레이어 최적화, 캐시 활용, 멀티스테이지 빌드의 원칙을 다룬다.

2. **Google의 distroless 이미지**
   [https://github.com/GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless)
   셸과 패키지 매니저를 제거한 최소 런타임 이미지. Java, Python, Go, Node.js 등 언어별 이미지를 제공한다.

3. **Hadolint — Dockerfile 린터**
   [https://github.com/hadolint/hadolint](https://github.com/hadolint/hadolint)
   ShellCheck를 내장한 Dockerfile 린터. 일반적인 안티패턴과 쉘 스크립트 오류를 자동으로 감지한다.

4. **BuildKit 공식 문서**
   [https://docs.docker.com/build/buildkit/](https://docs.docker.com/build/buildkit/)
   BuildKit의 캐시 마운트, 시크릿 마운트, SSH 포워딩 등 고급 기능을 설명한다. `RUN --mount=type=cache`와 `RUN --mount=type=secret` 사용법이 상세히 나온다.

5. **Trivy — 컨테이너 이미지 취약점 스캐너**
   [https://github.com/aquasecurity/trivy](https://github.com/aquasecurity/trivy)
   OS 패키지와 언어별 라이브러리 취약점을 스캔하는 오픈소스 도구. CI/CD 통합이 쉽고 GitHub Actions 액션도 제공한다.
