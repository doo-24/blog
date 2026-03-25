---
title: "[Frontend 통합] 7편 — 모노레포와 배포 파이프라인: 프론트와 백엔드를 함께 관리하기"
date: 2026-03-17T12:01:00+09:00
draft: false
tags: ["모노레포", "Turborepo", "Nx", "배포 파이프라인", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "모노레포 도구(Turborepo, Nx)와 패키지 경계 설계, 공유 코드(API 타입, 유효성 검증 스키마) 관리, 프론트/백 독립 배포 파이프라인과 빌드 트리거 최적화, 환경별 API 엔드포인트 주입까지"
---

서버 엔지니어 입장에서 프론트엔드와 백엔드를 별도 레포지토리로 관리하다 보면 특정 시점에 반드시 벽에 부딪힌다.

API 타입이 맞지 않아 런타임 오류가 발생하거나, 프론트팀이 구버전 응답 스키마에 맞춰 코드를 짜고 있을 때 백엔드가 이미 변경을 배포한 상황이 대표적이다.

모노레포는 이 문제를 구조적으로 해결하는 접근법이다. 단순히 코드를 한 곳에 모으는 것이 아니라, 의존성 그래프를 명확히 하고 공유 코드를 패키지 단위로 격리하는 것이 핵심이다.

---

## 1. 모노레포 도구 선택: Turborepo vs Nx

### Turborepo

Vercel이 만든 빌드 오케스트레이터다. 설정이 단순하고 기존 `package.json` 스크립트 구조를 그대로 활용한다.

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "outputs": []
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    }
  }
}
```

`dependsOn: ["^build"]`는 의존하는 패키지의 `build`가 먼저 끝나야 현재 패키지의 `build`를 실행한다는 의미다.

캐시가 Turborepo의 핵심 강점이다. 파일 해시 기반으로 이전 빌드 결과를 재사용하며, 원격 캐시(Remote Cache)를 설정하면 CI 환경에서도 팀 전체가 캐시를 공유한다.

### Nx

Nrwl이 만든 도구로 Turborepo보다 기능이 방대하다. 코드 생성기(generator), 플러그인 시스템, 영향 분석(affected) 명령어를 내장하고 있다.

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "lint", "test", "e2e"]
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    }
  },
  "affected": {
    "defaultBase": "main"
  }
}
```

Nx의 `affected` 명령어는 변경된 파일에 영향을 받는 패키지만 골라서 빌드하거나 테스트한다. 큰 모노레포에서 CI 시간을 획기적으로 줄여준다.

```bash
# main 브랜치 대비 변경된 패키지만 빌드
nx affected --target=build

# 변경된 패키지만 테스트
nx affected --target=test
```

### 어떤 것을 선택할까

단순한 빌드 파이프라인과 빠른 캐시가 필요하다면 Turborepo가 적합하다. 대규모 팀, 세밀한 코드 생성, 영향 범위 분석이 필요하다면 Nx를 선택한다.

서버 엔지니어링 관점에서 백엔드(Node.js/Nest.js)와 프론트엔드(Next.js)를 함께 관리하는 중소 규모 프로젝트라면 Turborepo가 진입 장벽이 낮다.

---

## 2. 패키지 경계 설계

모노레포 구조에서 패키지를 어떻게 나누느냐가 유지보수성을 결정한다.

```
apps/
  api/          # NestJS 백엔드
  web/          # Next.js 프론트엔드
  admin/        # 관리자 페이지

packages/
  shared-types/ # API 요청/응답 타입 정의
  validators/   # Zod 스키마 (공유 유효성 검증)
  ui/           # 공유 UI 컴포넌트
  config/       # ESLint, TypeScript 공유 설정
  utils/        # 날짜, 문자열 등 순수 유틸리티
```

각 패키지는 독립적인 `package.json`을 갖는다.

```json
// packages/shared-types/package.json
{
  "name": "@myapp/shared-types",
  "version": "0.1.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch"
  },
  "devDependencies": {
    "tsup": "^8.0.0",
    "typescript": "^5.0.0"
  }
}
```

패키지 경계를 나눌 때 지켜야 할 원칙이 있다.

첫째, `packages/`는 `apps/`를 의존해서는 안 된다. 순환 의존성의 시작점이 된다.

둘째, `packages/ui`는 서버 전용 코드를 포함하면 안 된다. 클라이언트 번들에 불필요한 코드가 포함된다.

셋째, `packages/validators`는 런타임에 프론트와 백이 모두 사용할 수 있어야 하므로 외부 의존성을 최소화한다.

---

## 3. 공유 API 타입 관리

### 타입 정의 방식

백엔드와 프론트엔드가 같은 타입을 공유하면 컴파일 타임에 계약이 강제된다. 런타임에서 데이터 불일치를 찾는 것보다 훨씬 효율적이다.

```typescript
// packages/shared-types/src/user.ts

export interface User {
  id: string;
  email: string;
  name: string;
  role: UserRole;
  createdAt: string; // ISO 8601
}

export type UserRole = "admin" | "member" | "viewer";

export interface CreateUserRequest {
  email: string;
  name: string;
  role?: UserRole;
}

export interface CreateUserResponse {
  user: User;
}

export interface UpdateUserRequest {
  name?: string;
  role?: UserRole;
}

export interface PaginatedResponse<T> {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
}

export type UserListResponse = PaginatedResponse<User>;
```

백엔드 NestJS에서 이 타입을 임포트해 컨트롤러 반환 타입에 활용한다.

```typescript
// apps/api/src/users/users.controller.ts
import {
  CreateUserRequest,
  CreateUserResponse,
  UserListResponse,
} from "@myapp/shared-types";
import { Body, Controller, Get, Post, Query } from "@nestjs/common";

@Controller("users")
export class UsersController {
  @Post()
  async create(@Body() dto: CreateUserRequest): Promise<CreateUserResponse> {
    const user = await this.usersService.create(dto);
    return { user };
  }

  @Get()
  async findAll(@Query("page") page = 1): Promise<UserListResponse> {
    return this.usersService.findAll({ page });
  }
}
```

프론트엔드 Next.js에서도 동일한 패키지를 사용한다.

```typescript
// apps/web/src/lib/api/users.ts
import { CreateUserRequest, CreateUserResponse, UserListResponse } from "@myapp/shared-types";

export async function createUser(
  data: CreateUserRequest
): Promise<CreateUserResponse> {
  const res = await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });

  if (!res.ok) throw new Error("Failed to create user");
  return res.json() as Promise<CreateUserResponse>;
}

export async function fetchUsers(page = 1): Promise<UserListResponse> {
  const res = await fetch(`/api/users?page=${page}`);
  if (!res.ok) throw new Error("Failed to fetch users");
  return res.json() as Promise<UserListResponse>;
}
```

`@myapp/shared-types`가 변경되면 TypeScript 컴파일러가 `apps/api`와 `apps/web` 양쪽에서 오류를 잡아낸다. 타입 불일치를 PR 단계에서 발견할 수 있다.

---

## 4. 공유 유효성 검증 스키마 (Zod)

타입만 공유하면 런타임 검증이 빠진다. Zod 스키마를 공유하면 타입 추론과 런타임 검증을 동시에 얻는다.

```typescript
// packages/validators/src/user.schema.ts
import { z } from "zod";

export const UserRoleSchema = z.enum(["admin", "member", "viewer"]);

export const CreateUserSchema = z.object({
  email: z.string().email("올바른 이메일 형식이어야 합니다"),
  name: z
    .string()
    .min(2, "이름은 최소 2자 이상이어야 합니다")
    .max(50, "이름은 50자를 초과할 수 없습니다"),
  role: UserRoleSchema.optional().default("member"),
});

export const UpdateUserSchema = z.object({
  name: z.string().min(2).max(50).optional(),
  role: UserRoleSchema.optional(),
});

export type CreateUserInput = z.infer<typeof CreateUserSchema>;
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>;
```

```json
// packages/validators/package.json
{
  "name": "@myapp/validators",
  "dependencies": {
    "zod": "^3.22.0"
  },
  "peerDependencies": {
    "zod": "^3.22.0"
  }
}
```

백엔드에서 NestJS 파이프와 연결한다.

```typescript
// apps/api/src/common/pipes/zod-validation.pipe.ts
import { BadRequestException, Injectable, PipeTransform } from "@nestjs/common";
import { ZodSchema } from "zod";

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);
    if (!result.success) {
      throw new BadRequestException(result.error.flatten());
    }
    return result.data;
  }
}

// users.controller.ts에서 사용
@Post()
async create(
  @Body(new ZodValidationPipe(CreateUserSchema)) dto: CreateUserInput
): Promise<CreateUserResponse> {
  ...
}
```

프론트엔드에서는 폼 검증에 동일한 스키마를 사용한다.

```typescript
// apps/web/src/components/CreateUserForm.tsx
import { CreateUserSchema } from "@myapp/validators";
import { zodResolver } from "@hookform/resolvers/zod";
import { useForm } from "react-hook-form";

export function CreateUserForm() {
  const form = useForm({
    resolver: zodResolver(CreateUserSchema),
    defaultValues: { email: "", name: "", role: "member" as const },
  });

  // 백엔드와 완전히 동일한 검증 규칙이 프론트에서도 동작
}
```

이렇게 하면 백엔드가 유효성 검증 규칙을 바꿀 때 프론트엔드 폼도 자동으로 동기화된다.

---

## 5. 프론트/백 독립 배포 파이프라인

모노레포라도 프론트와 백의 배포는 독립적이어야 한다. 프론트 UI 수정이 백엔드 재배포를 강제하면 안 된다.

### GitHub Actions 구성

```yaml
# .github/workflows/deploy-api.yml
name: Deploy API

on:
  push:
    branches: [main]
    paths:
      - "apps/api/**"
      - "packages/shared-types/**"
      - "packages/validators/**"
      - "package.json"
      - "turbo.json"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build shared packages
        run: npx turbo run build --filter=@myapp/shared-types --filter=@myapp/validators

      - name: Build API
        run: npx turbo run build --filter=@myapp/api

      - name: Run tests
        run: npx turbo run test --filter=@myapp/api

      - name: Deploy to production
        run: |
          # 실제 배포 명령어 (예: Docker 빌드 후 ECS 배포)
          docker build -f apps/api/Dockerfile -t myapp-api:${{ github.sha }} .
          aws ecs update-service --cluster prod --service api --force-new-deployment
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

```yaml
# .github/workflows/deploy-web.yml
name: Deploy Web

on:
  push:
    branches: [main]
    paths:
      - "apps/web/**"
      - "packages/shared-types/**"
      - "packages/validators/**"
      - "packages/ui/**"
      - "package.json"
      - "turbo.json"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build dependencies
        run: npx turbo run build --filter=@myapp/web^...

      - name: Build web
        run: npx turbo run build --filter=@myapp/web
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.PROD_API_URL }}

      - name: Deploy to Vercel
        run: vercel deploy --prod --token ${{ secrets.VERCEL_TOKEN }}
```

`paths` 필터가 빌드 트리거 최적화의 핵심이다. `apps/api/**`가 변경될 때만 API 워크플로우가 실행되고, `apps/web/**`이 변경될 때만 Web 워크플로우가 실행된다.

공유 패키지(`packages/shared-types`, `packages/validators`)가 변경되면 양쪽 모두 트리거된다. 타입 변경은 프론트와 백 양쪽에 영향을 주므로 의도적인 설계다.

### Turborepo 필터 문법

```bash
# @myapp/api와 그것이 의존하는 모든 패키지 빌드
npx turbo run build --filter=@myapp/api...

# @myapp/web과 그 의존성 빌드 (^는 의존성만, ...은 의존성 포함 본인)
npx turbo run build --filter=@myapp/web^...

# 특정 디렉토리의 패키지만
npx turbo run build --filter=./packages/*

# main 대비 변경된 패키지만 (Nx의 affected와 유사)
npx turbo run build --filter=[main]
```

---

## 6. 환경별 API 엔드포인트 주입

### 환경 변수 계층 구조

프론트엔드는 빌드 타임에 API URL이 번들에 포함된다. 환경마다 다른 URL을 주입해야 한다.

```
apps/web/
  .env.local          # 로컬 개발 (Git 무시)
  .env.development    # 개발 환경 기본값
  .env.staging        # 스테이징 환경
  .env.production     # 프로덕션 환경
```

```bash
# apps/web/.env.development
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_APP_ENV=development
NEXT_PUBLIC_FEATURE_FLAGS=analytics:false,newDashboard:true
```

```bash
# apps/web/.env.production
NEXT_PUBLIC_API_URL=https://api.myapp.com
NEXT_PUBLIC_APP_ENV=production
NEXT_PUBLIC_FEATURE_FLAGS=analytics:true,newDashboard:false
```

```bash
# apps/web/.env.staging
NEXT_PUBLIC_API_URL=https://api-staging.myapp.com
NEXT_PUBLIC_APP_ENV=staging
NEXT_PUBLIC_FEATURE_FLAGS=analytics:false,newDashboard:true
```

### 환경 변수 타입 안전성

Next.js의 환경 변수는 기본적으로 `string | undefined`다. 런타임에 누락된 변수를 조용히 넘기면 디버깅이 어렵다.

```typescript
// apps/web/src/lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_API_URL: z.string().url(),
  NEXT_PUBLIC_APP_ENV: z.enum(["development", "staging", "production"]),
  NEXT_PUBLIC_FEATURE_FLAGS: z.string().optional().default(""),
});

function parseEnv() {
  const result = envSchema.safeParse({
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_APP_ENV: process.env.NEXT_PUBLIC_APP_ENV,
    NEXT_PUBLIC_FEATURE_FLAGS: process.env.NEXT_PUBLIC_FEATURE_FLAGS,
  });

  if (!result.success) {
    console.error("환경 변수 검증 실패:", result.error.flatten());
    throw new Error("필수 환경 변수가 누락되었거나 형식이 올바르지 않습니다");
  }

  return result.data;
}

export const env = parseEnv();
```

빌드 시점에 환경 변수가 올바르지 않으면 즉시 오류가 발생한다. 프로덕션 배포 후 API URL이 잘못된 것을 발견하는 최악의 상황을 예방한다.

### API 클라이언트 팩토리

환경 변수를 직접 참조하는 대신, API 클라이언트를 팩토리로 래핑하면 테스트 격리가 쉬워진다.

```typescript
// apps/web/src/lib/api/client.ts
import { env } from "@/lib/env";

interface ApiClientOptions {
  baseUrl: string;
  defaultHeaders?: Record<string, string>;
}

function createApiClient(options: ApiClientOptions) {
  const { baseUrl, defaultHeaders = {} } = options;

  async function request<T>(
    path: string,
    init?: RequestInit
  ): Promise<T> {
    const url = `${baseUrl}${path}`;
    const res = await fetch(url, {
      ...init,
      headers: {
        "Content-Type": "application/json",
        ...defaultHeaders,
        ...init?.headers,
      },
    });

    if (!res.ok) {
      const error = await res.json().catch(() => ({ message: res.statusText }));
      throw new ApiError(res.status, error.message);
    }

    return res.json() as Promise<T>;
  }

  return { request };
}

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    message: string
  ) {
    super(message);
    this.name = "ApiError";
  }
}

// 환경 변수로 초기화된 싱글턴 클라이언트
export const apiClient = createApiClient({
  baseUrl: env.NEXT_PUBLIC_API_URL,
});
```

### 백엔드 환경 변수 관리

백엔드는 런타임에 환경 변수를 읽는다. NestJS에서는 `@nestjs/config`와 Joi 또는 Zod로 검증한다.

```typescript
// apps/api/src/config/env.validation.ts
import { plainToClass } from "class-transformer";
import { IsEnum, IsNumber, IsString, validateSync } from "class-validator";

enum Environment {
  Development = "development",
  Staging = "staging",
  Production = "production",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  PORT: number;

  @IsString()
  DATABASE_URL: string;

  @IsString()
  JWT_SECRET: string;

  @IsString()
  FRONTEND_URL: string;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToClass(EnvironmentVariables, config, {
    enableImplicitConversion: true,
  });

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }

  return validatedConfig;
}
```

```typescript
// apps/api/src/app.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { validate } from "./config/env.validation";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validate,
      envFilePath: `.env.${process.env.NODE_ENV || "development"}`,
    }),
  ],
})
export class AppModule {}
```

---

## 7. 빌드 캐시 최적화

### Turborepo 원격 캐시 설정

로컬 캐시는 개발자 한 명에게만 유효하다. 원격 캐시를 설정하면 CI와 팀원이 빌드 결과를 공유한다.

```bash
# Vercel 원격 캐시 사용 (무료)
npx turbo login
npx turbo link

# CI에서 원격 캐시 활용
TURBO_TOKEN=${{ secrets.TURBO_TOKEN }} \
TURBO_TEAM=${{ secrets.TURBO_TEAM }} \
npx turbo run build
```

자체 호스팅 원격 캐시 서버도 구성할 수 있다.

```yaml
# turbo.json - 원격 캐시 설정
{
  "remoteCache": {
    "enabled": true,
    "preflight": false
  }
}
```

### Docker 빌드 최적화

모노레포 구조에서 Docker 빌드는 컨텍스트 크기가 문제가 된다. `turbo prune` 명령어가 이를 해결한다.

```dockerfile
# apps/api/Dockerfile
FROM node:20-alpine AS base

# turbo prune으로 api에 필요한 파일만 추출
FROM base AS pruner
RUN npm install -g turbo
WORKDIR /app
COPY . .
RUN turbo prune --scope=@myapp/api --docker

# 의존성 설치
FROM base AS installer
WORKDIR /app
COPY --from=pruner /app/out/json/ .
COPY --from=pruner /app/out/package-lock.json ./package-lock.json
RUN npm ci

# 빌드
FROM base AS builder
WORKDIR /app
COPY --from=installer /app/node_modules ./node_modules
COPY --from=pruner /app/out/full/ .
RUN npx turbo run build --filter=@myapp/api

# 실행
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/apps/api/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

EXPOSE 3001
CMD ["node", "dist/main.js"]
```

`turbo prune`은 `@myapp/api`와 그 의존성에 해당하는 파일만 `out/` 디렉토리에 추출한다. 불필요한 앱과 패키지 코드가 Docker 컨텍스트에 포함되지 않는다.

---

## 8. 실전 운영 고려사항

### 패키지 버저닝 전략

모노레포 내부 패키지는 고정 버전을 사용하지 않고 워크스페이스 참조를 사용한다.

```json
// apps/api/package.json
{
  "dependencies": {
    "@myapp/shared-types": "*",
    "@myapp/validators": "*"
  }
}
```

외부에 공개하는 패키지가 아니라면 버전 관리에 `changesets`를 사용할 이유가 없다.

### 타입 변경 시 마이그레이션 절차

`packages/shared-types`에서 API 응답 타입을 변경할 때는 다음 순서를 지킨다.

첫째, 새로운 타입을 추가하되 기존 타입은 유지한다. `@deprecated` JSDoc으로 표시한다.

둘째, 백엔드가 새 타입으로 응답하도록 변경하고 배포한다.

셋째, 프론트엔드가 새 타입으로 전환하고 배포한다.

넷째, 기존 타입을 제거한다.

이 순서를 지키면 배포 간 타입 불일치로 인한 런타임 오류를 방지할 수 있다.

### CI 병렬 실행

```yaml
# .github/workflows/ci.yml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'apps/api/**'
              - 'packages/**'
            web:
              - 'apps/web/**'
              - 'packages/**'

  test-api:
    needs: changes
    if: ${{ needs.changes.outputs.api == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - name: Test API
        run: npx turbo run test --filter=@myapp/api

  test-web:
    needs: changes
    if: ${{ needs.changes.outputs.web == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - name: Test Web
        run: npx turbo run test --filter=@myapp/web
```

`paths-filter` 액션으로 변경 감지를 별도 잡으로 분리하면 이후 잡들이 조건부로 실행된다. 변경이 없는 앱은 CI 시간을 소비하지 않는다.

---

## 정리

모노레포는 단순히 코드를 한 곳에 두는 것이 아니다.

공유 타입과 스키마로 프론트-백 계약을 컴파일 타임에 강제하고, 독립 배포 파이프라인으로 변경 범위를 최소화하며, Turborepo나 Nx의 캐시와 필터로 빌드 속도를 유지하는 것이 핵심이다.

서버 엔지니어 입장에서 가장 큰 이점은 API 타입 변경이 프론트엔드에 즉시 반영된다는 점이다. "백엔드가 바꿨는데 프론트가 몰랐다"는 종류의 사고가 구조적으로 불가능해진다.

진입 비용은 있다. 초기 레포 구조 설계, 빌드 도구 학습, CI 파이프라인 구성에 시간이 든다. 하지만 팀 규모가 커지고 앱이 늘어날수록 분리된 레포지토리에서 오는 타입 불일치, 버전 드리프트, 배포 조율 비용이 훨씬 크다.

---

## 참고 자료

- [Turborepo 공식 문서](https://turbo.build/repo/docs)
- [Nx 공식 문서](https://nx.dev/getting-started/intro)
- [Turborepo — Remote Caching](https://turbo.build/repo/docs/core-concepts/remote-caching)
- [Turborepo — Filtering Packages](https://turbo.build/repo/docs/core-concepts/monorepos/filtering)
- [Zod 공식 문서](https://zod.dev)
- [GitHub Actions — paths filter](https://github.com/dorny/paths-filter)
- [NestJS ConfigModule](https://docs.nestjs.com/techniques/configuration)
- [Next.js 환경 변수](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables)
- [Changesets — 모노레포 버저닝](https://github.com/changesets/changesets)
