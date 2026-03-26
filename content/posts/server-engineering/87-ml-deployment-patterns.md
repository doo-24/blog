---
title: "[AI / ML 서빙] 2편 — 모델 배포 패턴: 안전하게 모델을 교체하는 법"
date: 2026-03-17T13:06:00+09:00
draft: false
tags: ["모델 배포", "A/B 테스트", "카나리", "MLflow", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "모델 버저닝과 레지스트리(MLflow Model Registry, SageMaker), Shadow 배포·A/B 테스트·카나리 배포의 모델 전용 구현, 멀티 모델 서빙과 앙상블 패턴, 롤백 전략과 모델 성능 회귀 자동 탐지까지"
---

ML 모델을 프로덕션에 배포하는 일은 일반 서비스 배포와 다르다. 코드 버그는 로그로 잡히지만, 모델 성능 저하는 메트릭을 보기 전까지 보이지 않는다.

트래픽을 전부 넘긴 후에야 새 모델이 특정 입력 분포에서 오작동한다는 사실을 알게 되는 경우가 흔하다.

이 글은 그런 상황을 막기 위한 배포 패턴들을 실제 구현 코드와 함께 정리한다.

---

## 1. 모델 버저닝과 레지스트리

### MLflow Model Registry

MLflow는 실험 추적(Experiment Tracking)과 모델 레지스트리를 통합 제공한다.

학습이 끝난 모델을 레지스트리에 등록하는 코드는 다음과 같다.

```python
import mlflow
import mlflow.sklearn
from mlflow.tracking import MlflowClient

mlflow.set_tracking_uri("http://mlflow-server:5000")
client = MlflowClient()

with mlflow.start_run() as run:
    # 학습 로직
    model = train_model(X_train, y_train)

    # 메트릭 기록
    mlflow.log_metric("accuracy", evaluate(model, X_val, y_val))
    mlflow.log_metric("f1_score", f1(model, X_val, y_val))

    # 모델 로깅
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        registered_model_name="fraud-detector",
        input_example=X_val[:5],
        signature=mlflow.models.infer_signature(X_val, model.predict(X_val)),
    )

run_id = run.info.run_id
```

레지스트리에는 모델 버전이 쌓인다. 각 버전은 `None`, `Staging`, `Production`, `Archived` 중 하나의 스테이지를 갖는다.

```python
# 최신 버전을 Staging으로 전환
model_version = client.get_latest_versions("fraud-detector", stages=["None"])[0]

client.transition_model_version_stage(
    name="fraud-detector",
    version=model_version.version,
    stage="Staging",
    archive_existing_versions=False,
)

# Staging 모델 로드 및 검증
staging_model = mlflow.sklearn.load_model(
    f"models:/fraud-detector/Staging"
)
```

검증이 끝나면 Production으로 승격한다.

```python
client.transition_model_version_stage(
    name="fraud-detector",
    version=model_version.version,
    stage="Production",
    archive_existing_versions=True,  # 기존 Production 버전을 Archived로 전환
)
```

### 모델 승인 워크플로

자동화된 검증 게이트를 두는 것이 핵심이다.

```python
def promote_to_production(model_name: str, version: str, thresholds: dict) -> bool:
    client = MlflowClient()
    run = client.get_run(
        client.get_model_version(model_name, version).run_id
    )

    metrics = run.data.metrics

    # 기준치 검사
    for metric, min_val in thresholds.items():
        if metrics.get(metric, 0) < min_val:
            print(f"[FAIL] {metric}: {metrics.get(metric)} < {min_val}")
            return False

    client.transition_model_version_stage(
        name=model_name,
        version=version,
        stage="Production",
        archive_existing_versions=True,
    )
    print(f"[OK] {model_name} v{version} -> Production")
    return True


promote_to_production(
    model_name="fraud-detector",
    version="5",
    thresholds={"accuracy": 0.92, "f1_score": 0.88, "auc_roc": 0.95},
)
```

### SageMaker Model Registry

AWS 환경에서는 SageMaker Model Registry를 쓴다.

```python
import boto3

sm = boto3.client("sagemaker", region_name="ap-northeast-2")

# 모델 패키지 그룹 생성 (최초 1회)
sm.create_model_package_group(
    ModelPackageGroupName="fraud-detector",
    ModelPackageGroupDescription="Fraud detection models",
)

# 모델 등록
response = sm.create_model_package(
    ModelPackageGroupName="fraud-detector",
    ModelApprovalStatus="PendingManualApproval",
    InferenceSpecification={
        "Containers": [
            {
                "Image": "123456789.dkr.ecr.ap-northeast-2.amazonaws.com/fraud-model:v5",
                "ModelDataUrl": "s3://my-bucket/models/fraud-v5/model.tar.gz",
            }
        ],
        "SupportedContentTypes": ["application/json"],
        "SupportedResponseMIMETypes": ["application/json"],
    },
    ModelMetrics={
        "ModelQuality": {
            "Statistics": {
                "ContentType": "application/json",
                "S3Uri": "s3://my-bucket/metrics/v5/model_quality.json",
            }
        }
    },
)

model_package_arn = response["ModelPackageArn"]

# 수동 승인 또는 자동 승인
sm.update_model_package(
    ModelPackageArn=model_package_arn,
    ModelApprovalStatus="Approved",
)
```

---

## 2. Shadow 배포

Shadow 배포는 실 트래픽을 새 모델에 복제해서 보내되, 응답은 기존 모델 결과만 반환하는 패턴이다.

새 모델의 예측 결과는 로그에만 기록된다. 사용자에게 영향이 없으므로 리스크 없이 새 모델의 실제 성능을 측정할 수 있다.

### 구현 코드

```python
import asyncio
import logging
from dataclasses import dataclass
from typing import Any

logger = logging.getLogger(__name__)


@dataclass
class PredictRequest:
    request_id: str
    features: dict


class ShadowRouter:
    def __init__(self, primary_model, shadow_model):
        self.primary = primary_model
        self.shadow = shadow_model

    async def predict(self, request: PredictRequest) -> Any:
        # Primary 모델은 동기적으로 실행 (응답에 포함)
        primary_result = await self._call_model(self.primary, request)

        # Shadow 모델은 비동기 fire-and-forget
        asyncio.create_task(
            self._shadow_predict(request, primary_result)
        )

        return primary_result

    async def _call_model(self, model, request: PredictRequest) -> Any:
        return await asyncio.to_thread(model.predict, request.features)

    async def _shadow_predict(
        self, request: PredictRequest, primary_result: Any
    ) -> None:
        try:
            shadow_result = await self._call_model(self.shadow, request)

            # 비교 결과 로깅 (메트릭 시스템으로 전송)
            logger.info(
                "shadow_comparison",
                extra={
                    "request_id": request.request_id,
                    "primary": primary_result,
                    "shadow": shadow_result,
                    "match": primary_result == shadow_result,
                },
            )
        except Exception as e:
            logger.error(f"Shadow model error [{request.request_id}]: {e}")
```

Shadow 비교 결과를 집계하면 새 모델이 기존 모델과 얼마나 다른 예측을 하는지, 어떤 입력에서 차이가 나는지 파악할 수 있다.

---

## 3. A/B 테스트

A/B 테스트는 트래픽 일부를 새 모델(B)로 보내고, 실제 비즈니스 메트릭 차이를 측정한다.

Shadow와 달리 B 모델의 결과를 실제 사용자에게 반환한다. 그래서 실제 클릭률, 전환율, 에러율 같은 다운스트림 메트릭을 측정할 수 있다.

### 트래픽 분할 구현

```python
import hashlib
from enum import Enum


class ModelVariant(str, Enum):
    CONTROL = "control"   # 기존 모델 (A)
    TREATMENT = "treatment"  # 신규 모델 (B)


class ABRouter:
    def __init__(
        self,
        control_model,
        treatment_model,
        treatment_ratio: float = 0.1,  # 10% 트래픽을 B로
    ):
        self.models = {
            ModelVariant.CONTROL: control_model,
            ModelVariant.TREATMENT: treatment_model,
        }
        self.treatment_ratio = treatment_ratio

    def _assign_variant(self, user_id: str) -> ModelVariant:
        # 동일 user_id는 항상 같은 variant에 배정 (일관성 보장)
        hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        bucket = (hash_val % 1000) / 1000.0

        if bucket < self.treatment_ratio:
            return ModelVariant.TREATMENT
        return ModelVariant.CONTROL

    def predict(self, user_id: str, features: dict) -> dict:
        variant = self._assign_variant(user_id)
        model = self.models[variant]

        result = model.predict(features)

        return {
            "prediction": result,
            "variant": variant.value,
            "user_id": user_id,
        }
```

### 통계적 유의성 판단

충분한 샘플이 쌓이면 두 그룹의 메트릭 차이가 통계적으로 유의한지 검사한다.

```python
from scipy import stats
import numpy as np


def evaluate_ab_test(
    control_metrics: list[float],
    treatment_metrics: list[float],
    alpha: float = 0.05,
    min_effect_size: float = 0.01,
) -> dict:
    control = np.array(control_metrics)
    treatment = np.array(treatment_metrics)

    # Welch's t-test (분산이 다를 수 있음)
    t_stat, p_value = stats.ttest_ind(control, treatment, equal_var=False)

    control_mean = control.mean()
    treatment_mean = treatment.mean()
    relative_lift = (treatment_mean - control_mean) / control_mean

    significant = p_value < alpha
    meaningful = abs(relative_lift) >= min_effect_size

    return {
        "control_mean": control_mean,
        "treatment_mean": treatment_mean,
        "relative_lift": relative_lift,
        "p_value": p_value,
        "significant": significant,
        "meaningful": meaningful,
        "recommendation": "promote" if (significant and meaningful and relative_lift > 0) else "rollback",
    }
```

### Istio weight routing

인프라 레벨에서 트래픽을 분할할 때는 Istio VirtualService를 쓴다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: model-serving
spec:
  hosts:
    - model-serving
  http:
    - route:
        - destination:
            host: model-serving
            subset: v1  # 기존 모델
          weight: 90
        - destination:
            host: model-serving
            subset: v2  # 신규 모델
          weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: model-serving
spec:
  host: model-serving
  subsets:
    - name: v1
      labels:
        version: "1"
    - name: v2
      labels:
        version: "2"
```

---

## 4. 카나리 배포

카나리 배포는 새 모델로 트래픽을 점진적으로 이동시키는 패턴이다.

A/B 테스트가 실험 목적이라면, 카나리는 배포 목적이다. 메트릭 기준을 통과하면 비율을 높이고, 실패하면 즉시 롤백한다.

### 점진적 트래픽 이동

```python
import time
from dataclasses import dataclass, field


@dataclass
class CanaryConfig:
    steps: list[int] = field(default_factory=lambda: [5, 10, 25, 50, 100])
    step_duration_minutes: int = 30
    error_rate_threshold: float = 0.02    # 에러율 2% 초과 시 롤백
    latency_p99_threshold_ms: float = 500.0  # P99 지연 500ms 초과 시 롤백


class CanaryController:
    def __init__(self, config: CanaryConfig, metrics_client, traffic_router):
        self.config = config
        self.metrics = metrics_client
        self.router = traffic_router

    def run(self, new_version: str) -> str:
        for step_pct in self.config.steps:
            print(f"[Canary] {new_version}: {step_pct}% 트래픽 전환")
            self.router.set_weight(new_version, step_pct)

            # 안정화 대기
            time.sleep(self.config.step_duration_minutes * 60)

            # 메트릭 검사
            health = self._check_health(new_version)

            if not health["healthy"]:
                print(f"[Canary] 롤백: {health['reason']}")
                self.router.set_weight(new_version, 0)
                return "rollback"

            print(f"[Canary] {step_pct}% 통과 — error_rate={health['error_rate']:.3f}, p99={health['p99_ms']:.0f}ms")

        print(f"[Canary] {new_version}: 100% 완료")
        return "success"

    def _check_health(self, version: str) -> dict:
        window_minutes = self.config.step_duration_minutes

        error_rate = self.metrics.get_error_rate(version, window_minutes)
        p99_ms = self.metrics.get_latency_p99(version, window_minutes)

        if error_rate > self.config.error_rate_threshold:
            return {
                "healthy": False,
                "reason": f"error_rate {error_rate:.3f} > {self.config.error_rate_threshold}",
                "error_rate": error_rate,
                "p99_ms": p99_ms,
            }

        if p99_ms > self.config.latency_p99_threshold_ms:
            return {
                "healthy": False,
                "reason": f"p99_latency {p99_ms:.0f}ms > {self.config.latency_p99_threshold_ms}ms",
                "error_rate": error_rate,
                "p99_ms": p99_ms,
            }

        return {"healthy": True, "error_rate": error_rate, "p99_ms": p99_ms}
```

### Argo Rollouts로 자동화

Kubernetes 환경에서는 Argo Rollouts가 카나리 전략을 선언적으로 관리한다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: fraud-model-rollout
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 30m }
        - setWeight: 25
        - pause: { duration: 30m }
        - setWeight: 50
        - pause: { duration: 30m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: model-error-rate
        startingStep: 1
        args:
          - name: service-name
            value: fraud-model-svc
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: model-error-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 5m
      successCondition: result[0] < 0.02
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(model_errors_total{service="{{args.service-name}}"}[5m]))
            /
            sum(rate(model_requests_total{service="{{args.service-name}}"}[5m]))
```

---

## 5. 멀티 모델 서빙과 앙상블

하나의 서빙 서버에서 여러 모델을 동시에 서빙하면 리소스 효율을 높이고 앙상블 추론을 구성할 수 있다.

### NVIDIA Triton 모델 파이프라인

Triton은 앙상블 파이프라인을 model config로 선언한다.

```
# 디렉터리 구조
model_repository/
  ensemble_fraud/
    config.pbtxt
    1/  (비어 있음 — 앙상블은 가중치 파일 불필요)
  fraud_model_v1/
    config.pbtxt
    1/model.plan
  fraud_model_v2/
    config.pbtxt
    1/model.plan
```

앙상블 config.pbtxt:

```protobuf
name: "ensemble_fraud"
platform: "ensemble"
max_batch_size: 64

input [
  { name: "INPUT", data_type: TYPE_FP32, dims: [128] }
]
output [
  { name: "OUTPUT", data_type: TYPE_FP32, dims: [1] }
]

ensemble_scheduling {
  step [
    {
      model_name: "fraud_model_v1"
      model_version: -1
      input_map  { key: "INPUT"  value: "INPUT" }
      output_map { key: "OUTPUT" value: "v1_output" }
    },
    {
      model_name: "fraud_model_v2"
      model_version: -1
      input_map  { key: "INPUT"  value: "INPUT" }
      output_map { key: "OUTPUT" value: "v2_output" }
    },
    {
      model_name: "weighted_average"
      model_version: -1
      input_map  { key: "V1_INPUT" value: "v1_output" }
      input_map  { key: "V2_INPUT" value: "v2_output" }
      output_map { key: "OUTPUT"   value: "OUTPUT" }
    }
  ]
}
```

### Python 레벨 가중 평균 앙상블

Triton 없이 Python에서 앙상블을 구성할 때는 다음과 같이 작성한다.

```python
from dataclasses import dataclass
import numpy as np
from typing import Protocol


class Model(Protocol):
    def predict_proba(self, features: np.ndarray) -> np.ndarray:
        ...


@dataclass
class WeightedEnsemble:
    models: list[tuple[Model, float]]  # (model, weight) 쌍

    def __post_init__(self):
        total = sum(w for _, w in self.models)
        if abs(total - 1.0) > 1e-6:
            raise ValueError(f"가중치 합이 1.0이어야 함: {total}")

    def predict_proba(self, features: np.ndarray) -> np.ndarray:
        result = np.zeros(len(features))

        for model, weight in self.models:
            proba = model.predict_proba(features)[:, 1]  # 양성 클래스 확률
            result += weight * proba

        return result

    def predict(self, features: np.ndarray, threshold: float = 0.5) -> np.ndarray:
        return (self.predict_proba(features) >= threshold).astype(int)


# 사용 예
ensemble = WeightedEnsemble(
    models=[
        (model_v1, 0.4),
        (model_v2, 0.6),
    ]
)
```

앙상블 가중치는 고정값이 아니라 각 모델의 검증 셋 성능에 비례해서 설정하는 것이 일반적이다.

---

## 6. 롤백 전략

### 모델 성능 회귀 자동 탐지

새 모델이 배포된 이후 프로덕션에서 측정되는 메트릭을 지속적으로 모니터링해야 한다.

탐지해야 할 신호는 두 가지다. 정확도 저하(accuracy drop)와 지연 급증(latency spike)이다.

```python
import numpy as np
from datetime import datetime, timedelta


class ModelHealthMonitor:
    def __init__(self, metrics_store, alert_client):
        self.metrics = metrics_store
        self.alert = alert_client

    def check_accuracy_regression(
        self,
        model_version: str,
        baseline_accuracy: float,
        threshold_drop: float = 0.03,  # 3% 이상 하락 시 알림
        window_hours: int = 1,
    ) -> bool:
        """프로덕션 레이블이 있는 경우에만 사용 가능 (지연 레이블 포함)."""
        current_accuracy = self.metrics.get_accuracy(
            model_version,
            since=datetime.utcnow() - timedelta(hours=window_hours),
        )

        if current_accuracy is None:
            return True  # 샘플 부족 — 판단 보류

        drop = baseline_accuracy - current_accuracy

        if drop >= threshold_drop:
            self.alert.fire(
                title=f"[{model_version}] 정확도 회귀",
                message=f"baseline={baseline_accuracy:.3f}, current={current_accuracy:.3f}, drop={drop:.3f}",
                severity="critical",
            )
            return False

        return True

    def check_latency_spike(
        self,
        model_version: str,
        baseline_p99_ms: float,
        threshold_ratio: float = 1.5,  # 1.5배 초과 시 알림
        window_minutes: int = 10,
    ) -> bool:
        current_p99 = self.metrics.get_latency_p99(
            model_version,
            since=datetime.utcnow() - timedelta(minutes=window_minutes),
        )

        if current_p99 is None:
            return True

        ratio = current_p99 / baseline_p99_ms

        if ratio >= threshold_ratio:
            self.alert.fire(
                title=f"[{model_version}] 지연 급증",
                message=f"baseline={baseline_p99_ms:.0f}ms, current={current_p99:.0f}ms, ratio={ratio:.2f}x",
                severity="warning",
            )
            return False

        return True
```

### 자동 롤백 구현

감지 즉시 롤백까지 자동화하면 인시던트 대응 시간을 줄일 수 있다.

```python
class AutoRollbackController:
    def __init__(
        self,
        monitor: ModelHealthMonitor,
        traffic_router,
        model_registry,
        check_interval_seconds: int = 60,
    ):
        self.monitor = monitor
        self.router = traffic_router
        self.registry = model_registry
        self.interval = check_interval_seconds

    def watch(
        self,
        new_version: str,
        previous_version: str,
        baseline_accuracy: float,
        baseline_p99_ms: float,
    ) -> None:
        import time

        print(f"[AutoRollback] {new_version} 모니터링 시작")

        while True:
            time.sleep(self.interval)

            accuracy_ok = self.monitor.check_accuracy_regression(
                new_version, baseline_accuracy
            )
            latency_ok = self.monitor.check_latency_spike(
                new_version, baseline_p99_ms
            )

            if not accuracy_ok or not latency_ok:
                print(f"[AutoRollback] 회귀 감지 — {previous_version}으로 롤백")
                self._rollback(new_version, previous_version)
                break

    def _rollback(self, failed_version: str, stable_version: str) -> None:
        # 트래픽을 이전 버전으로 즉시 전환
        self.router.set_weight(failed_version, 0)
        self.router.set_weight(stable_version, 100)

        # 레지스트리 스테이지 변경
        self.registry.archive_version(failed_version)
        self.registry.promote_to_production(stable_version)

        print(f"[AutoRollback] 완료: {stable_version} -> Production")
```

### MLflow에서 이전 버전으로 즉시 복구

```python
def emergency_rollback(model_name: str, target_version: str) -> None:
    client = MlflowClient()

    # 현재 Production 버전 조회
    current = client.get_latest_versions(model_name, stages=["Production"])

    # 현재 버전 Archived 처리
    for mv in current:
        client.transition_model_version_stage(
            name=model_name,
            version=mv.version,
            stage="Archived",
        )

    # 지정 버전을 Production으로 복구
    client.transition_model_version_stage(
        name=model_name,
        version=target_version,
        stage="Production",
    )

    print(f"[Rollback] {model_name} v{target_version} -> Production 복구 완료")


# 실행
emergency_rollback("fraud-detector", target_version="4")
```

### 롤백 의사결정 기준 정리

| 시그널 | 기준값 | 권장 액션 |
|--------|--------|-----------|
| 에러율 | > 2% (기준치 대비) | 즉시 롤백 |
| P99 지연 | > 1.5x 기준치 | 경보 후 10분 내 미해결 시 롤백 |
| 정확도 하락 | > 3%p | 트래픽 동결 후 수동 검토 |
| 예측 분포 편향 | KL divergence > 임계값 | 알림 후 Shadow 비교 재실행 |

에러율과 지연은 즉시 측정 가능하다. 정확도와 예측 분포는 레이블이 지연돼 들어오므로 별도 파이프라인을 구성해야 한다.

---

## 마무리

안전한 모델 교체를 위한 핵심 원칙은 단계적 노출과 자동화된 게이트다.

모든 트래픽을 한 번에 전환하는 방식은 검증할 기회 없이 위험을 떠안는 것이다. Shadow로 시작해서 카나리로 점진적으로 넘기고, 메트릭이 기준을 벗어나면 자동으로 롤백하는 파이프라인을 구성하면 대부분의 모델 배포 인시던트를 사전에 차단할 수 있다.

다음 편에서는 실시간 피처 스토어와 온라인 피처 서빙 아키텍처를 다룬다.

---

## 참고 자료

- [MLflow Model Registry Documentation](https://mlflow.org/docs/latest/model-registry.html) — MLflow 공식 모델 레지스트리 가이드
- [AWS SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html) — SageMaker 모델 버전 관리 및 승인 워크플로
- [NVIDIA Triton Inference Server: Ensemble Models](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/architecture.html#ensemble-models) — Triton 앙상블 파이프라인 구성
- [Argo Rollouts: Canary Deployments](https://argo-rollouts.readthedocs.io/en/stable/features/canary/) — Argo Rollouts 카나리 전략 및 Analysis Template
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/) — Istio VirtualService 기반 가중 라우팅
- [Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)](https://papers.nips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html) — ML 시스템의 기술 부채와 배포 복잡성
