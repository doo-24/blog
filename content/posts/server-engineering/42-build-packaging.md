---
title: "[배포와 CI/CD] 2편 — 빌드와 패키징: 코드가 실행 가능한 산출물이 되기까지"
date: 2026-03-17T20:06:00+09:00
draft: false
tags: ["빌드", "Gradle", "Maven", "Docker", "패키징", "서버"]
series: ["배포와 CICD"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 6
summary: "빌드 도구(Gradle/Maven)와 의존성 관리, 패키징 전략(Fat JAR, 레이어드 JAR, 네이티브 이미지), 빌드 재현성과 빌드 캐시 최적화, 컨테이너 이미지 태깅 전략까지"
---

소스 코드는 실행되지 않는다. 개발자가 작성한 `.java` 파일, `.kt` 파일은 그 자체로는 아무 의미가 없다.

빌드 파이프라인은 이 코드를 컴파일하고, 의존성을 묶고, 검증하고, 최종적으로 실행 가능한 산출물로 변환하는 과정이다.

이 과정이 느리거나, 재현 불가능하거나, 산출물이 불필요하게 크다면 이후의 모든 배포 전략은 흔들린다.

빌드를 설계하는 일은 곧 신뢰할 수 있는 소프트웨어 공급망을 설계하는 일이다.

---

## 빌드 도구: Gradle vs Maven

JVM 기반 서버 애플리케이션에서 빌드 도구의 양대 산맥은 Maven과 Gradle이다.

두 도구 모두 의존성 관리, 컴파일, 테스트, 패키징을 담당하지만 철학과 성능 특성이 다르다.

### Maven: 선언적이고 예측 가능한 빌드

Maven은 XML 기반의 `pom.xml`로 빌드를 선언한다. 고정된 라이프사이클(`validate → compile → test → package → verify → install → deploy`)이 있고, 이 순서를 벗어날 수 없다.

예측 가능성이 높지만 유연성이 낮다.

```xml
<!-- pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-service</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <java.version>21</java.version>
    <spring-boot.version>3.2.3</spring-boot.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <!-- 레이어드 JAR 활성화 -->
          <layers>
            <enabled>true</enabled>
          </layers>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### Gradle: 유연하고 빠른 빌드

Gradle은 Groovy 또는 Kotlin DSL로 빌드 스크립트를 작성한다.

태스크 그래프 기반으로 동작하며 증분 빌드(incremental build)와 빌드 캐시를 기본으로 지원한다.

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
}

group = "com.example"
version = "1.0.0"

java {
    toolchains {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

    runtimeOnly("org.postgresql:postgresql")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.9")
}

tasks.bootJar {
    // 레이어드 JAR 활성화
    layered {
        enabled = true
    }
    archiveFileName = "app.jar"
}

tasks.test {
    useJUnitPlatform()
    // 병렬 테스트 실행
    maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
}
```

### 의존성 관리의 핵심: 버전 충돌과 BOM

대규모 프로젝트에서 의존성 버전 충돌은 흔한 문제다.

Maven의 `dependencyManagement`와 Gradle의 `platform()`은 BOM(Bill of Materials)을 활용해 이를 해결한다.

```kotlin
// Gradle에서 BOM을 통한 버전 통합 관리
dependencies {
    // Spring Boot BOM - 하위 라이브러리 버전을 BOM이 결정
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.3"))
    implementation(platform("io.micrometer:micrometer-bom:1.12.3"))

    // 버전 명시 없이 사용 가능
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

Gradle에서 의존성 충돌을 분석하려면 다음 명령으로 의존성 트리를 확인한다.

```bash
# 특정 설정의 의존성 트리 출력
./gradlew dependencies --configuration runtimeClasspath

# 특정 라이브러리가 어디서 끌려오는지 추적
./gradlew dependencyInsight --dependency spring-core --configuration runtimeClasspath
```

**안티패턴: 버전을 각 의존성에 직접 명시하기.** BOM 없이 모든 의존성에 버전을 개별 명시하면 업그레이드 시 호환성 검증이 불가능해진다. 연관 라이브러리 간 버전 정합성은 BOM이 보장한다.

---

## 패키징 전략

코드를 컴파일했다면 이제 어떤 형태로 묶을지 결정해야 한다.

선택지에 따라 실행 속도, 이미지 크기, 배포 전략이 달라진다.

### Fat JAR (Uber JAR)

가장 단순한 방법이다. 애플리케이션 코드와 모든 의존성을 하나의 JAR로 묶는다.

`java -jar app.jar`로 실행 가능하며 별도의 클래스패스 설정이 필요 없다.

```bash
# Gradle로 Fat JAR 생성
./gradlew bootJar

# 결과물 확인
ls -lh build/libs/app.jar
# -rw-r--r-- 1 user user 85M app.jar

# 실행
java -jar build/libs/app.jar
```

Fat JAR의 문제는 **크기와 Docker 레이어 캐시**다.

의존성이 변경되지 않아도 코드 한 줄만 바꾸면 85MB 전체를 다시 전송해야 한다.

CI/CD 환경에서 이는 느린 빌드와 높은 레지스트리 대역폭으로 이어진다.

### 레이어드 JAR (Layered JAR)

Spring Boot 2.3부터 도입된 방식이다.

JAR 내부를 여러 레이어로 분리해 Docker 이미지 빌드 시 변경된 레이어만 갱신하도록 한다.

레이어 구조는 변경 빈도 순서로 구성된다.

1. `dependencies` — 외부 라이브러리 (거의 변경 없음)
2. `spring-boot-loader` — Spring Boot 로더 (거의 변경 없음)
3. `snapshot-dependencies` — SNAPSHOT 버전 의존성 (가끔 변경)
4. `application` — 애플리케이션 코드 (자주 변경)

```kotlin
// build.gradle.kts - 레이어드 JAR 설정
tasks.bootJar {
    layered {
        enabled = true
        // 커스텀 레이어 순서 정의
        application {
            intoLayer("spring-boot-loader") {
                include("org/springframework/boot/loader/**")
            }
            intoLayer("application")
        }
        dependencies {
            intoLayer("snapshot-dependencies") {
                include("*:*:*SNAPSHOT")
            }
            intoLayer("dependencies")
        }
        layerOrder = listOf(
            "dependencies",
            "spring-boot-loader",
            "snapshot-dependencies",
            "application"
        )
    }
}
```

레이어드 JAR를 위한 Dockerfile은 멀티 스테이지 빌드로 레이어를 추출한다.

```dockerfile
# --- 빌드 스테이지 ---
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Gradle Wrapper와 빌드 설정 파일 먼저 복사 (의존성 캐시 활용)
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# 소스 복사 후 빌드
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# --- 레이어 추출 스테이지 ---
FROM eclipse-temurin:21-jre-alpine AS extractor
WORKDIR /app
COPY --from=builder /app/build/libs/app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# --- 런타임 스테이지 ---
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# 변경 빈도가 낮은 레이어부터 복사 (Docker 캐시 최적화)
COPY --from=extractor /app/dependencies ./
COPY --from=extractor /app/spring-boot-loader ./
COPY --from=extractor /app/snapshot-dependencies ./
COPY --from=extractor /app/application ./

# 보안: non-root 사용자로 실행
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

이 구조에서 코드만 변경한다면 `application` 레이어만 교체된다.

85MB 전체 대신 수 MB만 전송된다.

### 네이티브 이미지 (GraalVM Native Image)

Spring Boot 3과 GraalVM Native Image를 조합하면 JVM 없이 실행되는 네이티브 바이너리를 생성할 수 있다.

시작 시간이 수십 밀리초로 단축되고 메모리 사용량이 대폭 줄어든다.

```kotlin
// build.gradle.kts - 네이티브 이미지 플러그인 추가
plugins {
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    id("org.graalvm.buildtools.native") version "0.9.28"
    kotlin("jvm") version "1.9.22"
}

graalvmNative {
    binaries {
        named("main") {
            imageName = "my-service"
            buildArgs.add("--initialize-at-build-time=org.slf4j")
            buildArgs.add("-H:+ReportExceptionStackTraces")
            // AOT 컴파일 최적화
            buildArgs.add("-O2")
        }
    }
    toolchainDetection = false
}
```

```bash
# 네이티브 이미지 빌드 (시간 소요 큼 — 5~15분)
./gradlew nativeCompile

# 결과물: JVM 없이 직접 실행 가능
./build/native/nativeCompile/my-service

# 시작 시간 비교
# Fat JAR:      Started in 3.2 seconds
# Native Image: Started in 0.08 seconds
```

네이티브 이미지는 리플렉션, 동적 프록시, 직렬화를 컴파일 타임에 알아야 한다.

Spring AOT가 대부분을 자동 처리하지만 커스텀 코드에서 추가 힌트가 필요할 수 있다.

```java
// 리플렉션 힌트 등록 예시
@Configuration
@ImportRuntimeHints(MyRuntimeHints.class)
public class AppConfig {
    // ...
}

public class MyRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection()
            .registerType(MyDynamicClass.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                MemberCategory.INVOKE_PUBLIC_METHODS);
    }
}
```

**안티패턴: 모든 서비스를 네이티브 이미지로 전환하기.** 빌드 시간이 수십 배 길어지고, 런타임 동적 기능에 제약이 생긴다. 네이티브 이미지는 짧은 실행 시간이 중요한 람다 함수나 CLI 도구에 적합하다. 장기 실행 서버에서는 JVM의 JIT 최적화가 더 유리한 경우가 많다.

---

## 빌드 재현성

같은 소스 코드를 빌드했을 때 항상 동일한 산출물이 나와야 한다.

이를 **재현 가능한 빌드(reproducible build)**라 한다.

### 재현성을 깨는 요인들

1. **타임스탬프**: JAR 파일 내 항목에 빌드 시각이 기록되면 바이너리가 달라진다.
2. **파일 시스템 정렬 순서**: OS마다 디렉터리 순회 순서가 다를 수 있다.
3. **환경 변수**: 빌드 스크립트가 환경 변수를 읽으면 환경마다 결과가 달라진다.
4. **부동 소수점 연산**: 아키텍처에 따라 결과가 다를 수 있다.

```kotlin
// build.gradle.kts - 재현 가능한 빌드 설정
tasks.withType<AbstractArchiveTask>().configureEach {
    // 타임스탬프 고정 (1980-02-01 00:00:00 UTC)
    isPreserveFileTimestamps = false
    isReproducibleFileOrder = true
}

// JAR Manifest에 빌드 시간 제외
tasks.bootJar {
    manifest {
        // 빌드 환경 정보를 manifest에서 제거
        attributes.remove("Build-Time")
        attributes.remove("Built-By")
    }
}
```

### Gradle Wrapper로 빌드 도구 버전 고정

빌드 도구 버전 자체도 고정해야 한다.

`gradle-wrapper.properties`가 이 역할을 한다.

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.6-bin.zip
# 체크섬으로 무결성 검증
distributionSha256Sum=85719317abd2112f021d4f41f09ec370534ba288432065f4b477b6a3b652de4d
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

Maven에서는 `mvnw`와 `.mvn/wrapper/maven-wrapper.properties`가 동일한 역할을 한다.

```properties
# .mvn/wrapper/maven-wrapper.properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
wrapperUrl=https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper/3.2.0/maven-wrapper-3.2.0.jar
```

---

## 빌드 캐시 최적화

빌드 시간을 줄이는 가장 효과적인 방법은 변경되지 않은 부분을 다시 빌드하지 않는 것이다.

### Gradle 빌드 캐시

Gradle은 태스크 출력을 캐시하고 입력이 동일하면 캐시에서 결과를 가져온다.

로컬 캐시와 원격 캐시 모두 지원한다.

```kotlin
// settings.gradle.kts - 원격 빌드 캐시 설정
buildCache {
    local {
        isEnabled = true
        directory = File(rootDir, ".gradle/build-cache")
        removeUnusedEntriesAfterDays = 30
    }
    remote<HttpBuildCache> {
        url = uri("https://build-cache.internal.example.com/cache/")
        isPush = System.getenv("CI") != null  // CI에서만 캐시 push
        credentials {
            username = System.getenv("BUILD_CACHE_USER")
            password = System.getenv("BUILD_CACHE_PASSWORD")
        }
    }
}
```

```bash
# 빌드 캐시 활성화하여 빌드
./gradlew bootJar --build-cache

# 캐시 적중률 확인 (--scan 플래그로 Gradle Build Scan 활성화)
./gradlew bootJar --build-cache --scan
```

빌드 캐시를 최대한 활용하려면 태스크 입력을 정확히 선언해야 한다.

```kotlin
// 커스텀 태스크에서 입력/출력 명확히 선언
abstract class GenerateApiDocs : DefaultTask() {

    @get:InputDirectory
    abstract val sourceDir: DirectoryProperty

    @get:InputFile
    abstract val configFile: RegularFileProperty

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @TaskAction
    fun generate() {
        // 태스크 로직
    }
}
```

### Docker 빌드 캐시

Dockerfile에서도 레이어 순서가 캐시 효율에 직접 영향을 준다.

```dockerfile
# 나쁜 예: 소스 복사 후 의존성 다운로드
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY . .                          # 코드 변경 시 아래 모두 재실행
RUN ./gradlew bootJar             # 의존성 다운로드 포함 — 느림

# 좋은 예: 의존성 파일 먼저 복사
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# 의존성 설정 파일만 먼저 복사
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./

# 의존성만 다운로드 (코드 변경과 무관하게 캐시됨)
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# 소스 복사 및 빌드
COPY src src
RUN ./gradlew bootJar --no-daemon -x test
```

BuildKit을 활용하면 Maven 로컬 저장소나 Gradle 캐시를 빌드 간에 마운트해 재사용할 수 있다.

```dockerfile
# BuildKit 캐시 마운트 활용 (Gradle)
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./
COPY src src

RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew bootJar --no-daemon -x test
```

```bash
# BuildKit 활성화하여 빌드
DOCKER_BUILDKIT=1 docker build -t my-service:latest .

# 또는 docker buildx 사용
docker buildx build --cache-from type=registry,ref=my-registry/my-service:cache \
                    --cache-to type=registry,ref=my-registry/my-service:cache,mode=max \
                    -t my-service:1.2.3 .
```

**안티패턴: CI에서 매번 처음부터 빌드하기.** 빌드 캐시 없이 CI 파이프라인을 실행하면 의존성 다운로드에만 수 분이 소요된다. GitHub Actions, GitLab CI, Jenkins 모두 캐시 기능을 제공한다.

```yaml
# GitHub Actions 빌드 캐시 예시
- name: Cache Gradle packages
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      ${{ runner.os }}-gradle-

- name: Build with Gradle
  run: ./gradlew bootJar --build-cache -x test
```

---

## 컨테이너 이미지 태깅 전략

이미지를 빌드했다면 어떻게 태그를 붙일지 결정해야 한다.

태깅 전략은 롤백 가능성, 배포 추적성, 레지스트리 용량에 직결된다.

### `latest`의 함정

`latest` 태그는 편리해 보이지만 운영 환경에서 심각한 문제를 일으킨다.

- **롤백 불가**: `latest`를 덮어쓰면 이전 버전을 가리킬 방법이 없다.
- **추적 불가**: 현재 배포된 버전이 어떤 커밋인지 알 수 없다.
- **경쟁 조건**: 여러 CI 파이프라인이 동시에 `latest`를 push하면 예측 불가한 이미지가 배포된다.
- **캐시 오염**: Kubernetes가 `imagePullPolicy: IfNotPresent`로 설정되어 있으면 `latest`가 갱신되어도 이전 이미지를 실행한다.

**`latest`는 로컬 개발과 테스트에서만 허용한다. 운영 환경에는 절대 사용하지 않는다.**

### 올바른 태깅 전략

#### Git SHA 태그

가장 신뢰할 수 있는 방식이다.

특정 이미지가 정확히 어떤 커밋에서 빌드되었는지 추적 가능하다.

```bash
# 전체 SHA
GIT_SHA=$(git rev-parse HEAD)
docker build -t my-registry/my-service:${GIT_SHA} .

# 단축 SHA (7자리)
GIT_SHA_SHORT=$(git rev-parse --short HEAD)
docker build -t my-registry/my-service:${GIT_SHA_SHORT} .
```

```yaml
# GitHub Actions에서 SHA 태깅
- name: Build and push Docker image
  run: |
    SHA=${{ github.sha }}
    SHORT_SHA=${SHA:0:7}
    IMAGE=my-registry/my-service

    docker build \
      --label "git.sha=${SHA}" \
      --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      -t ${IMAGE}:${SHORT_SHA} \
      -t ${IMAGE}:${GITHUB_REF_NAME} \
      .
    docker push ${IMAGE}:${SHORT_SHA}
    docker push ${IMAGE}:${GITHUB_REF_NAME}
```

#### 시맨틱 버전 태그

릴리스 버전이 명확한 경우 시맨틱 버저닝을 태그로 사용한다.

SHA 태그와 함께 사용하면 더욱 강력하다.

```bash
VERSION="1.4.2"
GIT_SHA=$(git rev-parse --short HEAD)

# 시맨틱 버전 + SHA를 함께 태깅
docker tag my-service:${GIT_SHA} my-registry/my-service:${VERSION}
docker tag my-service:${GIT_SHA} my-registry/my-service:${VERSION}-${GIT_SHA}

# 마이너 버전 태그도 업데이트 (1.4를 1.4.2로 이동)
docker tag my-service:${GIT_SHA} my-registry/my-service:1.4
```

#### 브랜치 기반 태그

`main`, `develop`, `staging` 같은 브랜치 이름을 태그로 사용하면 환경 별 추적이 용이하다.

단, 이 태그는 이동하므로 롤백 기준으로는 SHA 태그를 사용해야 한다.

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD | tr '/' '-')
GIT_SHA=$(git rev-parse --short HEAD)

# 브랜치 태그 (이동 가능 — 현재 상태 표시용)
docker tag my-service:${GIT_SHA} my-registry/my-service:${BRANCH}

# SHA 태그 (불변 — 롤백용)
docker push my-registry/my-service:${GIT_SHA}
docker push my-registry/my-service:${BRANCH}
```

### 이미지 레이블로 메타데이터 추가

OCI 이미지 스펙은 레이블을 통해 메타데이터를 이미지에 포함하도록 권장한다.

```dockerfile
# OCI 표준 레이블
LABEL org.opencontainers.image.title="My Service"
LABEL org.opencontainers.image.description="Core business service"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
LABEL org.opencontainers.image.created="${BUILD_DATE}"
LABEL org.opencontainers.image.source="https://github.com/example/my-service"
LABEL org.opencontainers.image.licenses="MIT"
```

```bash
# 레이블을 빌드 시 주입
docker build \
  --build-arg VERSION=1.4.2 \
  --build-arg GIT_SHA=$(git rev-parse HEAD) \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --label "org.opencontainers.image.revision=$(git rev-parse HEAD)" \
  --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -t my-service:1.4.2 .

# 레이블 확인
docker inspect --format '{{json .Config.Labels}}' my-service:1.4.2 | jq .
```

### 이미지 레지스트리 정리 전략

태그를 정교하게 붙이더라도 오래된 이미지를 정리하지 않으면 레지스트리 비용이 증가한다.

```bash
# GitHub Container Registry에서 30일 이상 된 비태그 이미지 삭제
gh api \
  -H "Accept: application/vnd.github+json" \
  /orgs/{org}/packages/container/{package_name}/versions \
  --jq '.[] | select(.metadata.container.tags | length == 0) | select(.updated_at < (now - 30*24*3600 | todate)) | .id' \
  | xargs -I{} gh api --method DELETE /orgs/{org}/packages/container/{package_name}/versions/{}
```

정책으로 명문화하면 다음과 같다.
- **SHA 태그**: 90일 보관 후 삭제
- **브랜치 태그**: 브랜치 삭제 시 함께 삭제
- **시맨틱 버전 태그**: 영구 보관 (또는 major 버전 종료 후 1년)
- **`latest` 태그**: 운영 레지스트리에서 허용하지 않음

---

## 완성된 CI 빌드 파이프라인

지금까지 다룬 내용을 하나의 GitHub Actions 워크플로우로 통합하면 다음과 같다.

```yaml
# .github/workflows/build.yml
name: Build and Package

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Build with Gradle
        run: ./gradlew bootJar --build-cache -x test

      - name: Run tests
        run: ./gradlew test --build-cache

      - name: Set image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            GIT_SHA=${{ github.sha }}
            BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

---

빌드와 패키징은 배포 파이프라인의 출발점이다.

느리고 재현 불가능한 빌드, 불필요하게 큰 이미지, 추적 불가능한 태그는 운영 장애 대응을 어렵게 만든다.

레이어드 JAR로 Docker 캐시를 최대한 활용하고, 빌드 캐시로 파이프라인 속도를 높이고, SHA 기반 불변 태그로 모든 배포를 추적 가능하게 만드는 것이 핵심이다.

다음 편에서는 이렇게 만들어진 산출물을 실제로 배포하는 전략을 다룬다.

---

## 참고 자료

- [Spring Boot Reference — Packaging Executable Archives](https://docs.spring.io/spring-boot/docs/current/reference/html/executable-jar.html)
- [Gradle Build Cache Documentation](https://docs.gradle.org/current/userguide/build_cache.html)
- [GraalVM Native Image Spring Boot Support](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)
- [Docker Best Practices for Java Applications](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [OCI Image Format Specification — Annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
- [Reproducible Builds — Why and How](https://reproducible-builds.org/docs/definition/)
