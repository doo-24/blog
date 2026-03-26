---
title: "[AWS 실전 구축] 6편 — AWS CI/CD 파이프라인: CodePipeline, CodeBuild, CodeDeploy"
date: 2026-03-17T11:03:00+09:00
draft: false
tags: ["CodePipeline", "CodeBuild", "CodeDeploy", "CI/CD", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "CodePipeline 구성(Source→Build→Deploy), CodeBuild buildspec.yml과 캐시 전략, CodeDeploy 배포 유형(In-Place, Blue/Green), ECS 블루/그린 배포 파이프라인 전체 구성까지"
---

코드를 작성하고 서버에 올리는 과정이 수동이면, 실수가 생기고 속도가 느려진다. AWS는 CodePipeline, CodeBuild, CodeDeploy 세 서비스를 조합해 소스 커밋부터 프로덕션 배포까지 완전 자동화된 파이프라인을 제공한다.

---

## 1. CI/CD 파이프라인 개념과 AWS 서비스 역할

CI/CD는 코드 변경이 자동으로 빌드되고 테스트되어 배포까지 이어지는 워크플로우다.

AWS에서는 세 서비스가 각각의 역할을 담당한다.

| 서비스 | 역할 |
|---|---|
| CodePipeline | 파이프라인 오케스트레이터. 단계 연결과 실행 흐름 제어 |
| CodeBuild | 빌드 및 테스트 실행 환경. 컨테이너 기반 빌드 워커 |
| CodeDeploy | 실제 배포 실행. EC2, ECS, Lambda 등 다양한 타깃 지원 |

세 서비스 없이도 GitHub Actions 같은 외부 도구를 쓸 수 있지만, AWS 네이티브 파이프라인은 IAM 통합, CloudWatch 로그 연동, 비용 가시성 면에서 장점이 있다.

---

## 2. CodePipeline 구성: Source → Build → Deploy

### 파이프라인 기본 구조

CodePipeline은 스테이지(Stage)와 액션(Action)으로 구성된다.

각 스테이지는 순서대로 실행되고, 스테이지 내 액션은 병렬 실행도 가능하다.

```
[Source Stage]
  └─ GitHub / CodeCommit / S3 에서 소스 감지

[Build Stage]
  └─ CodeBuild 프로젝트 실행 (빌드 + 테스트)

[Deploy Stage]
  └─ CodeDeploy / ECS / CloudFormation 배포
```

### GitHub 연동 설정 (AWS 콘솔 기준)

CodePipeline 생성 시 Source Provider를 `GitHub (Version 2)`로 선택한다.

GitHub Apps 방식으로 연결하면 OAuth 토큰 대신 GitHub App 설치 기반 인증을 사용하므로 보안이 강화된다.

```json
{
  "name": "Source",
  "actions": [
    {
      "name": "GitHub_Source",
      "actionTypeId": {
        "category": "Source",
        "owner": "AWS",
        "provider": "CodeStarSourceConnection",
        "version": "1"
      },
      "configuration": {
        "ConnectionArn": "arn:aws:codestar-connections:ap-northeast-2:123456789012:connection/abcd1234",
        "FullRepositoryId": "myorg/myrepo",
        "BranchName": "main",
        "OutputArtifactFormat": "CODE_ZIP",
        "DetectChanges": "true"
      },
      "outputArtifacts": [
        { "name": "SourceOutput" }
      ]
    }
  ]
}
```

`DetectChanges: true`로 설정하면 브랜치에 푸시할 때마다 파이프라인이 자동 트리거된다.

### 파이프라인 아티팩트 흐름

각 스테이지는 이전 스테이지의 출력 아티팩트를 입력으로 받는다.

아티팩트는 S3 버킷에 자동 저장되며, 파이프라인 생성 시 지정한 아티팩트 버킷을 사용한다.

```
SourceOutput (소스 ZIP) → BuildOutput (빌드 결과) → DeployOutput (배포 패키지)
```

---

## 3. CodeBuild: buildspec.yml 작성과 캐시 전략

### buildspec.yml 기본 구조

CodeBuild는 `buildspec.yml` 파일을 읽어 빌드를 실행한다.

파일이 없으면 CodeBuild 프로젝트 설정에서 인라인으로 정의할 수도 있지만, 버전 관리를 위해 소스 루트에 두는 것이 좋다.

```yaml
version: 0.2

env:
  variables:
    APP_ENV: production
  parameter-store:
    DB_PASSWORD: /myapp/db/password
  secrets-manager:
    API_KEY: myapp/api-key:api_key

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - echo "Installing dependencies..."
      - npm ci --prefer-offline

  pre_build:
    commands:
      - echo "Running tests..."
      - npm test
      - echo "Logging in to ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
          docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - echo "Build started at $(date)"
      - docker build -t $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION \
          $ECR_REGISTRY/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION \
          $ECR_REGISTRY/$IMAGE_REPO_NAME:latest

  post_build:
    commands:
      - echo "Pushing image to ECR..."
      - docker push $ECR_REGISTRY/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker push $ECR_REGISTRY/$IMAGE_REPO_NAME:latest
      - printf '[{"name":"app","imageUri":"%s"}]' \
          $ECR_REGISTRY/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION \
          > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
```

`CODEBUILD_RESOLVED_SOURCE_VERSION`은 CodeBuild가 자동으로 주입하는 환경 변수로, 소스 커밋 해시값이다.

이 값을 이미지 태그로 사용하면 어떤 커밋이 어떤 이미지에 대응하는지 추적이 가능하다.

### 환경 변수와 시크릿 관리

민감한 값은 buildspec.yml에 직접 쓰지 않는다.

SSM Parameter Store와 Secrets Manager 두 가지 방법으로 주입할 수 있다.

```yaml
env:
  parameter-store:
    # SSM Parameter Store의 SecureString 파라미터
    DB_HOST: /myapp/prod/db_host
    DB_PASSWORD: /myapp/prod/db_password
  secrets-manager:
    # Secrets Manager의 시크릿 (secretId:jsonKey 형식)
    STRIPE_KEY: myapp/stripe:secret_key
```

CodeBuild 프로젝트의 IAM 역할에 `ssm:GetParameters`, `secretsmanager:GetSecretValue` 권한이 있어야 한다.

### 캐시 전략

캐시를 쓰지 않으면 매 빌드마다 의존성을 새로 내려받아 빌드 시간이 길어진다.

CodeBuild는 S3 캐시와 로컬 캐시 두 가지를 제공한다.

**S3 캐시 설정 (buildspec.yml)**

```yaml
cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
    - '/root/.m2/**/*'
```

S3 캐시는 첫 빌드 후 지정 경로를 버킷에 업로드하고, 다음 빌드 시 내려받아 복원한다.

**로컬 캐시 (CodeBuild 프로젝트 설정)**

```json
{
  "cache": {
    "type": "LOCAL",
    "modes": [
      "LOCAL_SOURCE_CACHE",
      "LOCAL_DOCKER_LAYER_CACHE",
      "LOCAL_CUSTOM_CACHE"
    ]
  }
}
```

`LOCAL_DOCKER_LAYER_CACHE`는 Docker 레이어를 로컬에 캐시해 이미지 빌드 시간을 크게 줄인다.

같은 CodeBuild 인스턴스가 재사용될 때만 동작하므로, 빌드 빈도가 높을수록 효과적이다.

### ECR 이미지 빌드 권한 설정

CodeBuild 프로젝트의 IAM 역할에 ECR 접근 권한이 필요하다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-2:123456789012:repository/myapp"
    }
  ]
}
```

Docker 빌드를 사용하려면 CodeBuild 프로젝트의 환경 설정에서 "Privileged mode" 를 활성화해야 한다.

---

## 4. CodeDeploy: 배포 유형과 AppSpec

### 배포 유형 비교

CodeDeploy는 크게 두 가지 배포 방식을 지원한다.

| 배포 유형 | 설명 | 다운타임 |
|---|---|---|
| In-Place | 기존 인스턴스에서 구버전 중지 → 신버전 설치 | 있음 |
| Blue/Green | 신규 인스턴스 그룹 생성 후 트래픽 전환 | 없음 (순간 전환) |

프로덕션 환경에서는 Blue/Green이 권장된다. 롤백 시 트래픽만 원래 그룹으로 되돌리면 되기 때문이다.

### EC2/온프레미스 AppSpec (appspec.yml)

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/myapp

permissions:
  - object: /var/www/myapp
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - directory

hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 180
      runas: ec2-user
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: ec2-user
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 180
      runas: ec2-user
  ValidateService:
    - location: scripts/validate.sh
      timeout: 60
      runas: ec2-user
```

훅(hooks)은 배포 라이프사이클의 특정 시점에 스크립트를 실행할 수 있게 한다.

`ValidateService` 훅에서 헬스체크 스크립트를 실행해 배포 성공 여부를 확인하는 패턴이 일반적이다.

### 배포 구성 (Deployment Configuration)

배포 구성은 한 번에 몇 개의 인스턴스에 배포할지를 정의한다.

```
AllAtOnce  : 모든 인스턴스에 동시 배포 (빠르지만 위험)
HalfAtATime: 절반씩 순차 배포
OneAtATime : 한 번에 하나씩 (느리지만 안전)
Custom     : 사용자 정의 비율
```

사용자 지정 구성 예시 (CloudFormation):

```yaml
MyDeploymentConfig:
  Type: AWS::CodeDeploy::DeploymentConfig
  Properties:
    DeploymentConfigName: MyCustomConfig
    MinimumHealthyHosts:
      Type: FLEET_PERCENT
      Value: 75
```

`FLEET_PERCENT: 75`는 전체 인스턴스의 75%는 항상 정상 상태를 유지하면서 배포한다는 의미다.

### 자동 롤백 설정

배포 실패나 알람 발생 시 자동으로 롤백하도록 설정할 수 있다.

```json
{
  "autoRollbackConfiguration": {
    "enabled": true,
    "events": [
      "DEPLOYMENT_FAILURE",
      "DEPLOYMENT_STOP_ON_ALARM",
      "DEPLOYMENT_STOP_ON_REQUEST"
    ]
  },
  "alarmConfiguration": {
    "enabled": true,
    "alarms": [
      { "name": "HighErrorRate" },
      { "name": "HighLatency" }
    ]
  }
}
```

`DEPLOYMENT_STOP_ON_ALARM`을 사용하면 CloudWatch 알람이 발생했을 때 자동으로 배포를 중단하고 이전 버전으로 복원한다.

---

## 5. ECS 블루/그린 배포 파이프라인 전체 구성

### 아키텍처 개요

ECS 블루/그린 배포는 CodeDeploy가 ALB 타깃 그룹을 두 개 관리하며 트래픽을 전환하는 방식이다.

```
[ALB]
  ├─ Listener (80/443)
  │     ├─ Production Rule → Blue Target Group (현재 버전)
  │     └─ Test Rule (8080) → Green Target Group (신버전 테스트용)
  │
  ├─ Blue Target Group ← Blue ECS Tasks
  └─ Green Target Group ← Green ECS Tasks (배포 후 신규)
```

배포 완료 후 CodeDeploy가 Production 리스너를 Green 타깃 그룹으로 전환하고, 기존 Blue 태스크는 지정된 시간 후 종료된다.

### 필요한 파일 구성

ECS 블루/그린 배포를 위해 세 개의 파일이 필요하다.

**1. taskdef.json** — ECS 태스크 정의

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "<IMAGE1_NAME>",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ]
    }
  ]
}
```

`<IMAGE1_NAME>`은 CodePipeline이 실제 이미지 URI로 교체하는 플레이스홀더다.

**2. appspec.yaml** — CodeDeploy ECS 배포 명세

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 3000
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "arn:aws:lambda:ap-northeast-2:123456789012:function:PreDeployCheck"
  - AfterAllowTestTraffic: "arn:aws:lambda:ap-northeast-2:123456789012:function:RunIntegrationTests"
  - BeforeAllowTraffic: "arn:aws:lambda:ap-northeast-2:123456789012:function:ValidateNewVersion"
  - AfterAllowTraffic: "arn:aws:lambda:ap-northeast-2:123456789012:function:PostDeployCleanup"
```

`AfterAllowTestTraffic` 훅에서 8080 포트로 신버전에 접근해 통합 테스트를 실행하는 패턴이 많이 쓰인다.

**3. imagedefinitions.json** — CodeBuild 출력, 이미지 정보

```json
[
  {
    "name": "app",
    "imageUri": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:abc1234"
  }
]
```

이 파일은 CodeBuild의 `post_build` 단계에서 `printf` 명령으로 생성해 아티팩트로 내보낸다.

### CodePipeline 전체 파이프라인 정의 (CloudFormation)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ECS Blue/Green CI/CD Pipeline

Resources:
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: myapp-pipeline
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket

      Stages:
        - Name: Source
          Actions:
            - Name: GitHub_Source
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeStarSourceConnection
                Version: "1"
              Configuration:
                ConnectionArn: !Ref GitHubConnection
                FullRepositoryId: myorg/myapp
                BranchName: main
                DetectChanges: "true"
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Build
          Actions:
            - Name: Docker_Build
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: "1"
              Configuration:
                ProjectName: !Ref CodeBuildProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput

        - Name: Deploy
          Actions:
            - Name: ECS_BlueGreen_Deploy
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CodeDeployToECS
                Version: "1"
              Configuration:
                ApplicationName: !Ref CodeDeployApp
                DeploymentGroupName: !Ref CodeDeployDeploymentGroup
                TaskDefinitionTemplateArtifact: BuildOutput
                TaskDefinitionTemplatePath: taskdef.json
                AppSpecTemplateArtifact: BuildOutput
                AppSpecTemplatePath: appspec.yaml
                Image1ArtifactName: BuildOutput
                Image1ContainerName: IMAGE1_NAME
              InputArtifacts:
                - Name: BuildOutput
```

`Image1ContainerName: IMAGE1_NAME`은 CodePipeline이 `taskdef.json`의 `<IMAGE1_NAME>` 플레이스홀더를 실제 이미지 URI로 교체하도록 지시한다.

### CodeDeploy 배포 그룹 설정 (CloudFormation)

```yaml
  CodeDeployDeploymentGroup:
    Type: AWS::CodeDeploy::DeploymentGroup
    Properties:
      ApplicationName: !Ref CodeDeployApp
      DeploymentGroupName: myapp-deployment-group
      ServiceRoleArn: !GetAtt CodeDeployRole.Arn
      DeploymentConfigName: CodeDeployDefault.ECSAllAtOnce
      DeploymentStyle:
        DeploymentType: BLUE_GREEN
        DeploymentOption: WITH_TRAFFIC_CONTROL
      BlueGreenDeploymentConfiguration:
        TerminateBlueInstancesOnDeploymentSuccess:
          Action: TERMINATE
          TerminationWaitTimeInMinutes: 5
        DeploymentReadyOption:
          ActionOnTimeout: CONTINUE_DEPLOYMENT
          WaitTimeInMinutes: 0
      LoadBalancerInfo:
        TargetGroupPairInfoList:
          - TargetGroups:
              - Name: myapp-blue-tg
              - Name: myapp-green-tg
            ProdTrafficRoute:
              ListenerArns:
                - !Ref ALBListener443
            TestTrafficRoute:
              ListenerArns:
                - !Ref ALBListener8080
      ECSServices:
        - ClusterName: myapp-cluster
          ServiceName: myapp-service
      AutoRollbackConfiguration:
        Enabled: true
        Events:
          - DEPLOYMENT_FAILURE
          - DEPLOYMENT_STOP_ON_ALARM
      AlarmConfiguration:
        Enabled: true
        Alarms:
          - Name: myapp-error-rate-alarm
```

`TerminationWaitTimeInMinutes: 5`는 트래픽 전환 후 5분간 기존(Blue) 태스크를 유지한다는 의미다.

문제가 발생하면 이 시간 안에 수동 롤백이 가능하다.

---

## 6. 파이프라인 모니터링과 알람 설정

### CloudWatch 이벤트로 파이프라인 상태 감지

파이프라인 실패 시 알림을 받으려면 EventBridge 규칙을 설정한다.

```json
{
  "source": ["aws.codepipeline"],
  "detail-type": ["CodePipeline Pipeline Execution State Change"],
  "detail": {
    "state": ["FAILED"],
    "pipeline": ["myapp-pipeline"]
  }
}
```

이 이벤트를 SNS 토픽으로 라우팅하면 Slack이나 이메일로 알림을 받을 수 있다.

### CodeBuild 빌드 알람

빌드 실패율이나 빌드 시간을 CloudWatch 메트릭으로 모니터링할 수 있다.

```yaml
  BuildDurationAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: myapp-build-duration-high
      MetricName: Duration
      Namespace: AWS/CodeBuild
      Dimensions:
        - Name: ProjectName
          Value: myapp-build
      Statistic: Average
      Period: 300
      EvaluationPeriods: 3
      Threshold: 600
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref AlertSNSTopic
```

빌드 시간이 지속적으로 늘어나면 캐시 설정 문제나 의존성 증가 신호일 수 있다.

---

## 7. 실무 운영 팁

**멀티 환경 파이프라인 구성**

스테이징과 프로덕션을 별도 스테이지로 구성할 때 수동 승인 단계를 추가한다.

```yaml
        - Name: Approve_Production
          Actions:
            - Name: Manual_Approval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: "1"
              Configuration:
                NotificationArn: !Ref ApprovalSNSTopic
                CustomData: "프로덕션 배포 승인 필요. 스테이징 검증 완료 후 승인하세요."
                ExternalEntityLink: "https://staging.myapp.com"
```

**파이프라인 실행 조건 제한**

모든 브랜치 푸시가 파이프라인을 트리거하면 불필요한 빌드가 많아진다.

`main` 브랜치 머지만 트리거하고, PR은 별도 CodeBuild 프로젝트로 PR 체크만 수행하는 방식을 권장한다.

**아티팩트 버킷 수명 주기 정책**

파이프라인 아티팩트는 S3에 무제한 쌓일 수 있다.

수명 주기 정책으로 30일 이상된 아티팩트를 자동 삭제하거나 Glacier로 이동하도록 설정하면 비용을 절감할 수 있다.

```yaml
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      LifecycleConfiguration:
        Rules:
          - Status: Enabled
            ExpirationInDays: 30
```

---

## 정리

CodePipeline은 소스, 빌드, 배포 단계를 연결하는 오케스트레이터 역할을 하고, CodeBuild는 buildspec.yml 기반으로 Docker 이미지 빌드와 테스트를 수행한다.

CodeDeploy는 In-Place와 Blue/Green 두 방식으로 배포를 실행하며, ECS 블루/그린 배포는 ALB 타깃 그룹 전환으로 무중단 배포를 구현한다.

세 서비스를 연결한 파이프라인에 자동 롤백과 CloudWatch 알람을 붙이면, 코드 커밋부터 프로덕션 배포까지 안정적이고 추적 가능한 자동화 워크플로우가 완성된다.

---

## 참고 자료

- [AWS CodePipeline 공식 문서](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)
- [AWS CodeBuild buildspec 레퍼런스](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [AWS CodeDeploy ECS 블루/그린 배포](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html)
- [CodePipeline + ECS 블루/그린 튜토리얼](https://docs.aws.amazon.com/codepipeline/latest/userguide/ecs-cd-pipeline.html)
- [CodeBuild 로컬 캐시](https://docs.aws.amazon.com/codebuild/latest/userguide/build-caching.html)
