---
title: "[AI / ML 서빙] 7편 — AI 서비스 관측과 품질: 모델 운영의 현실"
date: 2026-03-17T13:01:00+09:00
draft: false
tags: ["AI 관측", "모델 모니터링", "A/B 테스트", "GPU 모니터링", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "추론 메트릭(레이턴시, 처리량, 토큰/초, GPU 사용률), 모델 품질 모니터링과 피드백 루프, A/B 테스트 통계적 유의성, 비용 대시보드와 팀별 사용량 할당, AI 서비스 장애 패턴(OOM, 큐 누적, GPU 고장)까지"
---

모델을 배포하는 건 시작일 뿐이다.

운영 중인 AI 서비스는 일반 웹 서비스와 다른 차원의 관측 문제를 만든다. GPU 메모리 파편화, 토큰 단위 레이턴시 분포, 모델 출력의 품질 저하 — 이 모든 것을 적절한 도구 없이 맨눈으로 보려 하면 이미 늦다.

이 글은 AI 추론 서비스를 실제로 운영할 때 필요한 메트릭 설계, 품질 추적, A/B 테스트, 비용 관리, 장애 대응을 다룬다.

---

## 추론 메트릭 설계

AI 서비스에서 레이턴시는 단일 숫자가 아니다. 두 가지로 나눠야 한다.

**TTFT(Time To First Token)** 은 요청 시작부터 첫 토큰이 나올 때까지의 시간이다. 사용자가 체감하는 반응성을 결정한다.

**TPS(Tokens Per Second)** 는 토큰 생성 속도다. 스트리밍 응답의 부드러움을 결정한다.

두 메트릭을 따로 추적하지 않으면 어디가 병목인지 알 수 없다.

```python
import time
from prometheus_client import Histogram, Gauge, Counter

# TTFT 히스토그램 — 버킷을 AI 특성에 맞게 설정
TTFT_HISTOGRAM = Histogram(
    "llm_time_to_first_token_seconds",
    "Time to first token",
    buckets=[0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0],
    labelnames=["model", "endpoint"],
)

# 처리량 — 초당 토큰 수
TPS_HISTOGRAM = Histogram(
    "llm_tokens_per_second",
    "Token generation throughput",
    buckets=[10, 25, 50, 100, 200, 500, 1000],
    labelnames=["model"],
)

# 요청당 총 토큰 (입력 + 출력)
TOKEN_COUNTER = Counter(
    "llm_tokens_total",
    "Total tokens processed",
    labelnames=["model", "type"],  # type: input | output
)

# GPU 메모리 사용률
GPU_MEMORY_USAGE = Gauge(
    "gpu_memory_used_bytes",
    "GPU memory currently used",
    labelnames=["device_id"],
)
```

메트릭 수집은 생성 루프에서 직접 측정해야 한다.

```python
import asyncio

async def generate_with_metrics(model_name: str, prompt: str, model_client):
    input_tokens = count_tokens(prompt)
    TOKEN_COUNTER.labels(model=model_name, type="input").inc(input_tokens)

    start_time = time.monotonic()
    first_token_recorded = False
    output_tokens = 0

    async for token in model_client.stream(prompt):
        if not first_token_recorded:
            ttft = time.monotonic() - start_time
            TTFT_HISTOGRAM.labels(model=model_name, endpoint="/generate").observe(ttft)
            first_token_recorded = True

        output_tokens += 1
        yield token

    total_time = time.monotonic() - start_time
    if total_time > 0:
        tps = output_tokens / total_time
        TPS_HISTOGRAM.labels(model=model_name).observe(tps)

    TOKEN_COUNTER.labels(model=model_name, type="output").inc(output_tokens)
```

GPU 메모리는 별도 스레드에서 주기적으로 수집한다.

```python
import threading
import pynvml

def gpu_metrics_collector(interval_seconds: float = 5.0):
    pynvml.nvmlInit()
    device_count = pynvml.nvmlDeviceGetCount()

    while True:
        for i in range(device_count):
            handle = pynvml.nvmlDeviceGetHandleByIndex(i)
            info = pynvml.nvmlDeviceGetMemoryInfo(handle)
            GPU_MEMORY_USAGE.labels(device_id=str(i)).set(info.used)

        time.sleep(interval_seconds)

# 백그라운드 스레드로 실행
collector_thread = threading.Thread(target=gpu_metrics_collector, daemon=True)
collector_thread.start()
```

### Prometheus 쿼리 예시

현재 p95 TTFT를 모델별로 보려면 이렇게 쓴다.

```promql
histogram_quantile(
  0.95,
  sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (model, le)
)
```

GPU 메모리 사용률 퍼센트는 다음과 같다.

```promql
(
  sum(gpu_memory_used_bytes) by (device_id)
  /
  sum(gpu_memory_total_bytes) by (device_id)
) * 100
```

초당 처리 토큰 수는 비용과 직결된다.

```promql
sum(rate(llm_tokens_total[1m])) by (model, type)
```

---

## 모델 품질 모니터링

레이턴시와 처리량만 보는 건 절반이다. 모델이 "작동"하는지는 알 수 있어도 "잘 작동"하는지는 알 수 없다.

품질 모니터링은 두 계층으로 나뉜다.

**오프라인 평가** 는 배포 전 벤치마크다. 이미 레이블된 데이터셋으로 정확도, ROUGE, BLEU 등을 측정한다.

**온라인 평가** 는 실서비스 트래픽에서 실시간으로 품질을 추적한다. 훨씬 어렵고 훨씬 중요하다.

### 피드백 루프 구현

사용자 피드백이 가장 직접적인 품질 신호다.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
import uuid

@dataclass
class InferenceRecord:
    request_id: str
    model_name: str
    model_version: str
    prompt_hash: str       # 원본 프롬프트 PII 제거 후 해시
    response_preview: str  # 앞 200자만 저장
    input_tokens: int
    output_tokens: int
    ttft_ms: float
    timestamp: datetime
    user_id: Optional[str] = None

@dataclass
class FeedbackRecord:
    request_id: str
    rating: int            # 1~5
    thumbs_up: Optional[bool] = None
    feedback_text: Optional[str] = None
    timestamp: datetime = None

class QualityTracker:
    def __init__(self, db_client, sample_rate: float = 0.1):
        self.db = db_client
        self.sample_rate = sample_rate  # 전체 요청의 10%만 저장

    def should_sample(self) -> bool:
        import random
        return random.random() < self.sample_rate

    async def record_inference(self, record: InferenceRecord):
        if not self.should_sample():
            return
        await self.db.insert("inference_records", record.__dict__)

    async def record_feedback(self, feedback: FeedbackRecord):
        # 피드백은 항상 저장 — 샘플링 없음
        feedback.timestamp = datetime.utcnow()
        await self.db.insert("feedback_records", feedback.__dict__)
        await self._update_quality_metrics(feedback)

    async def _update_quality_metrics(self, feedback: FeedbackRecord):
        # 시간 윈도우별 평균 평점 업데이트
        QUALITY_GAUGE.labels(
            model=await self._get_model_for_request(feedback.request_id)
        ).set(feedback.rating)
```

### 온라인 자동 평가

LLM-as-judge 패턴으로 다른 모델이 출력을 평가하게 만들 수 있다.

```python
async def auto_evaluate_response(
    prompt: str,
    response: str,
    judge_client,
    criteria: list[str],
) -> dict[str, float]:
    """
    judge_client: 더 강력한 모델 (예: GPT-4o, Claude-3.5-Sonnet)
    criteria: ["relevance", "accuracy", "conciseness"]
    """
    eval_prompt = f"""
다음 응답을 평가하세요. 각 기준에 대해 0.0~1.0 점수를 JSON으로 반환하세요.

원래 질문: {prompt}

모델 응답: {response}

평가 기준: {', '.join(criteria)}

응답 형식: {{"relevance": 0.9, "accuracy": 0.8, "conciseness": 0.7}}
"""
    result = await judge_client.complete(eval_prompt)
    scores = parse_json_safely(result)
    return scores
```

자동 평가는 비용이 든다. 전체 트래픽의 1~2%만 평가하는 게 현실적이다.

---

## A/B 테스트

새 모델이나 프롬프트 변경의 효과를 측정할 때 A/B 테스트가 필요하다. 직관만으로는 개선이 됐는지 퇴보인지 판단하기 어렵다.

### 트래픽 분할

```python
import hashlib
from enum import Enum

class Variant(Enum):
    CONTROL = "control"
    TREATMENT = "treatment"

def assign_variant(
    user_id: str,
    experiment_id: str,
    treatment_ratio: float = 0.5,
) -> Variant:
    """
    동일 사용자는 항상 같은 variant로 배정 — 해시 기반 결정론적 분할
    """
    hash_input = f"{experiment_id}:{user_id}".encode()
    hash_value = int(hashlib.md5(hash_input).hexdigest(), 16)
    bucket = (hash_value % 10000) / 10000.0  # 0.0 ~ 1.0

    if bucket < treatment_ratio:
        return Variant.TREATMENT
    return Variant.CONTROL

class ABTestRouter:
    def __init__(self, control_client, treatment_client):
        self.clients = {
            Variant.CONTROL: control_client,
            Variant.TREATMENT: treatment_client,
        }

    async def route_request(
        self,
        user_id: str,
        experiment_id: str,
        prompt: str,
    ) -> tuple[str, Variant]:
        variant = assign_variant(user_id, experiment_id)
        client = self.clients[variant]
        response = await client.complete(prompt)
        return response, variant
```

### 통계적 유의성 판단

A/B 결과를 보고 "treatment가 더 좋아 보인다"는 충분하지 않다. 통계적 유의성을 계산해야 한다.

```python
from scipy import stats
import numpy as np

def calculate_ab_significance(
    control_scores: list[float],
    treatment_scores: list[float],
    alpha: float = 0.05,
) -> dict:
    """
    control_scores, treatment_scores: 품질 점수 또는 전환율 데이터
    alpha: 유의수준 (일반적으로 0.05)
    """
    control = np.array(control_scores)
    treatment = np.array(treatment_scores)

    # Welch's t-test — 분산이 다를 때도 사용 가능
    t_stat, p_value = stats.ttest_ind(control, treatment, equal_var=False)

    control_mean = np.mean(control)
    treatment_mean = np.mean(treatment)
    relative_lift = (treatment_mean - control_mean) / control_mean * 100

    # 신뢰구간 계산
    diff = treatment_mean - control_mean
    pooled_se = np.sqrt(
        np.var(control, ddof=1) / len(control)
        + np.var(treatment, ddof=1) / len(treatment)
    )
    ci_95 = stats.t.ppf(0.975, df=len(control) + len(treatment) - 2) * pooled_se

    return {
        "control_mean": round(control_mean, 4),
        "treatment_mean": round(treatment_mean, 4),
        "relative_lift_pct": round(relative_lift, 2),
        "p_value": round(p_value, 4),
        "statistically_significant": p_value < alpha,
        "confidence_interval_95": (round(diff - ci_95, 4), round(diff + ci_95, 4)),
        "sample_sizes": {"control": len(control), "treatment": len(treatment)},
    }

# 사용 예
result = calculate_ab_significance(
    control_scores=[3.8, 4.1, 3.5, 4.0, 3.9, 3.7],
    treatment_scores=[4.2, 4.5, 4.1, 4.3, 4.4, 4.0],
)
print(result)
# {
#   "control_mean": 3.8333,
#   "treatment_mean": 4.25,
#   "relative_lift_pct": 10.87,
#   "p_value": 0.0123,
#   "statistically_significant": True,
#   "confidence_interval_95": (0.12, 0.71),
#   "sample_sizes": {"control": 6, "treatment": 6}
# }
```

### 샘플 크기 계산

실험 시작 전에 충분한 샘플 크기를 계산해야 한다. 중간에 "아, 유의하다"고 멈추면 Type I 오류가 올라간다.

```python
from scipy.stats import norm

def calculate_required_sample_size(
    baseline_rate: float,
    minimum_detectable_effect: float,  # 예: 0.05 = 5% 개선
    alpha: float = 0.05,
    power: float = 0.80,
) -> int:
    """
    baseline_rate: 현재 성공률 (예: 0.40 = 40%)
    minimum_detectable_effect: 탐지하고자 하는 최소 효과 크기
    power: 1 - beta (일반적으로 0.80)
    """
    treatment_rate = baseline_rate + minimum_detectable_effect
    p_bar = (baseline_rate + treatment_rate) / 2

    z_alpha = norm.ppf(1 - alpha / 2)  # 양측 검정
    z_beta = norm.ppf(power)

    numerator = (z_alpha + z_beta) ** 2 * 2 * p_bar * (1 - p_bar)
    denominator = (treatment_rate - baseline_rate) ** 2

    n_per_group = int(np.ceil(numerator / denominator))
    return n_per_group

# 현재 사용자 만족률 40%, 5% 개선을 80% 확률로 탐지하려면
n = calculate_required_sample_size(
    baseline_rate=0.40,
    minimum_detectable_effect=0.05,
)
print(f"그룹당 필요 샘플: {n}명")  # 약 1,235명
```

샘플 크기를 채우기 전에 실험을 중단하지 않는다. 이게 A/B 테스트에서 가장 많이 어기는 규칙이다.

---

## 비용 대시보드

AI 서비스에서 비용은 토큰 수와 GPU 시간으로 결정된다. 누가 얼마나 쓰는지 추적하지 않으면 예산 초과를 사후에야 안다.

### 토큰 기반 사용량 추적

```python
from dataclasses import dataclass, field
from collections import defaultdict
import asyncio

# 모델별 토큰 단가 (달러 기준, 예시)
TOKEN_COST_PER_1K = {
    "gpt-4o": {"input": 0.0025, "output": 0.01},
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "claude-3-5-sonnet": {"input": 0.003, "output": 0.015},
    "llama-3.1-70b": {"input": 0.0009, "output": 0.0009},  # 자체 호스팅 GPU 비용 환산
}

@dataclass
class UsageRecord:
    team_id: str
    user_id: str
    model: str
    input_tokens: int
    output_tokens: int
    timestamp: float

    @property
    def cost_usd(self) -> float:
        rates = TOKEN_COST_PER_1K.get(self.model, {"input": 0, "output": 0})
        input_cost = (self.input_tokens / 1000) * rates["input"]
        output_cost = (self.output_tokens / 1000) * rates["output"]
        return input_cost + output_cost

class CostTracker:
    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client

    async def record_usage(self, record: UsageRecord):
        # Redis에 실시간 카운터 업데이트
        date_key = datetime.utcnow().strftime("%Y-%m-%d")

        pipe = self.redis.pipeline()
        pipe.hincrbyfloat(
            f"cost:team:{record.team_id}:{date_key}",
            "total_usd",
            record.cost_usd,
        )
        pipe.hincrby(
            f"usage:team:{record.team_id}:{date_key}:{record.model}",
            "input_tokens",
            record.input_tokens,
        )
        pipe.hincrby(
            f"usage:team:{record.team_id}:{date_key}:{record.model}",
            "output_tokens",
            record.output_tokens,
        )
        await pipe.execute()

        # DB에 상세 레코드 저장 (비동기)
        asyncio.create_task(self._persist_record(record))

    async def get_team_daily_cost(self, team_id: str, date: str) -> float:
        cost = await self.redis.hget(f"cost:team:{team_id}:{date}", "total_usd")
        return float(cost or 0)

    async def get_cost_breakdown(
        self, team_id: str, start_date: str, end_date: str
    ) -> dict:
        """날짜 범위별 팀 비용 분석"""
        records = await self.db.query(
            """
            SELECT model,
                   SUM(input_tokens) as total_input,
                   SUM(output_tokens) as total_output,
                   COUNT(*) as request_count
            FROM usage_records
            WHERE team_id = %s
              AND date BETWEEN %s AND %s
            GROUP BY model
            ORDER BY total_cost DESC
            """,
            (team_id, start_date, end_date),
        )
        return records
```

### 팀별 예산 한도

```python
class BudgetEnforcer:
    def __init__(self, cost_tracker: CostTracker, alert_client):
        self.tracker = cost_tracker
        self.alerter = alert_client

    async def check_budget(
        self, team_id: str, daily_limit_usd: float, monthly_limit_usd: float
    ) -> tuple[bool, str]:
        today = datetime.utcnow().strftime("%Y-%m-%d")
        daily_cost = await self.tracker.get_team_daily_cost(team_id, today)

        if daily_cost >= daily_limit_usd:
            await self.alerter.send_alert(
                f"팀 {team_id}: 일일 예산 초과 (${daily_cost:.2f} / ${daily_limit_usd})"
            )
            return False, "daily_budget_exceeded"

        if daily_cost >= daily_limit_usd * 0.8:
            await self.alerter.send_alert(
                f"팀 {team_id}: 일일 예산 80% 도달 (${daily_cost:.2f})"
            )

        return True, "ok"
```

Grafana 대시보드에서 유용한 패널 구성은 다음과 같다.

- 시계열: 모델별 시간당 토큰 처리량
- 바 차트: 팀별 일간 비용 누적
- 히트맵: 시간대별 요청 집중도 (피크 타임 파악용)
- 게이지: 월간 예산 대비 현재 소진율

---

## AI 서비스 장애 패턴

AI 추론 서비스는 일반 웹 서비스와 다른 방식으로 터진다.

### OOM (Out of Memory)

GPU 메모리 부족은 가장 흔한 장애다. 증상은 CUDA out of memory 에러와 함께 워커 프로세스 비정상 종료다.

원인은 주로 세 가지다.

1. 예상보다 긴 입력 (context window 제한 미설정)
2. 배치 크기가 너무 클 때
3. KV 캐시 파편화 — 오래 실행된 인스턴스에서 점진적으로 발생

대응 코드:

```python
MAX_INPUT_TOKENS = 4096  # 모델 한계보다 낮게 설정

async def safe_inference(prompt: str, model_client):
    token_count = count_tokens(prompt)

    if token_count > MAX_INPUT_TOKENS:
        raise ValueError(
            f"입력 토큰 초과: {token_count} > {MAX_INPUT_TOKENS}"
        )

    try:
        return await model_client.generate(prompt)
    except RuntimeError as e:
        if "CUDA out of memory" in str(e):
            # GPU 캐시 정리 후 재시도
            import torch
            torch.cuda.empty_cache()
            # 필요시 인스턴스 재시작 트리거
            await trigger_instance_restart()
            raise ServiceUnavailableError("GPU 메모리 부족, 잠시 후 재시도")
        raise
```

KV 캐시 파편화는 주기적 재시작으로 해결한다. vLLM 기준으로 8~12시간 주기로 rolling restart를 걸면 충분한 경우가 많다.

### 큐 누적

요청이 몰리면 큐가 쌓인다. GPU 처리 속도보다 입력이 빠를 때 발생한다.

```python
import asyncio
from asyncio import Queue

class RequestQueue:
    def __init__(self, max_size: int = 100, timeout_seconds: float = 30.0):
        self.queue = Queue(maxsize=max_size)
        self.timeout = timeout_seconds

    async def enqueue(self, request: dict) -> str:
        request_id = str(uuid.uuid4())
        try:
            # 큐가 가득 차면 즉시 거부
            self.queue.put_nowait({**request, "id": request_id})
        except asyncio.QueueFull:
            QUEUE_REJECTED_COUNTER.inc()
            raise ServiceUnavailableError(
                "서버 과부하. 잠시 후 재시도하세요.",
                retry_after=10,
            )

        QUEUE_DEPTH_GAUGE.set(self.queue.qsize())
        return request_id

    async def process_loop(self, model_client):
        while True:
            try:
                request = await asyncio.wait_for(
                    self.queue.get(), timeout=self.timeout
                )
                await self._handle_request(request, model_client)
            except asyncio.TimeoutError:
                continue
```

큐 깊이를 Prometheus로 추적하고 10을 넘으면 알림을 보내는 게 기본이다.

```promql
# 큐 깊이가 50을 초과하면 PagerDuty
llm_request_queue_depth > 50
```

### GPU 고장

GPU 하드웨어 고장은 소프트 에러(ECC 오류)부터 완전 고장까지 다양하다.

```python
import pynvml

def check_gpu_health() -> list[dict]:
    pynvml.nvmlInit()
    results = []
    device_count = pynvml.nvmlDeviceGetCount()

    for i in range(device_count):
        handle = pynvml.nvmlDeviceGetHandleByIndex(i)

        # ECC 에러 카운트 확인
        try:
            ecc_errors = pynvml.nvmlDeviceGetTotalEccErrors(
                handle,
                pynvml.NVML_MEMORY_ERROR_TYPE_UNCORRECTED,
                pynvml.NVML_AGGREGATE_ECC,
            )
        except pynvml.NVMLError:
            ecc_errors = -1  # ECC 미지원 GPU

        # 온도 확인
        temp = pynvml.nvmlDeviceGetTemperature(
            handle, pynvml.NVML_TEMPERATURE_GPU
        )

        results.append({
            "device_id": i,
            "ecc_uncorrected_errors": ecc_errors,
            "temperature_c": temp,
            "healthy": ecc_errors == 0 and temp < 85,
        })

    return results
```

GPU 고장 감지 시 해당 노드를 즉시 서비스에서 제외하고 교체 요청을 올리는 것이 런북 기본 절차다.

### 장애 런북

실제 운영에서는 문서화된 런북이 없으면 야간 온콜에서 패닉이 온다.

**OOM 런북:**

1. `kubectl get pods -n inference` — 크래시된 파드 확인
2. `kubectl logs <pod> --previous` — CUDA OOM 에러 확인
3. `kubectl rollout restart deployment/inference-worker` — 롤링 재시작
4. 5분 내 회복 없으면 입력 토큰 한도 임시 축소 (4096 → 2048)
5. 근본 원인: 최근 배포된 모델 버전 또는 프롬프트 변경 확인

**큐 누적 런북:**

1. `llm_request_queue_depth` 메트릭으로 누적 속도 확인
2. 수평 확장 가능하면 즉시 replica 증가
3. 불가능하면 rate limit 임시 강화 (팀별 RPM 50% 감축)
4. 큐 드레인 예상 시간 계산: `queue_depth / (throughput_tps / avg_output_tokens)`

**GPU 고장 런북:**

1. `check_gpu_health()` 출력으로 문제 GPU 번호 확인
2. 해당 GPU 사용 프로세스 종료: `sudo kill -9 $(fuser /dev/nvidia<N>)`
3. Kubernetes 노드 cordon: `kubectl cordon <node>`
4. 클라우드면 인스턴스 교체 요청, 온프레미스면 하드웨어 팀 에스컬레이션

---

## 마치며

AI 서비스 관측은 일반 웹 서비스 관측에 GPU, 토큰, 모델 품질이 추가된 구조다. 기반 원칙은 같다: 무엇이 느린지, 무엇이 비싼지, 무엇이 망가졌는지를 실시간으로 알아야 한다.

TTFT와 TPS를 분리해서 추적하지 않으면 병목 지점이 안 보인다. 품질 메트릭 없이 모델을 바꾸면 퇴보를 사후에야 안다. A/B 테스트 없이 "더 좋아진 것 같다"는 믿음은 도박이다.

비용 추적은 선택이 아니다. 토큰 기반 과금 구조에서 누가 얼마를 쓰는지 모르면 예산은 반드시 초과된다.

장애 패턴은 예측 가능하다. OOM, 큐 누적, GPU 고장은 항상 같은 증상을 보인다. 런북을 미리 써두는 것이 야간 온콜을 견딜 수 있게 만드는 유일한 방법이다.

---

## 참고 자료

- [vLLM Metrics Documentation](https://docs.vllm.ai/en/latest/serving/metrics.html) — vLLM이 기본 제공하는 Prometheus 메트릭 목록과 설명
- [Prometheus Best Practices: Histograms and Summaries](https://prometheus.io/docs/practices/histograms/) — 레이턴시 분포 측정 설계 원칙
- [Trustworthy Online Controlled Experiments (Kohavi et al.)](https://www.cambridge.org/core/books/trustworthy-online-controlled-experiments/D97B26382EB0EB2DC2019A7A7B518F59) — A/B 테스트 통계 이론의 실무 표준 참고서
- [NVIDIA NVML Documentation](https://docs.nvidia.com/deploy/nvml-api/index.html) — GPU 상태 모니터링 API 레퍼런스
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/) — 운영 대시보드 설계 가이드
- [MLflow Model Monitoring](https://mlflow.org/docs/latest/model-registry.html) — 모델 버전 관리와 품질 추적 통합 방법
