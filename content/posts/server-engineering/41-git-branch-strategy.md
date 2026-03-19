---
title: "[배포와 CI/CD] 1편 — Git 브랜치 전략: 팀이 코드를 합치는 규칙"
date: 2026-03-17T20:07:00+09:00
draft: false
tags: ["Git", "브랜치 전략", "Git Flow", "Trunk-Based", "서버"]
series: ["배포와 CI/CD"]
summary: "Git Flow, GitHub Flow, Trunk-Based Development 비교, 브랜치 전략 선택 기준(팀 규모, 릴리즈 주기), PR/MR 리뷰 프로세스와 머지 전략(merge, squash, rebase), 모노레포 vs 멀티레포 트레이드오프까지"
---

코드베이스는 살아있는 유기체다. 혼자 작업할 때는 `main` 하나로 충분하지만, 팀이 다섯 명을 넘어서는 순간부터 브랜치 전략 없이 개발하면 매일 오후 4시에 머지 지옥이 펼쳐진다. "누가 내 코드 위에 커밋했어?", "release 브랜치가 두 개인데 어느 게 맞아?", "핫픽스를 어디에 반영해야 해?"라는 질문이 Slack에 쏟아진다. 브랜치 전략은 단순한 Git 규칙이 아니다. 팀이 얼마나 빠르게, 얼마나 안전하게 코드를 프로덕션에 올릴 수 있는지를 결정하는 배포 철학이다.

---

## Git Flow: 계획적인 릴리즈를 위한 구조

Git Flow는 2010년 Vincent Driessen이 제안한 브랜치 모델로, 지금도 많은 팀이 사용한다. 핵심 아이디어는 브랜치의 역할을 명확히 분리하는 것이다.

### 브랜치 구조

```
main          ── 프로덕션 코드. 태그로 릴리즈 버전을 관리한다.
develop       ── 다음 릴리즈를 위한 통합 브랜치.
feature/*     ── 기능 개발. develop에서 분기하고 develop으로 머지.
release/*     ── 릴리즈 준비. develop에서 분기하고 main + develop으로 머지.
hotfix/*      ── 긴급 수정. main에서 분기하고 main + develop으로 머지.
```

```bash
# feature 브랜치 시작
git checkout develop
git checkout -b feature/user-authentication

# 개발 완료 후 develop으로 머지
git checkout develop
git merge --no-ff feature/user-authentication
git branch -d feature/user-authentication

# release 브랜치 생성
git checkout develop
git checkout -b release/1.2.0

# release에서 버그 수정 후 main과 develop 모두에 머지
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0

# 핫픽스
git checkout main
git checkout -b hotfix/critical-payment-bug
# 수정 후
git checkout main
git merge --no-ff hotfix/critical-payment-bug
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git checkout develop
git merge --no-ff hotfix/critical-payment-bug
```

### Git Flow의 강점

릴리즈 주기가 명확할 때 빛난다. 2주 스프린트로 QA → 스테이징 → 프로덕션 파이프라인을 운영하는 팀이라면, release 브랜치가 완충 지대가 된다. QA가 release/1.2.0에서 버그를 잡는 동안 개발팀은 develop에서 다음 스프린트 작업을 계속할 수 있다.

버전 관리가 명시적이다. `git tag -a v2.3.1` 하나로 "이 커밋이 프로덕션에 올라간 것"이라는 불변의 기록을 남긴다. 6개월 뒤 "2.3.1 버전에 무슨 문제가 있었나요?"라는 질문에 즉시 답할 수 있다.

### Git Flow의 한계

develop과 main이 계속 분기한다. 장기간 운영하면 두 브랜치 사이의 diff가 커지고, 머지 충돌이 잦아진다. 더 근본적인 문제는 "continuously delivery"와 맞지 않는다는 점이다. 프로덕션에 배포하려면 반드시 release 브랜치를 거쳐야 하므로 핫픽스가 아닌 이상 즉각 배포가 어렵다.

### Anti-pattern: develop을 오래 묵히기

```bash
# 나쁜 예: feature 브랜치를 3주 동안 살려두기
git checkout -b feature/big-refactoring
# ... 3주 동안 200개 커밋 ...
git checkout develop
git merge feature/big-refactoring
# 결과: 수백 개의 충돌
```

feature 브랜치 수명은 1~2일이 이상적이다. 길어도 1주일을 넘기지 말자. 오래된 브랜치는 충돌의 온상이다.

---

## GitHub Flow: 단순함으로 빠르게

GitHub Flow는 Git Flow보다 훨씬 단순하다. 브랜치는 두 종류뿐이다: `main`과 작업 브랜치.

### 워크플로우

```
1. main에서 브랜치를 만든다.
2. 커밋한다.
3. PR을 연다.
4. 리뷰 후 main에 머지한다.
5. 즉시 배포한다.
```

```bash
# 모든 작업은 main에서 시작
git checkout main
git pull origin main
git checkout -b fix/login-redirect-bug

# 작업
git add .
git commit -m "fix: redirect to intended URL after login"
git push origin fix/login-redirect-bug

# GitHub에서 PR 생성, 리뷰, 머지
# main이 업데이트되면 즉시 배포 파이프라인 트리거
```

### GitHub Flow의 강점

"배포 가능한 main"이 핵심 원칙이다. main에 머지되는 순간 프로덕션에 나간다는 약속을 지키기 때문에, 팀 전체가 코드 품질에 긴장감을 유지한다. CI가 통과하지 않으면 머지할 수 없다는 규칙과 결합하면 강력하다.

복잡도가 없다. 신규 개발자가 "어느 브랜치에 PR 올려야 해요?"라고 물을 일이 없다. 항상 main이다.

### GitHub Flow의 한계

버전 관리가 없다. "2.3.1 버전을 롤백해줘"라는 요청에 커밋 해시를 뒤적여야 한다. 또한 feature flag 없이 미완성 기능이 main에 머지되면 프로덕션에 노출된다. GitHub Flow를 쓰는 팀은 feature flag를 반드시 함께 운용해야 한다.

다중 환경(스테이징, QA, 프로덕션)을 관리하기 어렵다. "스테이징은 됐는데 프로덕션은 안 돼"라는 상황에서 어떤 커밋이 어디에 있는지 추적하기 번거롭다.

---

## Trunk-Based Development: Google과 Facebook의 방식

Trunk-Based Development(TBD)는 모든 개발자가 단일 브랜치(trunk, 보통 `main`)에 하루에 여러 번 커밋하는 방식이다. Google, Facebook, Netflix가 이 방식을 사용한다.

### 두 가지 변형

**소규모 팀 (1~5명)**: 직접 main에 커밋한다.

```bash
# 직접 main에 푸시 (소규모 팀)
git checkout main
git pull
# 짧은 작업 후 즉시 커밋
git commit -m "feat: add request timeout to payment client"
git push origin main
```

**대규모 팀**: short-lived feature 브랜치를 사용하되, 수명을 하루 이내로 제한한다.

```bash
# short-lived 브랜치 (최대 1~2일)
git checkout -b feat/timeout-config
# 빠른 작업
git commit -m "feat: configurable timeout for external APIs"
git push origin feat/timeout-config
# PR 생성 → 자동 CI → 빠른 리뷰 → 머지
```

### Feature Flag와 함께

TBD의 핵심 도구는 feature flag다. 미완성 코드를 main에 머지하되, 실제 사용자에게는 비활성화한다.

```java
// Feature flag를 통한 미완성 기능 숨기기
public class PaymentService {
    private final FeatureFlag featureFlag;

    public PaymentResult process(PaymentRequest request) {
        if (featureFlag.isEnabled("new-payment-flow", request.getUserId())) {
            return newPaymentFlow(request);
        }
        return legacyPaymentFlow(request);
    }
}
```

```yaml
# LaunchDarkly, Unleash 등의 feature flag 설정
features:
  new-payment-flow:
    enabled: false
    rollout:
      percentage: 0
      whitelist:
        - "internal-test-users"
```

### TBD의 강점

머지 충돌이 거의 없다. 브랜치 수명이 짧으니 분기 폭이 작다. 지속적 통합이 진짜 "지속적"이 된다. "내 브랜치에서는 됐는데 main에서 안 돼"라는 말이 사라진다.

배포 빈도가 극적으로 높아진다. Google의 일부 팀은 하루에 수백 번 배포한다. 이게 가능한 이유는 각 변경이 작기 때문이다. 작은 변경은 롤백도 쉽다.

### TBD의 전제 조건

- CI 파이프라인이 5분 이내에 완료되어야 한다. 30분짜리 테스트는 TBD를 불가능하게 만든다.
- 포괄적인 자동화 테스트가 있어야 한다. 테스트 없이 매일 main에 커밋하는 것은 위험하다.
- 팀 전체가 작은 단위로 작업하는 습관이 있어야 한다. "1주일 작업을 하나의 PR로" 방식과는 맞지 않는다.

### Anti-pattern: Commit Freeze

```bash
# 나쁜 예: 배포 전날 커밋 금지
# "오늘은 코드 동결입니다, 내일 배포가 있어요"

# TBD에서는 이런 상황이 없어야 한다
# 배포는 항상 가능하고, 위험한 기능은 feature flag로 끈다
```

---

## 브랜치 전략 선택 기준

전략 선택은 팀의 현실에 맞아야 한다. 이론적으로 좋은 전략이 우리 팀에 최선이 아닐 수 있다.

### 팀 규모별 권장

| 팀 규모 | 권장 전략 | 이유 |
|---------|-----------|------|
| 1~3명 | GitHub Flow 또는 TBD | 오버헤드 없이 빠르게 |
| 4~10명 | GitHub Flow | 단순하되 PR 리뷰 문화 구축 |
| 10~30명 | Git Flow 또는 TBD + Feature Flag | 병렬 작업과 릴리즈 관리 필요 |
| 30명 이상 | TBD + Feature Flag + 강력한 CI | 충돌 최소화가 최우선 |

### 릴리즈 주기별 권장

```
배포 주기 < 1일:     TBD 또는 GitHub Flow
배포 주기 1일~1주:   GitHub Flow
배포 주기 1~4주:     Git Flow
배포 주기 > 4주:     Git Flow (단, 개선 필요 신호)
```

### 실무 체크리스트

브랜치 전략을 선택하기 전에 다음 질문에 답해보자.

```
□ 프로덕션 배포가 얼마나 자주 일어나는가?
□ QA가 별도 환경에서 며칠씩 테스트하는가?
□ 외부 릴리즈 일정(앱스토어 심사 등)이 있는가?
□ CI 파이프라인이 10분 이내로 완료되는가?
□ 자동화 테스트 커버리지가 70% 이상인가?
□ 팀 전체가 feature flag를 사용할 준비가 됐는가?
```

"QA가 1주일 테스트한다" → Git Flow
"하루에 여러 번 배포한다" → TBD
"중간 어딘가" → GitHub Flow

---

## PR/MR 리뷰 프로세스

브랜치 전략만큼 중요한 것이 PR 리뷰 프로세스다. 나쁜 리뷰 프로세스는 좋은 브랜치 전략을 망친다.

### PR 크기

PR이 크면 리뷰가 형식적이 된다. 2,000줄짜리 PR에 "LGTM"을 누르는 건 리뷰가 아니다.

```bash
# PR 크기 확인
git diff main...HEAD --stat

# 이상적인 PR 크기
# - 변경 파일: 5개 이하
# - 변경 라인: 400줄 이하
# - 하나의 논리적 변경

# 큰 작업을 작은 PR로 나누기
# 나쁜 예: feature/user-system (3,000줄, 3주 작업)
# 좋은 예:
#   feat/user-model       (DB 스키마 + 모델)
#   feat/user-repository  (데이터 접근 레이어)
#   feat/user-service     (비즈니스 로직)
#   feat/user-api         (HTTP 핸들러)
```

### PR 템플릿

```markdown
## 변경 사항
<!-- 무엇을 왜 바꿨는지 설명 -->

## 테스트 방법
<!-- 리뷰어가 직접 테스트할 수 있는 방법 -->
1. `curl -X POST /api/users -d '{"email": "test@example.com"}'`
2. 응답에 `id` 필드가 있어야 함

## 체크리스트
- [ ] 테스트 추가/수정
- [ ] API 문서 업데이트
- [ ] 마이그레이션 스크립트 포함 (DB 변경 시)
- [ ] 롤백 계획 확인

## 관련 이슈
Closes #123
```

### 리뷰 SLA

PR이 방치되면 브랜치가 오래 살아남아 충돌이 쌓인다. 팀 내 SLA를 정하자.

```
영업일 기준:
- 소규모 PR (< 100줄):   4시간 이내 초기 리뷰
- 일반 PR (100~400줄):   1영업일 이내
- 대규모 PR (> 400줄):   분할 요청 (원칙)
```

### 리뷰 품질

```python
# 나쁜 리뷰 댓글
# "여기 버그 있어요"

# 좋은 리뷰 댓글
# "이 코드는 `userId`가 null일 때 NullPointerException이 발생할 수 있습니다.
# `Optional.ofNullable(userId).orElseThrow(() -> new IllegalArgumentException(...))`
# 패턴을 사용하거나, 메서드 진입 시점에 null 체크를 추가하면 어떨까요?"

# 리뷰는 교육의 기회다. "왜"를 설명하고 대안을 제시하라.
```

---

## 머지 전략: Merge, Squash, Rebase

세 가지 머지 방식은 커밋 히스토리의 형태를 결정한다. 선택이 틀리면 `git log`가 쓰레기통이 된다.

### Merge Commit (--no-ff)

```bash
git merge --no-ff feature/user-auth
```

```
*   a1b2c3d Merge branch 'feature/user-auth'
|\
| * f1e2d3c feat: add JWT validation middleware
| * e4d5c6b feat: implement login endpoint
| * d7c8b9a feat: create user model
|/
* 9a8b7c6 prev commit on main
```

**언제 쓰나**: Git Flow의 feature → develop 머지. 브랜치의 존재가 의미 있을 때. "이 기능이 언제 통합됐나"를 알고 싶을 때.

**장점**: 브랜치 히스토리가 보존된다. `git log --graph`로 작업 흐름을 볼 수 있다.
**단점**: 히스토리가 복잡해 보인다. 작은 WIP 커밋이 그대로 남는다.

### Squash Merge

```bash
git merge --squash feature/user-auth
git commit -m "feat: user authentication with JWT"
```

```
* b2c3d4e feat: user authentication with JWT
* 9a8b7c6 prev commit on main
```

**언제 쓰나**: 브랜치의 커밋 히스토리가 지저분할 때. "WIP", "fix typo", "asdf" 같은 커밋을 정리하고 싶을 때.

**장점**: main 히스토리가 깔끔하다. 각 PR이 하나의 커밋으로 요약된다.
**단점**: 개별 커밋 단위 롤백이 불가능하다. `git bisect`를 쓸 때 granularity가 낮다.

### Rebase and Merge

```bash
git rebase main
git checkout main
git merge --ff-only feature/user-auth
```

```
* f1e2d3c feat: add JWT validation middleware
* e4d5c6b feat: implement login endpoint
* d7c8b9a feat: create user model
* 9a8b7c6 prev commit on main
```

**언제 쓰나**: 선형 히스토리를 원할 때. 각 커밋이 독립적으로 의미 있을 때.

**장점**: 히스토리가 선형이고 깨끗하다. `git bisect`가 잘 동작한다.
**단점**: 커밋 해시가 바뀐다. 공유 브랜치에서 rebase하면 다른 사람의 작업을 망친다.

### 팀 전략과 머지 전략 매핑

```
Git Flow:
  feature → develop:    --no-ff merge (브랜치 보존)
  develop → main:       --no-ff merge (릴리즈 보존)
  hotfix → main:        --no-ff merge + tag

GitHub Flow:
  feature → main:       squash merge (히스토리 정리)
  또는 rebase merge     (선형 유지)

TBD:
  short-lived → main:   squash merge
  (대부분 직접 main에 커밋)
```

### Protected Branch 설정

```yaml
# GitHub branch protection rules (.github/branch-protection.yml 참고)
# GitHub Settings > Branches > Branch protection rules

main:
  required_status_checks:
    strict: true                    # 최신 main 기준으로 CI 통과
    contexts:
      - "ci/build"
      - "ci/test"
      - "ci/lint"
  required_pull_request_reviews:
    required_approving_review_count: 2
    dismiss_stale_reviews: true     # 새 커밋 시 승인 초기화
    require_code_owner_reviews: true
  restrictions:
    push: []                        # 직접 push 금지
  enforce_admins: true
```

```bash
# CODEOWNERS 파일 (리뷰어 자동 지정)
# .github/CODEOWNERS

# 결제 관련 코드는 결제팀 리뷰 필수
/src/payment/          @payment-team
/src/billing/          @payment-team

# 인프라 코드는 SRE 팀 리뷰 필수
/infrastructure/       @sre-team
/terraform/            @sre-team

# API 스펙 변경은 백엔드 리드 리뷰 필수
/src/api/              @backend-lead @api-team
```

---

## 모노레포 vs 멀티레포

브랜치 전략 못지않게 중요한 결정이 "코드를 어디에 두느냐"다. 마이크로서비스 시대에 이 질문은 더욱 복잡해졌다.

### 멀티레포: 서비스별 독립 저장소

```
org/
├── user-service/        (GitHub 저장소 1)
├── payment-service/     (GitHub 저장소 2)
├── notification-service/(GitHub 저장소 3)
└── api-gateway/         (GitHub 저장소 4)
```

```bash
# 각 서비스가 독립적인 배포 주기를 가진다
cd payment-service
git log --oneline
# a1b2c3d feat: add Apple Pay support
# b2c3d4e fix: timeout handling for card verification
# 결제 서비스만의 릴리즈 히스토리
```

**장점**:
- 팀별 자율성. 결제팀은 알림팀의 배포를 기다릴 필요 없다.
- 명확한 소유권. 저장소 = 팀 = 서비스.
- 서비스별 독립적인 CI/CD 파이프라인.
- 저장소 크기가 작아 clone이 빠르다.

**단점**:
- cross-service 변경이 여러 PR로 분산된다. 인터페이스를 바꾸면 모든 소비자 서비스를 따로 업데이트해야 한다.
- 공통 라이브러리 관리가 어렵다. 버전 불일치 문제가 생긴다.
- 코드 재사용이 불편하다. "util 함수 하나 가져오려고 새 패키지를 npm에 올려야 해?"

### 모노레포: 단일 저장소에 모든 코드

```
monorepo/
├── services/
│   ├── user-service/
│   ├── payment-service/
│   └── notification-service/
├── packages/
│   ├── shared-types/
│   ├── common-utils/
│   └── ui-components/
├── infrastructure/
└── tools/
```

Google은 수십억 줄의 코드를 하나의 저장소에 관리한다. Meta, Twitter도 모노레포를 사용한다.

**장점**:
- atomic cross-service 커밋. 인터페이스 변경과 소비자 업데이트를 하나의 커밋으로.
- 코드 공유가 쉽다. `import { logger } from '@company/common-utils'`.
- 일관된 도구와 설정. ESLint, TypeScript 설정이 전체에 통일.
- 전체 시스템에 대한 가시성. 코드 탐색이 한 곳에서 이뤄진다.

**단점**:
- 저장소가 거대해진다. clone에 수십 분이 걸릴 수 있다.
- CI가 복잡해진다. "user-service만 바뀌었는데 payment-service CI도 돌아야 해?"
- 팀 간 결합도가 생긴다. 한 팀의 broken build가 전체에 영향을 줄 수 있다.

### 모노레포 도구

```bash
# Nx (JavaScript/TypeScript)
npx create-nx-workspace@latest my-workspace
cd my-workspace
nx generate @nx/node:app user-service
nx generate @nx/node:app payment-service
nx generate @nx/js:lib common-utils

# 영향받은 서비스만 테스트
nx affected --target=test --base=main

# Turborepo (JavaScript/TypeScript)
npx create-turbo@latest
# turbo.json으로 작업 파이프라인 정의
```

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**/*.ts", "test/**/*.ts"]
    },
    "deploy": {
      "dependsOn": ["test", "build"]
    }
  }
}
```

```bash
# Bazel (언어 독립적, Google 내부 도구 오픈소스)
# BUILD 파일로 의존성 그래프 명시
# 변경된 타겟만 빌드/테스트

# 변경사항이 영향을 미치는 서비스만 빌드
bazel build //services/... --output_filter=...
```

### CI 최적화: 변경된 서비스만 빌드

```yaml
# GitHub Actions - 변경된 서비스만 감지
name: CI

on: [push]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      user-service: ${{ steps.filter.outputs.user-service }}
      payment-service: ${{ steps.filter.outputs.payment-service }}
    steps:
      - uses: actions/checkout@v3
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            user-service:
              - 'services/user-service/**'
              - 'packages/common-utils/**'
            payment-service:
              - 'services/payment-service/**'
              - 'packages/common-utils/**'

  test-user-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.user-service == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: cd services/user-service && npm test

  test-payment-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.payment-service == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: cd services/payment-service && npm test
```

### 의사결정 가이드

```
팀이 2~5명이고 서비스가 2~3개다:
  → 모노레포. 복잡성보다 편의성이 중요.

팀이 독립적으로 배포해야 한다:
  → 멀티레포. 팀 간 결합도 최소화.

공통 라이브러리를 많이 공유한다:
  → 모노레포. 버전 지옥을 피하라.

서비스 간 API가 자주 바뀐다:
  → 모노레포. atomic cross-service 변경이 필수.

서비스 기술 스택이 완전히 다르다 (Python + Go + Java):
  → 멀티레포. 빌드 도구 통일이 어렵다.
```

### Anti-pattern: 혼합 전략

```
# 나쁜 예: 절반은 모노레포, 절반은 멀티레포
monorepo/
├── user-service/      ← 여기는 모노레포
└── common-utils/

payment-service-repo/  ← 여기는 멀티레포
notification-repo/     ← 여기도 멀티레포

# 결과:
# - common-utils를 npm 패키지로 배포해야 멀티레포에서 쓸 수 있다
# - 버전 관리가 이중으로 복잡해진다
# - "어디에 코드가 있지?"라는 질문이 끊이지 않는다
```

일관성이 중요하다. 섞으면 두 전략의 단점만 남는다.

---

## 실무에서 마주치는 시나리오

### 시나리오 1: 긴급 핫픽스

```bash
# Git Flow 팀의 핫픽스
# 문제: 프로덕션 결제 오류, 즉시 수정 필요

git checkout main
git pull origin main
git checkout -b hotfix/payment-null-pointer

# 수정
vim src/payment/PaymentProcessor.java

git add src/payment/PaymentProcessor.java
git commit -m "fix: null check for payment method in processor"

# 테스트 통과 확인
./mvnw test -pl payment-service

# main과 develop 모두에 반영
git checkout main
git merge --no-ff hotfix/payment-null-pointer
git tag -a v2.3.1 -m "Hotfix: payment null pointer"
git push origin main --tags

git checkout develop
git merge --no-ff hotfix/payment-null-pointer
git push origin develop

git branch -d hotfix/payment-null-pointer
```

### 시나리오 2: 대규모 리팩토링

```bash
# 데이터베이스 접근 레이어 전면 교체: JDBC → JPA

# 나쁜 접근: 하나의 거대한 PR
git checkout -b refactor/jdbc-to-jpa
# ... 2주간 수백 개 파일 수정 ...
# 머지 불가능한 PR 탄생

# 좋은 접근: 단계별 PR
# 1단계: 새 레이어 추가 (기존 코드 건드리지 않음)
git checkout -b refactor/add-jpa-layer
# JPA Repository 인터페이스 추가, 기존 JDBC 코드 유지
git commit -m "refactor: add JPA repositories alongside existing JDBC"

# 2단계: 서비스별 교체 (서비스 하나씩)
git checkout -b refactor/user-service-to-jpa
# UserService만 JPA로 교체
git commit -m "refactor: migrate UserService to JPA repository"

# 3단계: JDBC 코드 제거
git checkout -b refactor/remove-jdbc-layer
git commit -m "chore: remove deprecated JDBC repository implementations"
```

### 시나리오 3: 실험적 기능

```bash
# TBD + Feature Flag로 실험적 기능 배포

# 1. feature flag 설정
# Unleash 또는 LaunchDarkly에서 'new-recommendation-engine' 플래그 생성
# 초기값: 비활성화 (0% 롤아웃)

# 2. 코드 작성 후 main에 머지
git checkout -b feat/new-recommendation-engine
# 구현
git commit -m "feat: implement ML-based recommendation engine (behind flag)"
# PR → 리뷰 → main 머지 → 배포
# 사용자에게는 보이지 않음 (플래그 비활성화)

# 3. 단계적 롤아웃
# Unleash에서 롤아웃 비율 조정:
# 5% → 문제 없으면 → 25% → 50% → 100%

# 4. 문제 발생 시 즉시 비활성화 (코드 배포 없이)
# Unleash 대시보드에서 플래그를 false로 전환
# 수초 내 모든 사용자에게 이전 로직 적용
```

---

## 결론: 전략은 목적에 맞게

브랜치 전략에 정답은 없다. Git Flow를 쓴다고 나쁜 팀이 아니고, TBD를 쓴다고 좋은 팀이 아니다. 중요한 것은 전략이 팀의 배포 목표, 팀 규모, 기술 성숙도와 일치하는지다.

지금 당장 점검해야 할 것들:

1. **브랜치 수명**: feature 브랜치가 1주일 이상 살아있다면 경고 신호.
2. **머지 충돌 빈도**: 매일 충돌을 해결하고 있다면 전략을 재검토할 때.
3. **배포 빈도**: 한 달에 한 번 배포한다면 TBD는 무의미하다. 목표부터 바꿔야 한다.
4. **PR 리뷰 시간**: PR이 2일 이상 방치된다면 프로세스를 개선해야 한다.

팀이 성장하면 전략도 바뀐다. 5명 스타트업의 브랜치 전략을 50명 팀이 그대로 쓰면 안 된다. 6개월마다 "우리 전략이 여전히 우리 팀에 맞는가?"를 물어보자.

---

## 참고 자료

1. **Vincent Driessen, "A successful Git branching model" (2010)** — Git Flow의 원조 논문. https://nvie.com/posts/a-successful-git-branching-model/

2. **Paul Hammant, "Trunk Based Development" (trunkbaseddevelopment.com)** — TBD의 포괄적 가이드. 패턴, 안티패턴, 실제 사례. https://trunkbaseddevelopment.com/

3. **Accelerate: The Science of Lean Software and DevOps (Nicole Forsgren, Jez Humble, Gene Kim)** — 배포 빈도, 리드 타임, 브랜치 전략이 조직 성과에 미치는 영향을 데이터로 분석.

4. **GitHub Docs: "About pull request merges"** — squash, rebase, merge commit의 공식 설명과 트레이드오프. https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes/about-pull-request-merges

5. **Google Engineering Practices: "The Standard of Code Review"** — Google의 코드 리뷰 가이드라인, PR 크기와 리뷰 속도에 대한 실무 기준. https://google.github.io/eng-practices/review/

6. **Monorepo.tools** — Nx, Turborepo, Bazel, Lerna 등 모노레포 도구 비교. https://monorepo.tools/
