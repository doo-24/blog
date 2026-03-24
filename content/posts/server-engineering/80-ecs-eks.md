---
title: "[AWS / 클라우드] 3편 — 컨테이너 서비스: ECS와 EKS 실전"
date: 2026-03-20T14:06:00+09:00
draft: false
tags: ["ECS", "EKS", "Fargate", "ECR", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "ECS 아키텍처(Task Definition, Service, Cluster)와 Fargate vs EC2 모드, EKS 구성과 관리형 노드 그룹, ECR 이미지 관리와 배포 파이프라인, ECS vs EKS 선택 기준까지"
---

## 목차

1. [왜 컨테이너 오케스트레이션이 필요한가](#왜-컨테이너-오케스트레이션이-필요한가)
2. [ECR — 컨테이너 이미지 저장소](#ecr--컨테이너-이미지-저장소)
3. [ECS 아키텍처 개요](#ecs-아키텍처-개요)
4. [Task Definition 상세](#task-definition-상세)
5. [ECS Service와 Cluster](#ecs-service와-cluster)
6. [Fargate vs EC2 모드](#fargate-vs-ec2-모드)
7. [EKS 구성과 관리형 노드 그룹](#eks-구성과-관리형-노드-그룹)
8. [EKS 핵심 개념 — Kubernetes 리소스 매핑](#eks-핵심-개념--kubernetes-리소스-매핑)
9. [배포 파이프라인 구성](#배포-파이프라인-구성)
10. [ECS vs EKS 선택 기준](#ecs-vs-eks-선택-기준)
11. [운영 관찰성 — 로그 · 메트릭 · 트레이싱](#운영-관찰성--로그--메트릭--트레이싱)
12. [비용 최적화 패턴](#비용-최적화-패턴)
13. [참고 자료](#참고-자료)

---

## 왜 컨테이너 오케스트레이션이 필요한가

도커 컨테이너 한두 개를 띄울 때는 `docker run` 명령어로 충분하다.
하지만 서비스 규모가 커져 수십, 수백 개의 컨테이너가 필요해지면 직접 관리하는 방식은 금방 한계에 부딪힌다.

컨테이너 오케스트레이션이 해결하는 문제는 크게 세 가지다.

- **자동 스케일링** — 트래픽에 따라 컨테이너 수를 늘리거나 줄인다.
- **자가 복구(Self-healing)** — 컨테이너가 죽으면 자동으로 재시작한다.
- **서비스 디스커버리** — 컨테이너 IP가 바뀌어도 다른 서비스가 찾을 수 있다.

AWS는 이 문제를 두 가지 서비스로 해결한다: **ECS(Elastic Container Service)**와 **EKS(Elastic Kubernetes Service)**.

> **핵심 포인트**: ECS는 AWS 독자 오케스트레이터, EKS는 관리형 쿠버네티스다. 둘 다 같은 컨테이너 이미지를 실행하지만 운영 방식과 생태계가 다르다.

---

## ECR — 컨테이너 이미지 저장소

ECS와 EKS 모두 **ECR(Elastic Container Registry)**을 이미지 저장소로 사용한다.
Docker Hub 대신 ECR을 쓰면 IAM으로 접근 제어가 가능하고, 같은 리전에서 풀(pull)하면 데이터 전송 비용이 없다.

### 리포지토리 생성

```bash
# ECR 리포지토리 생성
aws ecr create-repository \
  --repository-name my-app \
  --region ap-northeast-2 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256
```

`scanOnPush=true`를 설정하면 이미지를 푸시할 때마다 취약점 스캔이 자동으로 실행된다.

### 이미지 빌드 및 푸시

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS \
    --password-stdin 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드 및 태그
docker build -t my-app:latest .
docker tag my-app:latest \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:v1.0.0

# 푸시
docker push 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:v1.0.0
```

### 이미지 수명 주기 정책

오래된 이미지를 방치하면 스토리지 비용이 쌓인다.
라이프사이클 정책으로 자동 정리 규칙을 설정할 수 있다.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "최근 10개 이미지만 유지",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "untagged 이미지 7일 후 삭제",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## ECS 아키텍처 개요

ECS는 세 가지 핵심 개념으로 구성된다.

| 개념 | 역할 | 비유 |
|------|------|------|
| **Task Definition** | 컨테이너 실행 명세 | 쿠버네티스의 Pod Spec |
| **Task** | Task Definition의 실행 인스턴스 | 실제로 돌아가는 컨테이너 |
| **Service** | Task를 원하는 수만큼 유지·관리 | Kubernetes Deployment |
| **Cluster** | Service와 Task의 논리적 그룹 | Kubernetes Namespace + Node Pool |

ECS는 AWS가 자체 개발한 오케스트레이터라 쿠버네티스보다 개념이 훨씬 단순하다.
학습 곡선이 낮고, AWS 서비스와의 통합이 자연스럽다.

---

## Task Definition 상세

Task Definition은 "이 컨테이너를 어떻게 실행할 것인가"를 정의하는 JSON 문서다.
버전 관리가 되며, 변경할 때마다 새 리비전이 생성된다.

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:v1.0.0",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "APP_ENV", "value": "production" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### 주요 필드 설명

**`networkMode: awsvpc`**는 각 Task에 독립적인 ENI(네트워크 인터페이스)를 부여한다.
보안 그룹을 Task 단위로 적용할 수 있어 Fargate에서는 필수 설정이다.

**`executionRoleArn`**은 ECS 에이전트가 ECR에서 이미지를 풀하거나 CloudWatch에 로그를 쓸 때 사용하는 IAM 역할이다.
**`taskRoleArn`**은 컨테이너 내 애플리케이션이 S3, DynamoDB 등에 접근할 때 사용하는 별개의 IAM 역할이다.

**`secrets`** 필드를 통해 Secrets Manager나 Parameter Store의 값을 환경 변수로 주입할 수 있다.
비밀 정보를 Task Definition에 평문으로 넣지 않아도 된다.

---

## ECS Service와 Cluster

### Service 생성

Service는 "이 Task Definition으로 항상 N개의 Task를 실행하라"는 선언이다.
Task가 죽으면 자동으로 새 Task를 시작한다.

```bash
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app-service \
  --task-definition my-app:3 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-abc123,subnet-def456],
    securityGroups=[sg-xyz789],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,
    containerName=my-app,containerPort=8080" \
  --deployment-configuration "minimumHealthyPercent=50,maximumPercent=200"
```

**`minimumHealthyPercent=50, maximumPercent=200`**의 의미를 풀어보면:

- 2개 Task가 desired count일 때, 최소 1개는 항상 실행 중이어야 한다.
- 배포 중에 최대 4개까지 동시에 실행할 수 있다.
- 즉, 새 버전 2개를 올리면서 구 버전 2개가 아직 살아있을 수 있다.

이 설정이 **롤링 업데이트(Rolling Update)**의 핵심 파라미터다.

### Auto Scaling 연결

```bash
# 스케일링 대상 등록
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# CPU 기반 스케일링 정책
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

CPU 70%를 기준으로 자동으로 Task 수를 조절한다.
Scale-in(축소)의 쿨다운을 Scale-out(확장)보다 길게 잡는 것이 일반적인 패턴이다.

---

## Fargate vs EC2 모드

ECS(그리고 EKS도)는 실제 컨테이너를 실행하는 인프라 계층을 선택할 수 있다.

### Fargate 모드

Fargate는 서버리스 컨테이너 실행 환경이다.
EC2 인스턴스를 직접 관리할 필요 없이 "CPU 0.5 vCPU, 메모리 1 GB의 컨테이너를 실행해줘"라고 선언만 하면 된다.

**장점:**
- EC2 인스턴스 패치, 보안 관리 불필요
- Task 단위 과금 (사용한 CPU·메모리 초 단위 청구)
- 노드 수준 보안 격리 강화

**단점:**
- EC2 대비 단가가 비쌈 (약 20~30% 더 비싼 경우 많음)
- 일부 고성능 GPU, 특수 인스턴스 유형 사용 불가
- Cold Start 시간이 EC2보다 약간 길 수 있음

### EC2 모드

EC2 인스턴스를 직접 관리하면서 그 위에 ECS 에이전트를 실행하는 방식이다.
ECS 에이전트가 Task를 스케줄링하고 컨테이너 상태를 보고한다.

**장점:**
- Reserved Instance, Savings Plans 활용 가능 → 비용 절감
- GPU 인스턴스, 고성능 로컬 스토리지 사용 가능
- 더 세밀한 리소스 제어

**단점:**
- EC2 인스턴스 패치, 업그레이드 직접 관리
- Cluster 용량 계획 필요 (인스턴스 수 조절)
- Bin-packing 전략 고민 필요 (인스턴스 자원 낭비 최소화)

### 선택 기준 요약

| 기준 | Fargate 선택 | EC2 선택 |
|------|-------------|----------|
| 팀 역량 | 인프라 관리 경험 적음 | 운영 경험 풍부 |
| 워크로드 | 트래픽 패턴 불규칙 | 안정적·예측 가능 |
| 비용 | 최적화 우선순위 낮음 | 비용 최소화 중요 |
| 특수 요구사항 | 없음 | GPU, 특수 스토리지 필요 |

---

## EKS 구성과 관리형 노드 그룹

EKS는 쿠버네티스 컨트롤 플레인(API Server, etcd, Scheduler 등)을 AWS가 관리해주는 서비스다.
사용자는 워커 노드(데이터 플레인)만 관리하면 된다.

### 클러스터 생성

`eksctl`이 EKS 관리에 가장 널리 쓰이는 CLI 도구다.

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: ap-northeast-2
  version: "1.29"

vpc:
  cidr: 10.0.0.0/16

managedNodeGroups:
  - name: general
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 50
    ssh:
      allow: false
    labels:
      role: general
    tags:
      nodegroup-type: general
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        albIngress: true

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
```

```bash
eksctl create cluster -f cluster.yaml
```

### 관리형 노드 그룹 vs 자체 관리 노드

**관리형 노드 그룹(Managed Node Group)**은 AWS가 노드의 프로비저닝, 업그레이드, 드레이닝을 자동화해준다.
자체 관리 노드는 Auto Scaling Group을 직접 구성하고 업그레이드도 수동으로 처리해야 한다.

특별한 이유가 없으면 관리형 노드 그룹을 쓰는 것이 좋다.
노드 업그레이드 시 자동으로 Cordon → Drain → 교체 → Uncordon 과정을 처리해준다.

### Fargate Profile (EKS + Fargate)

EKS에서도 Fargate를 사용할 수 있다.
특정 네임스페이스나 레이블의 Pod을 Fargate에서 실행하도록 프로파일을 정의한다.

```bash
eksctl create fargateprofile \
  --cluster my-cluster \
  --name fp-default \
  --namespace default \
  --namespace kube-system
```

---

## EKS 핵심 개념 — Kubernetes 리소스 매핑

ECS 개념에 익숙하다면 쿠버네티스 리소스와 매핑해서 이해하면 빠르다.

| ECS | Kubernetes(EKS) | 역할 |
|-----|-----------------|------|
| Task Definition | Pod Spec | 컨테이너 실행 명세 |
| Task | Pod | 실행 중인 컨테이너 인스턴스 |
| Service | Deployment + Service | 복제본 관리 + 네트워크 노출 |
| Cluster | Cluster | 전체 오케스트레이션 단위 |
| Target Group | Ingress / Service | 트래픽 라우팅 |

### Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:v1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: APP_ENV
              value: production
          envFrom:
            - secretRef:
                name: my-app-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### AWS Load Balancer Controller

EKS에서 ALB를 Ingress로 사용하려면 AWS Load Balancer Controller를 설치해야 한다.

```bash
# Helm으로 설치
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
# Ingress 예시
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 8080
```

---

## 배포 파이프라인 구성

### ECS 배포 파이프라인

ECS에서 가장 일반적인 CI/CD 흐름은 다음과 같다.

```
코드 커밋 → CodePipeline → CodeBuild (이미지 빌드/푸시) → ECS 서비스 업데이트
```

CodeBuild의 `buildspec.yml` 예시:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION
          | docker login --username AWS
            --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/my-app
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)

  build:
    commands:
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:latest

  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo Writing image definitions file...
      - printf '[{"name":"my-app","imageUri":"%s"}]'
          $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files: imagedefinitions.json
```

`imagedefinitions.json`은 ECS 배포 단계에서 어떤 컨테이너에 어떤 이미지를 사용할지 지정하는 파일이다.

### GitHub Actions를 이용한 ECS 배포

```yaml
name: Deploy to ECS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-app
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download task definition
        run: |
          aws ecs describe-task-definition --task-definition my-app \
            --query taskDefinition > task-definition.json

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: my-app
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: my-app-service
          cluster: my-cluster
          wait-for-service-stability: true
```

### EKS 배포 — kubectl을 이용한 롤링 업데이트

```bash
# 이미지 업데이트 (가장 단순한 방법)
kubectl set image deployment/my-app \
  my-app=123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:v1.1.0 \
  -n production

# 롤아웃 상태 확인
kubectl rollout status deployment/my-app -n production

# 롤백이 필요하면
kubectl rollout undo deployment/my-app -n production
```

프로덕션에서는 `kubectl set image`보다 GitOps 방식이 권장된다.
**ArgoCD**나 **Flux**를 사용하면 Git 리포지토리가 클러스터 상태의 단일 진실 원천(Single Source of Truth)이 된다.

---

## ECS vs EKS 선택 기준

이 둘을 선택하는 기준은 기술적 요인보다 **팀 상황**과 **조직 목표**에 따라 결정되는 경우가 많다.

### ECS를 선택해야 할 때

**팀이 쿠버네티스 경험이 없고 빠르게 시작해야 한다면** ECS가 낫다.
ECS는 개념이 단순하고, AWS 콘솔에서 대부분의 작업을 직관적으로 처리할 수 있다.

**AWS 서비스와의 깊은 통합이 필요하다면** ECS가 유리하다.
Service Connect, App Mesh, CodeDeploy Blue/Green 배포 등이 네이티브로 지원된다.

**소규모 마이크로서비스(10개 미만)**를 운영한다면 ECS의 단순함이 오히려 장점이다.
쿠버네티스의 복잡한 생태계를 관리하는 오버헤드 없이 목표에 집중할 수 있다.

### EKS를 선택해야 할 때

**쿠버네티스 경험이 있는 팀**이라면 EKS가 자연스럽다.
기존 Helm 차트, 쿠버네티스 생태계 도구(Prometheus, Grafana, ArgoCD 등)를 그대로 사용할 수 있다.

**멀티 클라우드 전략**을 고려한다면 쿠버네티스 기반인 EKS가 유리하다.
애플리케이션을 GKE(GCP)나 AKS(Azure)로 이전하기 훨씬 쉽다.

**복잡한 배포 전략**(카나리, 트래픽 분할, 서킷 브레이커 등)이 필요하다면 Istio나 Linkerd 같은 서비스 메시를 쉽게 붙일 수 있는 EKS가 적합하다.

**100개 이상의 마이크로서비스**를 운영하는 대규모 환경에서는 쿠버네티스의 강력한 리소스 관리와 생태계가 빛을 발한다.

### 정리

```
간단하게 시작 → ECS (특히 Fargate)
AWS 일체형 운영 → ECS
이식성 + 생태계 → EKS
대규모 복잡한 서비스 → EKS
```

> **현실적 조언**: 어느 서비스를 써도 잘 동작하는 아키텍처를 만들 수 있다. 기술 선택보다 **일관된 운영 방식**과 **자동화 수준**이 더 중요하다.

---

## 운영 관찰성 — 로그 · 메트릭 · 트레이싱

### ECS 로그 수집

Task Definition에서 `awslogs` 드라이버를 사용하면 컨테이너 로그가 CloudWatch Logs로 자동 전송된다.

```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/my-app",
    "awslogs-region": "ap-northeast-2",
    "awslogs-stream-prefix": "ecs",
    "awslogs-datetime-format": "%Y-%m-%dT%H:%M:%S"
  }
}
```

로그 볼륨이 크다면 FireLens(Fluent Bit)를 사이드카 컨테이너로 사용해 S3나 OpenSearch로 직접 보내는 방식도 일반적이다.

### EKS 로그 수집

EKS에서는 Fluent Bit DaemonSet을 배포해 노드의 모든 컨테이너 로그를 수집한다.

```bash
# AWS for Fluent Bit DaemonSet 설치
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
```

### Container Insights

CloudWatch Container Insights는 ECS와 EKS 모두에서 컨테이너 레벨 메트릭을 자동으로 수집한다.
CPU, 메모리, 네트워크 사용량을 Task/Pod 단위로 시각화할 수 있다.

```bash
# EKS에 Container Insights 활성화
eksctl utils update-cluster-logging \
  --enable-types=api,audit,authenticator,controllerManager,scheduler \
  --region ap-northeast-2 \
  --cluster my-cluster \
  --approve
```

### X-Ray 분산 트레이싱

서비스 간 호출을 추적하려면 X-Ray를 사이드카로 붙인다.
ECS에서는 Task Definition에 X-Ray 데몬 컨테이너를 추가하고, EKS에서는 DaemonSet으로 배포한다.

```json
{
  "name": "xray-daemon",
  "image": "amazon/aws-xray-daemon",
  "portMappings": [
    {
      "containerPort": 2000,
      "protocol": "udp"
    }
  ],
  "cpu": 32,
  "memoryReservation": 256
}
```

---

## 비용 최적화 패턴

### Spot 인스턴스 활용 (ECS EC2 모드)

배치 작업이나 중단 허용 가능한 워크로드에는 Spot 인스턴스를 활용해 최대 70~90% 비용을 절감할 수 있다.

```json
{
  "capacityProviders": ["FARGATE", "FARGATE_SPOT"],
  "defaultCapacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4,
      "base": 0
    },
    {
      "capacityProvider": "FARGATE",
      "weight": 1,
      "base": 1
    }
  ]
}
```

이 설정은 최소 1개 Task는 일반 Fargate로 유지하고, 나머지 확장분의 80%는 Fargate Spot으로 처리한다.

### EKS Karpenter

Karpenter는 EKS의 차세대 노드 오토스케일러다.
기존 Cluster Autoscaler보다 빠르게 노드를 프로비저닝하고, 워크로드에 맞는 인스턴스 유형을 자동으로 선택한다.

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5.large", "m5.xlarge", "m5.2xlarge",
               "m5a.large", "m5a.xlarge"]
  limits:
    resources:
      cpu: 1000
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
```

### 스케줄 기반 스케일링

트래픽이 낮은 야간/주말에는 Task/Pod 수를 줄인다.
ECS에서는 EventBridge(CloudWatch Events) + Lambda로 스케줄 기반 스케일링을 구현한다.

```bash
# 평일 오전 9시에 desired count를 10으로 확장
aws events put-rule \
  --name scale-out-morning \
  --schedule-expression "cron(0 0 ? * MON-FRI *)" \
  --state ENABLED
```

---

## 핵심 포인트 요약

> **ECR**: 이미지 저장소. `scanOnPush`로 취약점 자동 스캔, 라이프사이클 정책으로 비용 절감.

> **ECS Task Definition**: 컨테이너 실행 명세. `executionRole`(ECR 접근)과 `taskRole`(앱 권한)을 반드시 분리.

> **Fargate**: 서버리스 컨테이너 실행. EC2 관리 불필요, 단가는 높음. Fargate Spot으로 최대 70% 절감 가능.

> **EKS 관리형 노드 그룹**: 노드 프로비저닝·업그레이드를 AWS가 자동화. 특수 요구 없으면 자체 관리 노드보다 우선.

> **ECS vs EKS**: 팀 경험과 규모가 핵심. 소규모·AWS 일체형이면 ECS, 대규모·이식성·생태계가 필요하면 EKS.

> **배포 파이프라인**: ECS는 `imagedefinitions.json` 기반 CodePipeline이 표준. EKS는 GitOps(ArgoCD/Flux)가 권장.

---

## 참고 자료

- [Amazon ECS 공식 문서](https://docs.aws.amazon.com/ecs/)
- [Amazon EKS 공식 문서](https://docs.aws.amazon.com/eks/)
- [Amazon ECR 공식 문서](https://docs.aws.amazon.com/ecr/)
- [AWS Fargate 공식 문서](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
- [eksctl 공식 문서](https://eksctl.io/)
- [Karpenter 공식 문서](https://karpenter.sh/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
