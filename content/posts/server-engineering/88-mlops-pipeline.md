---
title: "[AI / ML 서빙] 3편 — MLOps 파이프라인: 모델의 CI/CD"
date: 2026-03-17T13:05:00+09:00
draft: false
tags: ["MLOps", "Kubeflow", "Feature Store", "데이터 드리프트", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "ML 파이프라인 구성(데이터 수집→전처리→학습→평가→배포), Kubeflow·Airflow·SageMaker Pipelines 비교, 피처 스토어(Feast, Tecton)와 학습/서빙 피처 일관성, 데이터/모델 드리프트 탐지와 재학습 트리거까지"
---

소프트웨어 개발에 CI/CD가 있다면 ML에는 MLOps가 있다. 모델을 한 번 학습하고 배포하면 끝나는 것이 아니다. 데이터는 변하고, 세상은 변하고, 모델의 예측은 조용히 틀려간다. MLOps 파이프라인은 이 사이클 전체를 자동화한다. 데이터 수집부터 배포, 모니터링, 재학습까지.

---

## 1. ML 파이프라인 전체 구조

ML 파이프라인은 크게 다섯 단계로 나뉜다.

```
데이터 수집 → 전처리/피처 엔지니어링 → 학습 → 평가 → 배포
                    ↑                                      |
                    └──────── 재학습 트리거 ←──────────────┘
```

각 단계는 독립적으로 실행 가능해야 하고, 실패 시 재시도할 수 있어야 한다.

### 1-1. 데이터 수집 단계

원시 데이터를 데이터 레이크나 데이터 웨어하우스에서 끌어온다.

```python
# data_ingestion.py
import pandas as pd
from google.cloud import bigquery

def ingest_training_data(
    project_id: str,
    dataset: str,
    start_date: str,
    end_date: str,
    output_path: str
) -> pd.DataFrame:
    client = bigquery.Client(project=project_id)

    query = f"""
        SELECT
            user_id,
            item_id,
            event_type,
            timestamp,
            session_duration
        FROM `{project_id}.{dataset}.events`
        WHERE DATE(timestamp) BETWEEN '{start_date}' AND '{end_date}'
          AND event_type IN ('click', 'purchase', 'view')
    """

    df = client.query(query).to_dataframe()

    # 데이터 품질 검증
    assert len(df) > 0, "수집된 데이터가 없습니다"
    assert df['user_id'].notna().all(), "user_id에 null 값이 있습니다"

    df.to_parquet(output_path, index=False)
    print(f"수집 완료: {len(df):,}건 → {output_path}")

    return df
```

데이터 품질 검증은 파이프라인 초입에 넣는 것이 원칙이다. 나중에 학습이 끝나고 나서 데이터 문제를 발견하면 시간 낭비가 크다.

### 1-2. 전처리 단계

전처리 로직은 반드시 재현 가능해야 한다. 학습 때 쓴 전처리와 서빙 때 쓰는 전처리가 달라지는 순간 예측 품질이 무너진다.

```python
# preprocessing.py
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OrdinalEncoder
from sklearn.compose import ColumnTransformer
import joblib

def build_preprocessor(numeric_features: list, categorical_features: list):
    numeric_transformer = Pipeline(steps=[
        ('scaler', StandardScaler())
    ])

    categorical_transformer = Pipeline(steps=[
        ('encoder', OrdinalEncoder(handle_unknown='use_encoded_value', unknown_value=-1))
    ])

    preprocessor = ColumnTransformer(transformers=[
        ('num', numeric_transformer, numeric_features),
        ('cat', categorical_transformer, categorical_features)
    ])

    return preprocessor

def fit_and_save(df, output_path: str):
    numeric_features = ['session_duration', 'page_views', 'click_count']
    categorical_features = ['device_type', 'os', 'country']

    preprocessor = build_preprocessor(numeric_features, categorical_features)
    preprocessor.fit(df[numeric_features + categorical_features])

    # 전처리기를 아티팩트로 저장
    joblib.dump(preprocessor, output_path)
    return preprocessor
```

전처리기 객체를 아티팩트로 저장하고 모델과 함께 버전 관리하는 것이 핵심이다. 전처리기와 모델은 한 쌍이다.

### 1-3. 학습 단계

```python
# train.py
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score
import joblib

def train(X_train, y_train, X_val, y_val, params: dict):
    with mlflow.start_run():
        mlflow.log_params(params)

        model = GradientBoostingClassifier(**params)
        model.fit(X_train, y_train)

        # 검증 메트릭 기록
        val_pred = model.predict_proba(X_val)[:, 1]
        auc = roc_auc_score(y_val, val_pred)
        mlflow.log_metric("val_auc", auc)

        # 모델 아티팩트 저장
        mlflow.sklearn.log_model(model, "model")

        run_id = mlflow.active_run().info.run_id
        print(f"학습 완료 | val_auc={auc:.4f} | run_id={run_id}")

        return model, auc, run_id
```

### 1-4. 평가 단계

학습이 끝났다고 바로 배포하지 않는다. 기존 운영 모델(챔피언)과 비교해서 신규 모델(챌린저)이 실제로 더 나은지 확인한다.

```python
# evaluate.py
def evaluate_and_gate(
    champion_model,
    challenger_model,
    X_test,
    y_test,
    threshold: float = 0.01  # 최소 개선 요구치
) -> bool:
    champion_auc = roc_auc_score(y_test, champion_model.predict_proba(X_test)[:, 1])
    challenger_auc = roc_auc_score(y_test, challenger_model.predict_proba(X_test)[:, 1])

    improvement = challenger_auc - champion_auc
    print(f"챔피언 AUC: {champion_auc:.4f}")
    print(f"챌린저 AUC: {challenger_auc:.4f}")
    print(f"개선폭: {improvement:+.4f}")

    if improvement >= threshold:
        print("배포 승인")
        return True
    else:
        print(f"배포 거부 — 개선폭 {improvement:.4f} < 임계값 {threshold}")
        return False
```

### 1-5. 배포 단계

평가를 통과한 모델만 운영 환경으로 올린다.

```python
# deploy.py
import mlflow.pyfunc
from kubernetes import client as k8s_client

def deploy_to_kubernetes(run_id: str, namespace: str, model_name: str):
    # MLflow에서 모델 URI 가져오기
    model_uri = f"runs:/{run_id}/model"

    # Kubernetes Deployment 업데이트
    apps_v1 = k8s_client.AppsV1Api()

    deployment = apps_v1.read_namespaced_deployment(
        name=f"{model_name}-serving",
        namespace=namespace
    )

    # 새 모델 URI를 환경변수로 주입
    containers = deployment.spec.template.spec.containers
    for container in containers:
        for env in container.env:
            if env.name == "MODEL_URI":
                env.value = model_uri

    apps_v1.patch_namespaced_deployment(
        name=f"{model_name}-serving",
        namespace=namespace,
        body=deployment
    )

    print(f"배포 완료: {model_name} → {model_uri}")
```

---

## 2. 파이프라인 도구 비교

### 2-1. Kubeflow Pipelines

Kubernetes 네이티브 ML 파이프라인 플랫폼이다. 각 단계를 컨테이너로 격리하고, 의존성 그래프를 선언적으로 정의한다.

```python
# kubeflow_pipeline.py
import kfp
from kfp import dsl
from kfp.components import func_to_container_op

@func_to_container_op
def ingest_op(start_date: str, end_date: str) -> str:
    import subprocess
    result = subprocess.run(
        ['python', 'data_ingestion.py', start_date, end_date],
        capture_output=True
    )
    return '/data/raw.parquet'

@func_to_container_op
def preprocess_op(input_path: str) -> str:
    import subprocess
    subprocess.run(['python', 'preprocessing.py', input_path])
    return '/data/processed.parquet'

@func_to_container_op
def train_op(input_path: str, params: dict) -> str:
    import subprocess
    result = subprocess.run(
        ['python', 'train.py', input_path],
        capture_output=True,
        text=True
    )
    # run_id 추출
    for line in result.stdout.split('\n'):
        if 'run_id=' in line:
            return line.split('run_id=')[1].strip()

@dsl.pipeline(
    name='ml-training-pipeline',
    description='데이터 수집부터 배포까지'
)
def ml_pipeline(
    start_date: str = '2026-01-01',
    end_date: str = '2026-03-01'
):
    ingest_task = ingest_op(start_date, end_date)

    preprocess_task = preprocess_op(ingest_task.output)
    preprocess_task.after(ingest_task)

    train_task = train_op(
        preprocess_task.output,
        {'n_estimators': 100, 'max_depth': 5}
    )
    train_task.after(preprocess_task)

    # GPU 리소스 요청
    train_task.set_gpu_limit(1)
    train_task.set_memory_limit('16Gi')
```

Kubeflow의 강점은 각 단계가 완전히 격리된다는 점이다. 전처리 컨테이너와 학습 컨테이너는 서로 다른 Python 환경을 가질 수 있다.

### 2-2. Apache Airflow

데이터 엔지니어링 팀이 이미 Airflow를 쓰고 있다면 ML 파이프라인도 Airflow로 통합하는 것이 현실적이다.

```python
# airflow_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'ml-team',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['ml-alerts@company.com']
}

with DAG(
    dag_id='ml_training_pipeline',
    default_args=default_args,
    schedule_interval='0 2 * * 1',  # 매주 월요일 오전 2시
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['ml', 'training']
) as dag:

    ingest = KubernetesPodOperator(
        task_id='data_ingestion',
        name='data-ingestion',
        namespace='ml-pipelines',
        image='company/data-ingestion:latest',
        arguments=['--start-date', '{{ ds }}'],
        get_logs=True,
        is_delete_operator_pod=True
    )

    preprocess = KubernetesPodOperator(
        task_id='preprocessing',
        name='preprocessing',
        namespace='ml-pipelines',
        image='company/preprocessing:latest',
        get_logs=True,
        is_delete_operator_pod=True
    )

    train = KubernetesPodOperator(
        task_id='model_training',
        name='model-training',
        namespace='ml-pipelines',
        image='company/training:latest',
        resources={
            'request_cpu': '4',
            'request_memory': '16Gi',
            'limit_gpu': '1'
        },
        get_logs=True,
        is_delete_operator_pod=True
    )

    ingest >> preprocess >> train
```

Airflow는 스케줄링과 모니터링이 강력하다. 반면 ML 전용 기능(아티팩트 추적, 하이퍼파라미터 관리)은 별도 도구와 조합해야 한다.

### 2-3. SageMaker Pipelines

AWS 환경에 올인하고 있다면 SageMaker Pipelines가 가장 빠른 선택지다.

```python
# sagemaker_pipeline.py
import boto3
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep
from sagemaker.workflow.parameters import ParameterString, ParameterFloat
from sagemaker.sklearn.processing import SKLearnProcessor
from sagemaker.estimator import Estimator

role = "arn:aws:iam::123456789:role/SageMakerRole"

# 파이프라인 파라미터 정의
start_date = ParameterString(name="StartDate", default_value="2026-01-01")
min_auc = ParameterFloat(name="MinAUC", default_value=0.75)

# 전처리 스텝
sklearn_processor = SKLearnProcessor(
    framework_version="1.2-1",
    instance_type="ml.m5.xlarge",
    instance_count=1,
    role=role
)

preprocess_step = ProcessingStep(
    name="PreprocessData",
    processor=sklearn_processor,
    inputs=[...],
    outputs=[...],
    code="preprocessing.py",
    job_arguments=["--start-date", start_date]
)

# 학습 스텝
estimator = Estimator(
    image_uri="763104351884.dkr.ecr.ap-northeast-2.amazonaws.com/pytorch-training:2.0-gpu-py310",
    instance_type="ml.g4dn.xlarge",
    instance_count=1,
    role=role,
    output_path="s3://my-bucket/models/"
)

train_step = TrainingStep(
    name="TrainModel",
    estimator=estimator,
    inputs={
        "train": preprocess_step.properties.ProcessingOutputConfig.Outputs["train"].S3Output.S3Uri
    }
)

pipeline = Pipeline(
    name="MLTrainingPipeline",
    parameters=[start_date, min_auc],
    steps=[preprocess_step, train_step]
)

pipeline.upsert(role_arn=role)
```

### 도구 선택 기준

| 기준 | Kubeflow | Airflow | SageMaker |
|------|----------|---------|-----------|
| 인프라 | Kubernetes 필수 | 범용 | AWS 전용 |
| ML 전용 기능 | 강함 | 약함 | 강함 |
| 학습 곡선 | 높음 | 중간 | 중간 |
| 비용 | 인프라 직접 운영 | 인프라 직접 운영 | 관리형 서비스 |
| 유연성 | 최고 | 높음 | AWS 생태계 내 |

---

## 3. 피처 스토어

피처 스토어는 ML에서 가장 자주 간과되는 컴포넌트다. 없으면 어떤 일이 생기는지 먼저 보자.

### 3-1. 학습/서빙 피처 불일치 문제

```python
# 학습 시 (Jupyter Notebook)
df['user_purchase_rate_7d'] = (
    df.groupby('user_id')['purchase']
    .transform(lambda x: x.rolling(7).mean())
)

# 서빙 시 (API 서버) — 다른 엔지니어가 작성
def get_user_features(user_id: int) -> dict:
    # rolling window 대신 단순 평균 사용 → 계산 방식이 다름
    purchases = db.query(
        "SELECT AVG(purchase) FROM events WHERE user_id = %s AND ts > NOW() - INTERVAL 7 DAY",
        user_id
    )
    return {'user_purchase_rate_7d': purchases}
```

이 코드의 문제는 겉으로 드러나지 않는다. 학습 때와 서빙 때 피처 계산 로직이 달라서 모델이 본 적 없는 분포로 예측한다. 성능 저하는 있지만 명시적 오류는 없다.

피처 스토어가 이 문제를 해결한다. 피처 계산 로직을 한 곳에 정의하고, 학습과 서빙 모두 같은 로직을 통해 피처를 가져간다.

### 3-2. Feast 구현

Feast는 오픈소스 피처 스토어다. 로컬 환경부터 대규모 프로덕션까지 사용할 수 있다.

```python
# feature_repo/features.py
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32, Int64
from datetime import timedelta

# 엔티티 정의
user = Entity(
    name="user_id",
    description="사용자 ID"
)

# 데이터 소스 정의
user_stats_source = FileSource(
    path="data/user_stats.parquet",
    timestamp_field="event_timestamp"
)

# 피처 뷰 정의 — 학습과 서빙에서 공유
user_stats_fv = FeatureView(
    name="user_statistics",
    entities=["user_id"],
    ttl=timedelta(days=1),
    schema=[
        Field(name="purchase_rate_7d", dtype=Float32),
        Field(name="session_count_30d", dtype=Int64),
        Field(name="avg_session_duration", dtype=Float32),
        Field(name="item_category_diversity", dtype=Float32),
    ],
    online=True,  # 온라인 스토어(Redis)에도 저장
    source=user_stats_source
)
```

```python
# 학습 시 — 포인트인타임 정합성을 보장한 피처 조회
from feast import FeatureStore
import pandas as pd

store = FeatureStore(repo_path="feature_repo/")

entity_df = pd.DataFrame({
    "user_id": [1001, 1002, 1003],
    "event_timestamp": pd.to_datetime([
        "2026-01-15", "2026-01-15", "2026-01-15"
    ])
})

# 레이블 시간 기준으로 올바른 피처값 조회 (데이터 누수 방지)
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_statistics:purchase_rate_7d",
        "user_statistics:session_count_30d",
        "user_statistics:avg_session_duration",
    ]
).to_df()

print(training_df.head())
```

```python
# 서빙 시 — 동일한 피처 정의에서 온라인 조회
def predict(user_id: int) -> float:
    feature_vector = store.get_online_features(
        features=[
            "user_statistics:purchase_rate_7d",
            "user_statistics:session_count_30d",
            "user_statistics:avg_session_duration",
        ],
        entity_rows=[{"user_id": user_id}]
    ).to_dict()

    # 학습 때와 완전히 동일한 피처
    X = [[
        feature_vector["purchase_rate_7d"][0],
        feature_vector["session_count_30d"][0],
        feature_vector["avg_session_duration"][0],
    ]]

    return model.predict_proba(X)[0][1]
```

피처 스토어의 핵심은 `get_historical_features`의 포인트인타임 쿼리다. 레이블 시점 기준으로 과거 피처값을 정확하게 조회해 미래 데이터 누수를 막는다.

### 3-3. 피처 구체화 파이프라인

온라인 스토어에 최신 피처를 채우는 작업도 파이프라인의 일부다.

```python
# materialize_features.py
from feast import FeatureStore
from datetime import datetime, timedelta

def materialize_features():
    store = FeatureStore(repo_path="feature_repo/")

    end_date = datetime.utcnow()
    start_date = end_date - timedelta(days=1)

    # 오프라인 스토어 → 온라인 스토어(Redis) 동기화
    store.materialize(
        start_date=start_date,
        end_date=end_date
    )

    print(f"피처 구체화 완료: {start_date} ~ {end_date}")

# Airflow나 Kubernetes CronJob으로 주기적 실행
```

---

## 4. 데이터/모델 드리프트 탐지

모델은 시간이 지나면서 조용히 성능이 나빠진다. 드리프트 탐지는 이를 조기에 포착한다.

### 4-1. 데이터 드리프트 — PSI (Population Stability Index)

PSI는 학습 데이터와 현재 운영 데이터의 분포 차이를 측정한다.

```python
# drift_detection.py
import numpy as np
import pandas as pd
from scipy import stats

def calculate_psi(expected: np.ndarray, actual: np.ndarray, buckets: int = 10) -> float:
    """
    PSI 계산.
    PSI < 0.1: 분포 안정
    0.1 <= PSI < 0.2: 약간의 변화 — 모니터링 강화
    PSI >= 0.2: 심각한 변화 — 재학습 필요
    """
    def scale_range(input_arr, min_val, max_val):
        return (input_arr - min_val) / (max_val - min_val + 1e-10)

    breakpoints = np.arange(0, buckets + 1) / buckets

    expected_percents = np.histogram(
        scale_range(expected, expected.min(), expected.max()),
        bins=breakpoints
    )[0] / len(expected)

    actual_percents = np.histogram(
        scale_range(actual, expected.min(), expected.max()),
        bins=breakpoints
    )[0] / len(actual)

    # 0 방지
    expected_percents = np.where(expected_percents == 0, 0.0001, expected_percents)
    actual_percents = np.where(actual_percents == 0, 0.0001, actual_percents)

    psi = np.sum(
        (actual_percents - expected_percents) * np.log(actual_percents / expected_percents)
    )

    return psi

def detect_feature_drift(reference_df: pd.DataFrame, current_df: pd.DataFrame) -> dict:
    results = {}
    numeric_cols = reference_df.select_dtypes(include=[np.number]).columns

    for col in numeric_cols:
        psi = calculate_psi(
            reference_df[col].dropna().values,
            current_df[col].dropna().values
        )

        status = 'stable'
        if psi >= 0.2:
            status = 'critical'
        elif psi >= 0.1:
            status = 'warning'

        results[col] = {'psi': round(psi, 4), 'status': status}

    return results

# 사용 예시
reference_df = pd.read_parquet('data/training_reference.parquet')
current_df = pd.read_parquet('data/last_7days.parquet')

drift_report = detect_feature_drift(reference_df, current_df)
for feature, info in drift_report.items():
    if info['status'] != 'stable':
        print(f"[{info['status'].upper()}] {feature}: PSI={info['psi']}")
```

### 4-2. 콜모고로프-스미르노프 검정

두 분포가 통계적으로 같은지 검정한다. PSI보다 엄밀한 통계적 근거를 제공한다.

```python
def ks_test_drift(reference: np.ndarray, current: np.ndarray, alpha: float = 0.05) -> dict:
    statistic, p_value = stats.ks_2samp(reference, current)

    return {
        'statistic': round(statistic, 4),
        'p_value': round(p_value, 6),
        'drifted': p_value < alpha,  # p < 0.05면 분포 차이 유의미
        'interpretation': '드리프트 감지됨' if p_value < alpha else '분포 안정'
    }

# 여러 피처에 대해 일괄 검정
def batch_ks_test(reference_df: pd.DataFrame, current_df: pd.DataFrame) -> pd.DataFrame:
    results = []
    for col in reference_df.select_dtypes(include=[np.number]).columns:
        result = ks_test_drift(
            reference_df[col].dropna().values,
            current_df[col].dropna().values
        )
        result['feature'] = col
        results.append(result)

    return pd.DataFrame(results).sort_values('p_value')
```

### 4-3. 모델 드리프트 탐지

데이터 드리프트가 없어도 예측 분포가 바뀔 수 있다. 예측값의 분포를 추적한다.

```python
# model_drift.py
import redis
import json
from collections import deque

class PredictionMonitor:
    def __init__(self, window_size: int = 10000):
        self.redis = redis.Redis(host='redis', port=6379, db=1)
        self.window_size = window_size

    def record_prediction(self, model_version: str, prediction: float):
        key = f"predictions:{model_version}"
        self.redis.lpush(key, prediction)
        self.redis.ltrim(key, 0, self.window_size - 1)

    def get_prediction_stats(self, model_version: str) -> dict:
        key = f"predictions:{model_version}"
        raw = self.redis.lrange(key, 0, -1)
        predictions = [float(x) for x in raw]

        if not predictions:
            return {}

        arr = np.array(predictions)
        return {
            'count': len(arr),
            'mean': round(float(arr.mean()), 4),
            'std': round(float(arr.std()), 4),
            'p10': round(float(np.percentile(arr, 10)), 4),
            'p50': round(float(np.percentile(arr, 50)), 4),
            'p90': round(float(np.percentile(arr, 90)), 4),
        }

    def compare_with_baseline(
        self,
        current_version: str,
        baseline_stats: dict
    ) -> bool:
        current_stats = self.get_prediction_stats(current_version)
        if not current_stats:
            return False

        # 평균 예측값이 기준 대비 20% 이상 이탈하면 드리프트
        mean_diff = abs(current_stats['mean'] - baseline_stats['mean'])
        threshold = baseline_stats['mean'] * 0.2

        if mean_diff > threshold:
            print(f"모델 드리프트 감지: mean {baseline_stats['mean']:.4f} → {current_stats['mean']:.4f}")
            return True

        return False
```

---

## 5. 재학습 트리거 설계

언제 재학습을 시작할지 결정하는 것이 MLOps의 핵심 의사결정 중 하나다.

### 5-1. 스케줄 기반 트리거

가장 단순한 방법이다. 매주 월요일 새벽에 재학습한다.

```yaml
# kubernetes/cronjob-retrain.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: model-retrain
  namespace: ml-pipelines
spec:
  schedule: "0 2 * * 1"  # 매주 월요일 오전 2시 (UTC)
  concurrencyPolicy: Forbid  # 중복 실행 방지
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: retrain
            image: company/ml-training:latest
            env:
            - name: MLFLOW_TRACKING_URI
              value: "http://mlflow-service:5000"
            - name: FEATURE_STORE_URI
              value: "redis://redis-service:6379"
            resources:
              requests:
                memory: "8Gi"
                cpu: "4"
              limits:
                nvidia.com/gpu: "1"
          restartPolicy: OnFailure
```

장점은 예측 가능하다는 것이다. 단점은 드리프트가 갑자기 발생해도 다음 스케줄까지 기다려야 한다.

### 5-2. 이벤트 기반 트리거

드리프트가 탐지되면 즉시 재학습을 시작한다.

```python
# retrain_trigger.py
import boto3
import json
from drift_detection import detect_feature_drift, batch_ks_test

class RetrainTrigger:
    def __init__(self, pipeline_name: str, region: str = 'ap-northeast-2'):
        self.sm_client = boto3.client('sagemaker', region_name=region)
        self.pipeline_name = pipeline_name

    def check_and_trigger(
        self,
        reference_df,
        current_df,
        psi_threshold: float = 0.2,
        ks_alpha: float = 0.01
    ) -> bool:
        # PSI 기반 검사
        drift_report = detect_feature_drift(reference_df, current_df)
        critical_features = [
            f for f, info in drift_report.items()
            if info['status'] == 'critical'
        ]

        # KS 검정으로 이중 확인
        ks_results = batch_ks_test(reference_df, current_df)
        drifted_features = ks_results[ks_results['drifted']]['feature'].tolist()

        # 두 방법 모두에서 드리프트가 감지된 피처가 있을 때만 트리거
        confirmed_drifts = set(critical_features) & set(drifted_features)

        if confirmed_drifts:
            print(f"드리프트 확인된 피처: {confirmed_drifts}")
            self._trigger_pipeline()
            return True

        return False

    def _trigger_pipeline(self):
        response = self.sm_client.start_pipeline_execution(
            PipelineName=self.pipeline_name,
            PipelineExecutionDisplayName=f"drift-triggered-{datetime.utcnow().strftime('%Y%m%d-%H%M')}",
            PipelineParameters=[
                {'Name': 'TriggerReason', 'Value': 'data_drift'}
            ]
        )
        print(f"파이프라인 실행 시작: {response['PipelineExecutionArn']}")
```

이벤트 기반 트리거는 재학습 빈도가 불규칙해질 수 있다. 최소 재학습 간격을 두어 과도한 재학습을 방지한다.

```python
def _should_trigger(self) -> bool:
    # 마지막 재학습으로부터 최소 24시간 경과 확인
    last_run = self.redis.get('last_retrain_timestamp')
    if last_run:
        last_dt = datetime.fromisoformat(last_run.decode())
        if (datetime.utcnow() - last_dt).total_seconds() < 86400:
            print("최소 재학습 간격 미경과, 스킵")
            return False
    return True
```

### 5-3. 하이브리드 전략

현실에서 가장 많이 쓰는 방식이다.

```
- 스케줄: 매주 월요일 정기 재학습 (드리프트 여부 무관)
- 이벤트: PSI >= 0.2 + KS p < 0.01 동시 만족 시 즉시 트리거
- 성능 저하: 실시간 AUC가 기준 대비 5% 이상 하락 시 알림 + 수동 확인
```

---

## 6. 실험 추적과 자동 승인

### 6-1. MLflow 설정

```python
# mlflow_setup.py
import mlflow
from mlflow.tracking import MlflowClient

# MLflow 서버 설정
mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("purchase-prediction-v2")

def log_experiment(model, params: dict, metrics: dict, artifacts: dict):
    with mlflow.start_run() as run:
        # 파라미터 기록
        mlflow.log_params(params)

        # 메트릭 기록
        mlflow.log_metrics(metrics)

        # 아티팩트 기록
        for name, path in artifacts.items():
            mlflow.log_artifact(path, artifact_path=name)

        # 모델 기록
        mlflow.sklearn.log_model(
            model,
            "model",
            registered_model_name="purchase-prediction"
        )

        return run.info.run_id
```

### 6-2. 모델 레지스트리와 자동 승인

```python
# model_registry.py
from mlflow.tracking import MlflowClient
from mlflow.entities.model_registry import ModelVersion

client = MlflowClient()

def auto_promote(
    model_name: str,
    run_id: str,
    metrics: dict,
    thresholds: dict
) -> bool:
    # 임계값 기반 자동 승인
    for metric, min_value in thresholds.items():
        if metrics.get(metric, 0) < min_value:
            print(f"자동 승인 거부: {metric}={metrics[metric]:.4f} < {min_value}")
            # Staging 단계로 등록 (수동 검토 대기)
            client.transition_model_version_stage(
                name=model_name,
                version=get_latest_version(model_name),
                stage="Staging"
            )
            return False

    # 모든 임계값 통과 → Production 자동 승격
    latest = get_latest_version(model_name)
    client.transition_model_version_stage(
        name=model_name,
        version=latest,
        stage="Production"
    )
    print(f"자동 승인 완료: {model_name} v{latest} → Production")
    return True

def get_latest_version(model_name: str) -> str:
    versions = client.get_latest_versions(model_name, stages=["None"])
    return versions[0].version if versions else None

# 실제 사용
thresholds = {
    'val_auc': 0.78,
    'val_precision': 0.70,
    'val_recall': 0.65
}

promoted = auto_promote(
    model_name="purchase-prediction",
    run_id=run_id,
    metrics={'val_auc': 0.81, 'val_precision': 0.73, 'val_recall': 0.68},
    thresholds=thresholds
)
```

### 6-3. W&B와 MLflow 비교

| 항목 | MLflow | W&B (Weights & Biases) |
|------|--------|------------------------|
| 호스팅 | 셀프 호스팅 가능 | SaaS (셀프 호스팅 옵션 있음) |
| 시각화 | 기본 수준 | 풍부한 실시간 차트 |
| 협업 | 제한적 | 강력한 팀 협업 기능 |
| 비용 | 오픈소스 | 유료 (소규모 무료) |
| 아티팩트 관리 | 내장 | Artifact 별도 관리 |
| 모델 레지스트리 | 내장 | 내장 |

팀 규모가 작고 비용에 민감하면 MLflow. 빠른 실험과 시각화, 팀 공유가 중요하면 W&B.

---

## 정리

MLOps 파이프라인을 구축할 때 현실적인 우선순위는 다음과 같다.

첫째, 재현성을 먼저 확보한다. 같은 코드와 데이터로 같은 결과가 나와야 한다.

둘째, 피처 스토어를 일찍 도입한다. 나중에 도입하려면 기존 코드를 전면 수정해야 한다.

셋째, 드리프트 탐지는 두 가지 방법을 조합한다. PSI 하나만 쓰면 오탐이 많다.

넷째, 처음에는 스케줄 기반 트리거로 시작한다. 이벤트 기반은 안정된 드리프트 탐지 기반 위에서 추가한다.

파이프라인 도구 선택은 팀의 현재 스택을 따른다. 이미 Airflow가 있으면 Airflow, AWS에 올인이면 SageMaker, Kubernetes가 기반이면 Kubeflow다. 모든 도구가 같은 원칙 위에 있다. 데이터와 코드의 버전을 추적하고, 학습과 서빙이 같은 피처를 보게 만들고, 모델 품질을 지속적으로 감시하는 것.

---

## 참고 자료

- [Feast 공식 문서](https://docs.feast.dev/) — 피처 스토어 설계와 구현
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html) — 실험 추적, 모델 레지스트리
- [Kubeflow Pipelines SDK](https://www.kubeflow.org/docs/components/pipelines/) — Kubernetes 기반 ML 파이프라인
- [Evidently AI](https://docs.evidentlyai.com/) — 데이터/모델 드리프트 탐지 프레임워크
- [Chip Huyen, "Designing Machine Learning Systems"](https://www.oreilly.com/library/view/designing-machine-learning/9781098107956/) — MLOps 전반의 실무 설계
- [Google Cloud, "Practitioners Guide to MLOps"](https://cloud.google.com/resources/mlops-whitepaper) — MLOps 성숙도 모델과 구현 전략
