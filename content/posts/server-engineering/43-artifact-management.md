---
title: "[배포와 CI/CD] 3편 — 아티팩트 관리: 빌드 산출물의 생명주기"
date: 2026-03-17T20:05:00+09:00
draft: false
tags: ["아티팩트", "Nexus", "Docker Registry", "SemVer", "서버"]
series: ["배포와 CICD"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 6
summary: "아티팩트 저장소 역할(Nexus, JFrog Artifactory), Maven/Gradle 아티팩트 배포와 SemVer 버저닝, 컨테이너 레지스트리(ECR, Docker Hub, Harbor), 보존 정책과 취약점 스캔 자동화까지"
---

빌드 파이프라인이 성공적으로 완료됐다. 테스트를 통과했고, 컴파일도 끝났다.

그런데 정작 가장 중요한 질문에는 아직 답하지 못한 경우가 많다. "이 빌드 결과물을 어디에, 어떻게 저장할 것인가?"

단순히 파일 서버에 올리거나 S3 버킷에 던져 두는 것은 해결책이 아니다.

6개월 뒤에 특정 버전으로 롤백해야 할 때, 어떤 커밋에서 빌드된 것인지 추적해야 할 때, 또는 의존성 라이브러리에 CVE 취약점이 발견됐을 때 — 이 모든 상황에서 아티팩트 관리 전략의 수준이 드러난다.

아티팩트 저장소는 단순한 파일 보관함이 아니라, 배포 생명주기 전체를 지탱하는 핵심 인프라다.

---

## 아티팩트 저장소란 무엇인가

아티팩트(Artifact)는 빌드 프로세스가 생성한 모든 산출물을 통칭한다. JAR 파일, WAR 파일, npm 패키지, Python 휠, 도커 이미지, Helm 차트, 심지어 terraform 모듈까지 — 배포에 필요한 모든 바이너리와 패키지가 여기에 해당한다.

아티팩트 저장소는 이런 산출물을 체계적으로 관리하기 위한 특화된 저장 시스템이다.

일반 파일 저장소와 다른 점은 다음과 같다.

- **메타데이터 관리**: 버전, 빌드 시간, 커밋 해시, 빌드 환경 정보를 함께 저장한다
- **의존성 해석**: Maven/Gradle/npm 같은 패키지 매니저 프로토콜을 네이티브로 지원한다
- **접근 제어**: 저장소별, 패키지별 세밀한 권한 관리를 제공한다
- **프록시 기능**: 외부 레지스트리를 프록시하여 인터넷 의존성을 줄인다
- **보존 정책**: 오래된 아티팩트를 자동으로 정리하는 규칙을 지원한다

### Nexus Repository Manager

Sonatype Nexus는 엔터프라이즈 환경에서 가장 널리 쓰이는 오픈소스 아티팩트 저장소다.

Maven, npm, PyPI, Docker, Helm, Go, NuGet 등 40개 이상의 패키지 포맷을 단일 플랫폼에서 관리할 수 있다.

Nexus의 저장소 타입은 세 가지로 구분된다.

**Hosted Repository**: 내부적으로 빌드한 아티팩트를 업로드하는 저장소다. `releases`와 `snapshots`로 나누어 관리하는 것이 일반적이다.

**Proxy Repository**: Maven Central, Docker Hub, npm registry 같은 외부 저장소를 캐싱하는 저장소다. 빌드 시 외부에 직접 접근하지 않고 Nexus를 거치게 되므로, 외부 서비스 장애나 네트워크 이슈의 영향을 줄인다.

**Group Repository**: Hosted와 Proxy 저장소를 하나의 URL로 묶어 제공하는 논리 저장소다.

빌드 도구 설정을 단순하게 유지할 수 있다.

```yaml
# docker-compose.yml로 Nexus 빠르게 구동하기
version: '3.8'
services:
  nexus:
    image: sonatype/nexus3:3.65.0
    container_name: nexus
    ports:
      - "8081:8081"   # Web UI / Maven
      - "8082:8082"   # Docker hosted
      - "8083:8083"   # Docker proxy
    volumes:
      - nexus-data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx2g -XX:MaxDirectMemorySize=3g
    restart: unless-stopped

volumes:
  nexus-data:
    driver: local
```

### JFrog Artifactory

JFrog Artifactory는 Nexus와 함께 양대 산맥을 이루는 상용 아티팩트 저장소다.

오픈소스 버전(Artifactory OSS)도 있지만, 엔터프라이즈 기능은 유료다.

Nexus 대비 Artifactory의 차별점은 다음과 같다.

- **빌드 통합**: CI/CD 파이프라인과의 통합이 더 긴밀하다. 빌드별 의존성 그래프, 아티팩트 출처 추적(Build Info)을 자동으로 기록한다
- **Xray 연동**: 보안 스캔 도구인 JFrog Xray와 네이티브 연동으로 취약점 탐지가 간편하다
- **분산 저장**: 글로벌 팀을 위한 Edge Node와 복제 기능이 강력하다

```bash
# JFrog CLI를 활용한 아티팩트 배포
# CLI 설치
curl -fL https://install-cli.jfrog.io | sh

# 서버 설정
jf config add my-artifactory \
  --url=https://mycompany.jfrog.io \
  --user=ci-bot \
  --password=${ARTIFACTORY_TOKEN}

# Maven 아티팩트 배포
jf mvn deploy \
  --build-name=my-service \
  --build-number=${BUILD_NUMBER}

# 빌드 정보 게시 (추적성 확보)
jf rt build-publish my-service ${BUILD_NUMBER}
```

---

## Maven/Gradle 아티팩트 배포와 SemVer 버저닝

### Semantic Versioning 전략

아티팩트 버저닝의 사실상 표준은 Semantic Versioning(SemVer)다.

`MAJOR.MINOR.PATCH` 형식을 따르며, 각 숫자의 의미는 명확하다.

- **MAJOR**: 하위 호환이 깨지는 변경 (breaking change)
- **MINOR**: 하위 호환을 유지하는 새 기능 추가
- **PATCH**: 하위 호환을 유지하는 버그 수정

실무에서는 여기에 Pre-release 식별자와 Build metadata를 덧붙인다.

```
1.2.3-alpha.1+build.20260317
│ │ │  │    │  └── Build metadata (정렬 무시)
│ │ │  └────┘───── Pre-release 식별자
│ │ └─────────────── PATCH
│ └───────────────── MINOR
└─────────────────── MAJOR
```

**SNAPSHOT vs Release 분리 전략**

Maven 생태계에서는 개발 중인 버전을 `1.2.3-SNAPSHOT`으로, 최종 릴리스를 `1.2.3`으로 구분하는 관행이 정착해 있다.

SNAPSHOT은 같은 버전 식별자로 여러 번 덮어쓸 수 있고, Release는 한 번 게시하면 변경 불가(immutable)로 취급한다.

```
Release 저장소: releases/
  com/example/my-service/1.2.3/my-service-1.2.3.jar  (불변)

Snapshot 저장소: snapshots/
  com/example/my-service/1.2.4-SNAPSHOT/
    my-service-1.2.4-20260317.143022-1.jar  (타임스탬프 기반)
    my-service-1.2.4-20260317.151044-2.jar  (재빌드시 추가)
```

### Gradle 배포 설정

```kotlin
// build.gradle.kts
plugins {
    id("java")
    id("maven-publish")
    id("signing")
}

group = "com.example"
version = System.getenv("RELEASE_VERSION") ?: "1.0.0-SNAPSHOT"

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])

            // 소스와 Javadoc JAR 포함
            artifact(tasks.named("sourcesJar"))
            artifact(tasks.named("javadocJar"))

            pom {
                name.set("My Service")
                description.set("Core service library")
                url.set("https://github.com/example/my-service")

                licenses {
                    license {
                        name.set("Apache License 2.0")
                        url.set("https://www.apache.org/licenses/LICENSE-2.0")
                    }
                }

                scm {
                    connection.set("scm:git:git://github.com/example/my-service.git")
                    developerConnection.set("scm:git:ssh://github.com/example/my-service.git")
                    url.set("https://github.com/example/my-service")
                }
            }
        }
    }

    repositories {
        maven {
            name = "Nexus"
            // SNAPSHOT과 Release 저장소를 자동으로 분기
            url = if (version.toString().endsWith("SNAPSHOT")) {
                uri("https://nexus.example.com/repository/maven-snapshots/")
            } else {
                uri("https://nexus.example.com/repository/maven-releases/")
            }
            credentials {
                username = System.getenv("NEXUS_USERNAME")
                password = System.getenv("NEXUS_PASSWORD")
            }
        }
    }
}

// 릴리스 시 서명 활성화
signing {
    val signingKey: String? by project
    val signingPassword: String? by project
    useInMemoryPgpKeys(signingKey, signingPassword)
    sign(publishing.publications["mavenJava"])
}
```

### CI/CD 파이프라인에서 버전 자동화

버전을 수동으로 관리하면 실수가 발생한다.

Git 태그를 기반으로 버전을 자동 결정하는 방식이 안정적이다.

```yaml
# .github/workflows/publish.yml
name: Publish Artifact

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'    # v1.2.3 형태 태그
  push:
    branches:
      - main                         # main 브랜치 push는 SNAPSHOT

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 태그 히스토리 전체 필요

      - name: Determine Version
        id: version
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            VERSION="${GITHUB_REF_NAME#v}"
          else
            # main 브랜치: 마지막 태그 기반 SNAPSHOT
            LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
            BASE_VERSION="${LAST_TAG#v}"
            # PATCH 버전을 하나 올려서 SNAPSHOT
            NEXT_PATCH=$(echo "$BASE_VERSION" | awk -F. '{print $1"."$2"."$3+1}')
            VERSION="${NEXT_PATCH}-SNAPSHOT"
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Resolved version: ${VERSION}"

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Publish to Nexus
        env:
          RELEASE_VERSION: ${{ steps.version.outputs.version }}
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
        run: ./gradlew publish --no-daemon
```

### 안티패턴: 버저닝 함정들

**SNAPSHOT을 프로덕션에 배포하는 실수**

```kotlin
// 잘못된 예: 프로덕션 의존성에 SNAPSHOT 사용
dependencies {
    implementation("com.example:core-lib:2.1.0-SNAPSHOT")  // 절대 금지
}
```

SNAPSHOT은 같은 식별자로 내용이 바뀔 수 있다. 오늘 빌드한 것과 내일 빌드한 것이 달라진다.

프로덕션 의존성은 반드시 고정된 릴리스 버전을 사용해야 한다.

**버전 고정 없는 Latest 태그 의존**

```dockerfile
# 잘못된 예: latest 태그로 기반 이미지 지정
FROM openjdk:latest

# 올바른 예: 다이제스트나 정확한 버전 고정
FROM eclipse-temurin:21.0.2_13-jre-jammy@sha256:a1b2c3d4...
```

`latest` 태그는 언제든 바뀔 수 있다.

재현 가능한 빌드를 위해서는 정확한 버전을 고정해야 한다.

---

## 컨테이너 레지스트리

도커 이미지는 현대 배포의 핵심 아티팩트다.

컨테이너 레지스트리는 이미지를 저장하고 배포하는 특화된 저장소다.

### Amazon ECR (Elastic Container Registry)

AWS 환경에서 가장 자연스러운 선택이다.

IAM과 통합된 접근 제어, VPC 내부 트래픽, Lambda/ECS/EKS와의 네이티브 연동이 강점이다.

```bash
# ECR 레지스트리 로그인
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="ap-northeast-2"

aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# 이미지 태그 지정 및 푸시
IMAGE_NAME="my-service"
VERSION="1.2.3"
ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}"

docker build -t ${IMAGE_NAME}:${VERSION} .
docker tag ${IMAGE_NAME}:${VERSION} ${ECR_URI}:${VERSION}
docker tag ${IMAGE_NAME}:${VERSION} ${ECR_URI}:latest  # 선택적
docker push ${ECR_URI}:${VERSION}
```

ECR 레포지토리 생성과 라이프사이클 정책은 Terraform으로 코드화하는 것이 좋다.

```hcl
# terraform/ecr.tf
resource "aws_ecr_repository" "my_service" {
  name                 = "my-service"
  image_tag_mutability = "IMMUTABLE"  # 같은 태그로 덮어쓰기 방지

  image_scanning_configuration {
    scan_on_push = true  # 푸시 시 자동 취약점 스캔
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Environment = "production"
    Team        = "backend"
  }
}

# 이미지 보존 정책
resource "aws_ecr_lifecycle_policy" "my_service" {
  repository = aws_ecr_repository.my_service.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Release 태그는 30개까지 보존"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 30
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "untagged 이미지는 7일 후 삭제"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

### Docker Hub

공개 이미지 배포나 소규모 팀에서 가장 접근하기 쉬운 선택이다.

무료 플랜은 public 레포지토리 무제한, private 레포지토리 1개를 제공한다.

```yaml
# GitHub Actions에서 Docker Hub 푸시
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}  # 계정 비밀번호 대신 토큰 사용

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      myorg/my-service:${{ github.ref_name }}
      myorg/my-service:latest
    # 멀티 플랫폼 지원
    platforms: linux/amd64,linux/arm64
    # 빌드 캐시 활용
    cache-from: type=registry,ref=myorg/my-service:buildcache
    cache-to: type=registry,ref=myorg/my-service:buildcache,mode=max
```

### Harbor - 온프레미스 프라이빗 레지스트리

Harbor는 CNCF 졸업 프로젝트로, 온프레미스 또는 프라이빗 클라우드 환경에서 엔터프라이즈급 컨테이너 레지스트리를 구축할 때 표준 선택이다.

**Harbor의 핵심 기능:**
- 프로젝트별 역할 기반 접근 제어(RBAC)
- 이미지 취약점 스캔 (Trivy, Clair 내장)
- 이미지 서명 및 검증 (Cosign 연동)
- 레플리케이션: 다른 레지스트리와 자동 동기화
- 웹훅: 이미지 푸시/스캔 완료 시 외부 시스템 알림

```yaml
# harbor-values.yaml (Helm 설치용)
expose:
  type: ingress
  ingress:
    hosts:
      core: registry.example.com
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls

externalURL: https://registry.example.com

persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      storageClass: "fast-ssd"
      size: 500Gi
    database:
      storageClass: "fast-ssd"
      size: 20Gi

trivy:
  enabled: true
  # 스캔 결과를 DB에 캐싱하여 반복 스캔 속도 향상
  ignoreUnfixed: false
  severity: "CRITICAL,HIGH"
  # 취약점 발견 시 이미지 푸시 차단 여부
  skipUpdate: false

notary:
  enabled: true  # 이미지 서명 지원

# Harbor 내장 데이터베이스 (프로덕션에서는 외부 PostgreSQL 권장)
database:
  type: internal
```

```bash
# Helm으로 Harbor 설치
helm repo add harbor https://helm.goharbor.io
helm repo update

helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  -f harbor-values.yaml \
  --version 1.14.0
```

### 이미지 태깅 전략

컨테이너 이미지의 태그 전략은 배포 추적성과 직결된다.

```bash
# 권장하는 다중 태그 전략
VERSION="1.2.3"
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +"%Y%m%dT%H%M%SZ")

# 1. 시맨틱 버전 태그 (릴리스 추적)
docker tag my-service:build registry.example.com/my-service:${VERSION}

# 2. Git SHA 태그 (소스 추적)
docker tag my-service:build registry.example.com/my-service:${GIT_SHA}

# 3. 날짜+SHA 조합 태그 (시간 기반 조회)
docker tag my-service:build registry.example.com/my-service:${BUILD_DATE}-${GIT_SHA}

# 4. 환경 별칭 태그 (배포 상태 추적, mutable)
docker tag my-service:build registry.example.com/my-service:stable
```

OCI Image Spec의 `org.opencontainers.image.*` 라벨로 메타데이터를 이미지에 내장하는 것도 좋은 관행이다.

```dockerfile
FROM eclipse-temurin:21.0.2_13-jre-jammy

ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.source="https://github.com/example/my-service" \
      org.opencontainers.image.title="My Service" \
      org.opencontainers.image.description="Core backend service"

WORKDIR /app
COPY --chown=nobody:nobody build/libs/my-service.jar app.jar

USER nobody
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 아티팩트 보존 정책

아티팩트를 무한정 보관하는 것은 비용 문제와 관리 복잡도를 야기한다.

체계적인 보존 정책이 필요하다.

### 보존 정책 설계 원칙

**릴리스 등급별 차등 보존**

| 아티팩트 유형 | 보존 기간 | 보존 개수 | 근거 |
|---|---|---|---|
| Production Release | 영구 or 24개월 | 무제한 | 규제 준수, 롤백 보장 |
| Release Candidate | 90일 | 최근 10개 | 릴리스 후 불필요 |
| SNAPSHOT / Nightly | 7~14일 | 최근 5개 | 빠른 순환 |
| Feature Branch | 3~7일 | 최근 3개 | 병합 후 불필요 |
| PR 빌드 | 1~3일 | 최근 2개 | 검토 완료 후 불필요 |

**Nexus 보존 정책 설정 예시**

Nexus에서는 REST API로 정리 정책(Cleanup Policy)과 컴팩트 작업(Compact)을 자동화할 수 있다.

```bash
# Nexus REST API로 SNAPSHOT 정리 정책 생성
curl -X POST \
  "https://nexus.example.com/service/rest/v1/cleanup-policies" \
  -H "Content-Type: application/json" \
  -u "${NEXUS_ADMIN}:${NEXUS_PASSWORD}" \
  -d '{
    "name": "snapshot-cleanup-7days",
    "format": "maven2",
    "notes": "7일 이상 된 SNAPSHOT 정리",
    "criteria": {
      "lastDownloaded": "7",
      "isPrerelease": "true"
    }
  }'

# Compact blobstore 작업 수동 실행 (정리 후 실제 디스크 공간 회수)
curl -X POST \
  "https://nexus.example.com/service/rest/v1/tasks/run/blobstore.compact" \
  -H "Content-Type: application/json" \
  -u "${NEXUS_ADMIN}:${NEXUS_PASSWORD}"
```

### Gradle 의존성 캐시 정리 자동화

빌드 에이전트의 로컬 캐시도 관리 대상이다.

CI/CD 파이프라인에서 캐시가 무한 증가하면 디스크를 잠식한다.

```bash
# CI 에이전트의 Gradle 캐시 주기적 정리 스크립트
#!/bin/bash
set -euo pipefail

GRADLE_CACHE_DIR="${HOME}/.gradle/caches"
MAX_AGE_DAYS=30

echo "Gradle 캐시 정리 시작: ${GRADLE_CACHE_DIR}"
echo "정리 기준: ${MAX_AGE_DAYS}일 이상 미접근 파일"

# 모듈 캐시 정리
find "${GRADLE_CACHE_DIR}/modules-*" \
  -type f \
  -atime +${MAX_AGE_DAYS} \
  -name "*.jar" \
  -delete

# 빌드 캐시 정리 (30일 이상 미사용)
find "${GRADLE_CACHE_DIR}/build-cache-*" \
  -type f \
  -atime +${MAX_AGE_DAYS} \
  -delete

echo "정리 완료. 현재 캐시 크기:"
du -sh "${GRADLE_CACHE_DIR}" 2>/dev/null || echo "캐시 없음"
```

---

## 취약점 스캔 자동화

아티팩트 저장소에 올라온 이미지나 라이브러리에 CVE 취약점이 발견되면, 배포를 즉시 차단하거나 운영팀에 알려야 한다.

취약점 스캔을 파이프라인에 통합하는 것은 선택이 아닌 필수다.

### Trivy를 활용한 컨테이너 이미지 스캔

Trivy는 가장 널리 쓰이는 오픈소스 취약점 스캐너다.

OS 패키지, 언어별 의존성(JAR, npm, pip), Dockerfile 미스설정, IaC 파일까지 스캔한다.

```yaml
# GitHub Actions 파이프라인에 Trivy 통합
name: Build and Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # SARIF 업로드 권한

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-service:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          # CRITICAL 취약점 발견 시 빌드 실패
          exit-code: '1'
          ignore-unfixed: false

      - name: Upload Trivy scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()  # 스캔 실패해도 결과 업로드
        with:
          sarif_file: 'trivy-results.sarif'
```

ECR에 푸시 전 로컬 스캔으로 취약 이미지 차단:

```bash
#!/bin/bash
# scan-before-push.sh

IMAGE="my-service:${VERSION}"
SEVERITY_THRESHOLD="CRITICAL"

echo "이미지 취약점 스캔 시작: ${IMAGE}"

# Trivy 스캔 실행 (JSON 출력)
SCAN_RESULT=$(trivy image \
  --severity "${SEVERITY_THRESHOLD}" \
  --exit-code 0 \
  --format json \
  "${IMAGE}")

# 취약점 개수 추출
VULN_COUNT=$(echo "${SCAN_RESULT}" | \
  jq '[.Results[]?.Vulnerabilities // [] | .[] | select(.Severity == "CRITICAL")] | length')

echo "발견된 CRITICAL 취약점: ${VULN_COUNT}개"

if [ "${VULN_COUNT}" -gt 0 ]; then
  echo "취약점 목록:"
  echo "${SCAN_RESULT}" | jq -r '
    .Results[]?.Vulnerabilities // [] |
    .[] |
    select(.Severity == "CRITICAL") |
    "  \(.VulnerabilityID): \(.PkgName) \(.InstalledVersion) -> \(.FixedVersion // "no fix"): \(.Title)"
  '
  echo ""
  echo "CRITICAL 취약점이 발견되어 푸시를 차단합니다."
  exit 1
fi

echo "취약점 없음. 이미지를 푸시합니다."
docker push "${ECR_URI}:${VERSION}"
```

### Dependency-Check를 활용한 JAR 의존성 스캔

컨테이너 이미지뿐 아니라 Maven/Gradle 의존성 JAR 파일 자체에 대한 CVE 스캔도 필요하다.

OWASP Dependency-Check가 표준 도구다.

```kotlin
// build.gradle.kts에 OWASP Dependency-Check 플러그인 추가
plugins {
    id("org.owasp.dependencycheck") version "9.0.9"
}

dependencyCheck {
    // NVD API 키 (속도 제한 완화를 위해 등록 권장)
    nvd.apiKey = System.getenv("NVD_API_KEY") ?: ""

    // CVSS 점수 7.0 이상에서 빌드 실패
    failBuildOnCVSS = 7.0f

    // 억제 파일 (알려진 false positive 처리)
    suppressionFile = "config/dependency-check-suppressions.xml"

    formats = listOf("HTML", "JSON", "SARIF")
    outputDirectory = "${buildDir}/reports/dependency-check"
}
```

```xml
<!-- config/dependency-check-suppressions.xml -->
<!-- 알려진 false positive나 수용 가능한 취약점을 문서화 -->
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">

  <!-- 예시: 특정 CVE를 억제하는 경우 반드시 근거를 문서화 -->
  <suppress>
    <notes>
      CVE-2023-12345: 해당 코드 경로를 사용하지 않음.
      보안팀 검토 완료: 2024-03-15 (담당: security@example.com)
      재검토 예정: 2024-09-15
    </notes>
    <cve>CVE-2023-12345</cve>
  </suppress>

</suppressions>
```

### 레지스트리 수준의 지속적 스캔

빌드 시점 스캔만으로는 부족하다. 이미 배포된 이미지에 새로운 CVE가 발견될 수 있다.

Harbor의 스케줄 스캔과 ECR의 Enhanced Scanning을 활용한다.

```bash
# Harbor API로 모든 레포지토리 스캔 트리거
HARBOR_URL="https://registry.example.com"
PROJECT="my-project"

# 특정 프로젝트의 모든 레포지토리 조회
REPOS=$(curl -s \
  -u "${HARBOR_USER}:${HARBOR_PASSWORD}" \
  "${HARBOR_URL}/api/v2.0/projects/${PROJECT}/repositories?page_size=100" | \
  jq -r '.[].name')

# 각 레포지토리의 최신 아티팩트 스캔
for REPO in ${REPOS}; do
  ARTIFACTS=$(curl -s \
    -u "${HARBOR_USER}:${HARBOR_PASSWORD}" \
    "${HARBOR_URL}/api/v2.0/projects/${PROJECT}/repositories/${REPO##*/}/artifacts?page_size=20" | \
    jq -r '.[].digest')

  for DIGEST in ${ARTIFACTS}; do
    echo "스캔 시작: ${REPO}@${DIGEST}"
    curl -s -X POST \
      -u "${HARBOR_USER}:${HARBOR_PASSWORD}" \
      "${HARBOR_URL}/api/v2.0/projects/${PROJECT}/repositories/${REPO##*/}/artifacts/${DIGEST}/scan" \
      > /dev/null
  done
done

echo "전체 스캔 트리거 완료"
```

취약점 발견 시 Slack 알림을 보내는 Harbor 웹훅 핸들러 예시:

```python
# webhook_handler.py (AWS Lambda or 간단한 Flask 서버)
import json
import os
import urllib.request

def handle_harbor_webhook(event):
    """Harbor 웹훅 이벤트 처리"""
    payload = json.loads(event['body'])

    if payload.get('type') != 'SCANNING_COMPLETED':
        return {'statusCode': 200}

    resource = payload['event_data']['resources'][0]
    repo_name = resource['repository']['name']
    tag = resource.get('tag', 'unknown')
    scan_overview = resource.get('scan_overview', {})

    # 취약점 요약 추출 (Trivy 스캐너 결과)
    summary = scan_overview.get(
        'application/vnd.scanner.adapter.vuln.report.harbor+json; version=1.0',
        {}
    ).get('summary', {})

    critical = summary.get('fixable', 0) + summary.get('total', 0)
    critical_count = summary.get('summary', {}).get('CRITICAL', 0)
    high_count = summary.get('summary', {}).get('HIGH', 0)

    if critical_count > 0 or high_count > 0:
        message = {
            "text": f":rotating_light: *취약점 발견*: `{repo_name}:{tag}`",
            "attachments": [{
                "color": "danger" if critical_count > 0 else "warning",
                "fields": [
                    {"title": "CRITICAL", "value": str(critical_count), "short": True},
                    {"title": "HIGH", "value": str(high_count), "short": True},
                    {"title": "레지스트리", "value": os.environ['HARBOR_URL'], "short": False}
                ]
            }]
        }

        req = urllib.request.Request(
            os.environ['SLACK_WEBHOOK_URL'],
            data=json.dumps(message).encode(),
            headers={'Content-Type': 'application/json'},
            method='POST'
        )
        urllib.request.urlopen(req)

    return {'statusCode': 200}
```

### 안티패턴: 스캔 결과 무시

```yaml
# 잘못된 예: exit-code를 0으로 설정하여 취약점을 탐지해도 빌드 계속 진행
- name: Trivy scan (경고만)
  uses: aquasecurity/trivy-action@master
  with:
    exit-code: '0'   # 취약점이 있어도 빌드 성공 처리 — 의미 없는 스캔
    severity: 'CRITICAL'
```

취약점 스캔 결과를 파이프라인에 반영하지 않으면 스캔 자체가 형식적인 절차로 전락한다.

`exit-code: 1`로 설정하고, 억제(suppression)는 반드시 문서화된 근거와 함께 허용해야 한다.

---

## 전체 파이프라인 통합 예시

지금까지 다룬 내용을 하나의 완성된 CI/CD 파이프라인으로 통합하면 다음과 같다.

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-northeast-2.amazonaws.com
  IMAGE_NAME: my-service

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      git-sha: ${{ steps.version.outputs.git-sha }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Resolve version
        id: version
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            VERSION="${GITHUB_REF_NAME#v}"
          else
            LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
            NEXT=$(echo "${LAST_TAG#v}" | awk -F. '{print $1"."$2"."$3+1}')
            VERSION="${NEXT}-SNAPSHOT"
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "git-sha=${GIT_SHA}" >> $GITHUB_OUTPUT

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build and test
        run: ./gradlew build test --no-daemon
        env:
          RELEASE_VERSION: ${{ steps.version.outputs.version }}

      - name: Dependency vulnerability scan
        run: ./gradlew dependencyCheckAnalyze --no-daemon
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}

      - name: Publish Maven artifact
        if: success()
        run: ./gradlew publish --no-daemon
        env:
          RELEASE_VERSION: ${{ steps.version.outputs.version }}
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}

  build-and-push-image:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            --build-arg GIT_COMMIT=${{ needs.build-and-test.outputs.git-sha }} \
            --build-arg VERSION=${{ needs.build-and-test.outputs.version }} \
            -t ${IMAGE_NAME}:build .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:build
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          format: 'table'

      - name: Push to ECR
        if: success()
        run: |
          VERSION="${{ needs.build-and-test.outputs.version }}"
          GIT_SHA="${{ needs.build-and-test.outputs.git-sha }}"

          docker tag ${IMAGE_NAME}:build ${REGISTRY}/${IMAGE_NAME}:${VERSION}
          docker tag ${IMAGE_NAME}:build ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}

          docker push ${REGISTRY}/${IMAGE_NAME}:${VERSION}
          docker push ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}
```

---

아티팩트 관리는 "빌드 결과를 어딘가에 저장하는 것"이 아니다.

버전 추적, 보안, 재현성, 비용 효율을 동시에 달성하는 엔지니어링 규율이다.

Nexus나 Harbor 같은 저장소를 도입하는 것이 시작점이지만, SemVer 규칙을 CI에 강제하고, 스캔 결과가 배포를 차단하도록 파이프라인을 구성하고, 보존 정책으로 비용을 통제할 때 비로소 완성된다.

취약점이 발견됐을 때 "어느 팀의 어떤 서비스가 영향을 받는가"를 30초 안에 답할 수 있다면, 아티팩트 관리가 제대로 작동하고 있다는 증거다.

---

## 참고 자료

- [Sonatype Nexus Repository Documentation](https://help.sonatype.com/repomanager3) — Nexus 3.x 공식 문서, 저장소 설정과 REST API 레퍼런스
- [JFrog Artifactory User Guide](https://jfrog.com/help/r/jfrog-artifactory-documentation) — Artifactory 공식 문서, Build Info와 Xray 연동 가이드
- [Semantic Versioning 2.0.0](https://semver.org/lang/ko/) — SemVer 공식 명세 한국어 번역
- [Trivy Documentation](https://aquasecurity.github.io/trivy/) — Aqua Security Trivy 공식 문서, 스캔 유형별 설정 가이드
- [Harbor Documentation](https://goharbor.io/docs/) — CNCF Harbor 공식 문서, 설치와 운영 가이드
- [OCI Image Format Specification](https://github.com/opencontainers/image-spec) — 컨테이너 이미지 레이블과 메타데이터 표준 명세
