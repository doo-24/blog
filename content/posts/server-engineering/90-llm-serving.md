---
title: "[AI / ML 서빙] 5편 — LLM 서빙 인프라: 대규모 언어 모델 운영"
date: 2026-03-17T13:03:00+09:00
draft: false
tags: ["LLM", "vLLM", "KV 캐시", "PagedAttention", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "LLM 추론 특성(autoregressive, KV 캐시, 메모리 요구량), 서빙 프레임워크(vLLM, TGI, Ollama)와 PagedAttention 원리, 프롬프트 캐싱과 스트리밍 응답(SSE), LoRA/QLoRA 파인튜닝 인프라와 비용 관리까지"
---

대규모 언어 모델(LLM)은 기존 ML 모델과 추론 특성이 근본적으로 다르다. 이미지 분류 모델은 입력을 받아 한 번에 결과를 내지만, GPT 계열 모델은 토큰을 하나씩 순차 생성한다.

이 차이가 서빙 인프라 설계 전반을 바꾼다.

메모리 관리, 배칭 전략, 스트리밍 응답, 비용 구조 모두 LLM 특유의 방식이 필요하다.

---

## LLM 추론 특성: 왜 느린가

### Autoregressive Decoding

LLM은 토큰을 하나씩 생성한다. 100토큰짜리 응답을 만들려면 모델 포워드 패스를 100번 실행한다.

각 스텝에서 모델은 이전에 생성한 모든 토큰을 참조한다. 이게 핵심 병목이다.

```
입력: "서울의 수도는" (4토큰)
1스텝: "서울의 수도는" → "입니다" 예측
2스텝: "서울의 수도는 입니다" → "서울" 예측
...반복
```

이미지 분류는 배치 크기를 늘려도 GPU 활용률이 유지된다. LLM은 시퀀스 길이가 길어질수록 메모리와 연산이 함께 증가한다.

### Prefill vs Decode 단계

LLM 추론은 두 단계로 나뉜다.

**Prefill**: 입력 프롬프트 전체를 한 번에 처리한다. 병렬 연산이 가능해 GPU 활용률이 높다. 프롬프트 길이에 비례해 시간이 걸린다.

**Decode**: 토큰을 하나씩 생성한다. 각 스텝마다 모델 전체를 통과해야 한다. 메모리 대역폭이 병목이다.

실제 추론에서 TTFT(Time To First Token)는 prefill 시간이고, TPS(Tokens Per Second)는 decode 속도다. 두 지표가 독립적으로 움직인다.

### KV 캐시 메모리 계산

Attention 연산 시 각 토큰의 Key/Value 텐서를 캐싱한다. 다음 스텝에서 재계산하지 않기 위해서다.

KV 캐시 크기 공식:

```
KV 캐시 크기 = 2 × num_layers × num_heads × head_dim × seq_len × batch_size × dtype_bytes
```

앞의 `2`는 각 레이어마다 **Key**와 **Value** 텐서 두 가지를 캐싱하기 때문이다. Query는 현재 스텝에서만 쓰이고 버려지므로 캐시 대상이 아니다.

Llama-2-70B 기준 계산:

```
num_layers = 80
num_heads = 64 (GQA 적용 시 8 KV heads)
head_dim = 128
dtype = float16 (2 bytes)

요청 1개, 4096 토큰 기준:
2 × 80 × 8 × 128 × 4096 × 1 × 2 = 약 1.3GB
```

동시 요청 10개면 13GB가 KV 캐시만으로 사용된다. 모델 가중치(약 140GB)에 더해 GPU 메모리 압박이 심하다.

---

## 서빙 프레임워크 비교

### vLLM: PagedAttention

vLLM은 현재 가장 널리 쓰이는 LLM 서빙 엔진이다. 핵심은 PagedAttention이다.

**기존 방식의 문제**: KV 캐시를 연속된 메모리 블록으로 할당한다. 최대 시퀀스 길이를 사전에 예약해야 해서 메모리 낭비가 크다. 요청마다 최대 4096 토큰을 미리 잡으면 실제 사용량이 적어도 메모리가 묶인다.

**PagedAttention 원리**: OS 가상 메모리의 페이징 개념을 KV 캐시에 적용한다.

KV 캐시를 고정 크기 블록(기본 16토큰)으로 나눈다. 요청이 늘어나면 필요한 블록을 동적으로 할당한다. 블록은 연속될 필요가 없다. 논리적 블록 테이블이 물리적 위치를 추적한다.

```
논리 블록 [0, 1, 2, 3] → 물리 블록 [7, 2, 15, 3]
```

메모리 단편화가 거의 없다. 실험 결과 기존 대비 메모리 효율 2-4배 향상이 보고된다.

**Continuous Batching**: 기존 정적 배칭은 배치 내 모든 요청이 끝날 때까지 기다린다. 짧은 요청이 끝나도 긴 요청 때문에 GPU가 대기한다.

Continuous Batching은 요청이 완료되면 즉시 새 요청을 배치에 추가한다. GPU 활용률이 크게 올라간다.

```python
# vLLM 기본 서버 실행
from vllm import AsyncLLMEngine, AsyncEngineArgs, SamplingParams
from vllm.entrypoints.openai.api_server import app
import asyncio

engine_args = AsyncEngineArgs(
    model="meta-llama/Llama-2-70b-chat-hf",
    tensor_parallel_size=4,      # GPU 4개에 분산
    gpu_memory_utilization=0.90, # GPU 메모리 90% 사용
    max_model_len=4096,
    block_size=16,               # KV 캐시 블록 크기
)

engine = AsyncLLMEngine.from_engine_args(engine_args)
```

```bash
# CLI로 서버 실행
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-70b-chat-hf \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.9 \
  --max-model-len 4096 \
  --port 8000
```

vLLM은 OpenAI API 호환 엔드포인트를 제공한다. 기존 OpenAI SDK 코드를 base_url만 바꿔 재사용할 수 있다.

### TGI (Text Generation Inference)

Hugging Face가 개발한 서빙 프레임워크다. Rust로 작성된 고성능 서버와 Python 추론 엔진을 결합했다.

Flash Attention 2, Paged Attention을 지원한다. Safetensors 형식 모델 로딩이 빠르다.

```bash
# Docker로 TGI 실행
docker run --gpus all \
  -v $HOME/.cache/huggingface:/data \
  -p 8080:80 \
  ghcr.io/huggingface/text-generation-inference:2.0 \
  --model-id meta-llama/Llama-2-70b-chat-hf \
  --num-shard 4 \
  --max-input-length 2048 \
  --max-total-tokens 4096
```

```python
# TGI Python 클라이언트
from huggingface_hub import InferenceClient

client = InferenceClient("http://localhost:8080")
response = client.text_generation(
    "한국의 수도는",
    max_new_tokens=100,
    stream=True,
)

for token in response:
    print(token, end="", flush=True)
```

### Ollama

로컬 개발 환경에 특화된 도구다. 모델 관리, 실행, API 서버를 통합한다.

프로덕션 서빙보다 개발자 로컬 환경이나 소규모 배포에 적합하다.

```bash
# 설치 및 모델 실행
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull llama2:70b
ollama serve

# API 호출
curl http://localhost:11434/api/generate \
  -d '{"model": "llama2", "prompt": "서울의 수도는", "stream": true}'
```

### 프레임워크 선택 기준

| 항목 | vLLM | TGI | Ollama |
|------|------|-----|--------|
| 프로덕션 적합성 | 높음 | 높음 | 낮음 |
| 메모리 효율 | 최상 | 높음 | 보통 |
| 설치 복잡도 | 보통 | 낮음(Docker) | 매우 낮음 |
| 커스터마이징 | 높음 | 보통 | 낮음 |
| 멀티 어댑터 지원 | 지원 | 제한적 | 없음 |

---

## 프롬프트 캐싱

### Prefix Caching 원리

동일한 시스템 프롬프트를 쓰는 요청이 많다면, 매번 prefill을 반복하는 건 낭비다.

Prefix Caching은 공통 프롬프트 앞부분의 KV 캐시를 저장해 재사용한다. 캐시 히트 시 prefill 비용이 0에 가까워진다.

```
시스템 프롬프트 (1000토큰) + 사용자 입력 (50토큰)
→ 1000토큰 캐시 히트 → 50토큰만 prefill
```

vLLM은 자동 prefix caching을 지원한다.

```python
engine_args = AsyncEngineArgs(
    model="...",
    enable_prefix_caching=True,  # 자동 prefix caching 활성화
)
```

### 구현 패턴

시스템 프롬프트를 요청 앞에 고정한다. 캐시 히트율을 높이려면 프롬프트 순서가 중요하다.

```python
# 캐시 히트율을 높이는 프롬프트 구성
def build_prompt(system_prompt: str, user_input: str) -> str:
    # 공통 부분(system_prompt)을 항상 앞에 배치
    return f"<|system|>\n{system_prompt}\n<|user|>\n{user_input}\n<|assistant|>\n"

# RAG 패턴에서 문서를 system 뒤에 배치
def build_rag_prompt(
    system_prompt: str,
    documents: list[str],
    user_query: str,
) -> str:
    doc_context = "\n\n".join(documents)
    return (
        f"<|system|>\n{system_prompt}\n\n"
        f"참고 문서:\n{doc_context}\n"
        f"<|user|>\n{user_query}\n<|assistant|>\n"
    )
```

캐시 히트율을 모니터링한다.

```python
# vLLM 메트릭에서 캐시 히트율 확인
import httpx

async def get_cache_stats():
    async with httpx.AsyncClient() as client:
        resp = await client.get("http://localhost:8000/metrics")
        # vllm_cache_config_info, vllm_gpu_cache_usage_perc 등 확인
        return resp.text
```

---

## 스트리밍 응답

### SSE 구현

LLM 응답을 실시간으로 보여주려면 Server-Sent Events(SSE)가 적합하다.

HTTP 연결을 유지하면서 서버가 토큰을 생성할 때마다 청크를 전송한다. WebSocket보다 구현이 단순하고 단방향 통신에 맞다.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from vllm import AsyncLLMEngine, AsyncEngineArgs, SamplingParams
import asyncio
import json
import uuid

app = FastAPI()

engine_args = AsyncEngineArgs(model="meta-llama/Llama-2-7b-chat-hf")
engine = AsyncLLMEngine.from_engine_args(engine_args)

@app.post("/v1/chat/completions")
async def chat_completions(request: dict):
    prompt = request["messages"][-1]["content"]
    stream = request.get("stream", False)

    sampling_params = SamplingParams(
        temperature=request.get("temperature", 0.7),
        max_tokens=request.get("max_tokens", 512),
    )

    request_id = str(uuid.uuid4())

    if stream:
        return StreamingResponse(
            stream_response(engine, prompt, sampling_params, request_id),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # nginx 버퍼링 비활성화
            },
        )

    # 비스트리밍 응답
    results = []
    async for output in engine.generate(prompt, sampling_params, request_id):
        results.append(output)

    final = results[-1]
    return {
        "choices": [{"message": {"content": final.outputs[0].text}}]
    }


async def stream_response(engine, prompt, sampling_params, request_id):
    """토큰 레벨 SSE 스트림 생성기"""
    prev_text = ""

    async for output in engine.generate(prompt, sampling_params, request_id):
        current_text = output.outputs[0].text
        delta = current_text[len(prev_text):]  # 새로 생성된 토큰만 추출
        prev_text = current_text

        if delta:
            chunk = {
                "choices": [{
                    "delta": {"content": delta},
                    "finish_reason": None,
                }]
            }
            yield f"data: {json.dumps(chunk, ensure_ascii=False)}\n\n"

    # 완료 신호
    done_chunk = {
        "choices": [{"delta": {}, "finish_reason": "stop"}]
    }
    yield f"data: {json.dumps(done_chunk)}\n\n"
    yield "data: [DONE]\n\n"
```

### 클라이언트 측 스트림 처리

```python
import httpx
import json

async def call_streaming_llm(prompt: str):
    async with httpx.AsyncClient(timeout=120) as client:
        async with client.stream(
            "POST",
            "http://localhost:8000/v1/chat/completions",
            json={
                "messages": [{"role": "user", "content": prompt}],
                "stream": True,
                "max_tokens": 512,
            },
        ) as response:
            async for line in response.aiter_lines():
                if not line.startswith("data: "):
                    continue
                data = line[6:]  # "data: " 제거
                if data == "[DONE]":
                    break
                try:
                    chunk = json.loads(data)
                    delta = chunk["choices"][0]["delta"].get("content", "")
                    if delta:
                        print(delta, end="", flush=True)
                except json.JSONDecodeError:
                    continue
```

### nginx 설정

스트리밍 응답에서 nginx 버퍼링을 꺼야 한다. 버퍼링이 켜져 있으면 토큰이 쌓였다가 한꺼번에 전달된다.

```nginx
location /v1/ {
    proxy_pass http://llm_backend;
    proxy_buffering off;           # 버퍼링 비활성화
    proxy_cache off;
    proxy_read_timeout 300s;       # 긴 응답 타임아웃
    proxy_send_timeout 300s;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding on;
}
```

---

## LoRA/QLoRA 파인튜닝 인프라

### LoRA 개요

전체 모델 가중치를 파인튜닝하면 70B 모델 기준 수백 GB GPU 메모리가 필요하다. LoRA는 이를 우회한다.

원본 가중치 행렬 W(d×k)를 고정하고, 저랭크 행렬 A(d×r)와 B(r×k)만 학습한다. r은 보통 4~64로 설정한다.

```
W_new = W + BA  (r << min(d, k))
```

70B 모델 기준 r=16이면 어댑터 크기가 약 600MB다. 전체 대비 0.4% 수준이다.

QLoRA는 LoRA에 4비트 양자화를 더한다. 70B 모델을 A100 80GB 1개로 파인튜닝할 수 있다.

### 파인튜닝 코드

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, TaskType
import torch

# 4비트 양자화 설정
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
)

# 모델 로드
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto",
)

# LoRA 설정
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                  # 랭크
    lora_alpha=32,         # 스케일링 파라미터
    target_modules=[       # 어댑터를 붙일 레이어
        "q_proj", "v_proj",
        "k_proj", "o_proj",
    ],
    lora_dropout=0.05,
    bias="none",
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# 출력: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.06

# 어댑터 저장
model.save_pretrained("./my-lora-adapter")
```

### 어댑터 핫스왑

여러 태스크별 어댑터를 동적으로 교체하는 패턴이다. 모델 가중치는 GPU에 고정하고 어댑터만 교체한다.

vLLM은 LoRA 멀티 어댑터 서빙을 지원한다.

```bash
# vLLM에서 LoRA 어댑터 서빙
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-7b-hf \
  --enable-lora \
  --lora-modules \
    sql-adapter=./adapters/sql-lora \
    code-adapter=./adapters/code-lora \
  --max-loras 4 \
  --max-lora-rank 64
```

```python
# 요청 시 어댑터 지정
import openai

client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="none")

# SQL 어댑터 사용
response = client.chat.completions.create(
    model="sql-adapter",  # 어댑터 이름으로 모델 지정
    messages=[{"role": "user", "content": "사용자 테이블에서 활성 사용자를 조회하는 SQL 작성"}],
)

# 코드 어댑터 사용
response = client.chat.completions.create(
    model="code-adapter",
    messages=[{"role": "user", "content": "파이썬으로 퀵정렬 구현"}],
)
```

### 멀티 어댑터 서빙 아키텍처

```
                    ┌─────────────────────┐
클라이언트           │   API Gateway / LB   │
   │                └──────────┬──────────┘
   │                           │ 태스크 유형에 따라 라우팅
   │                ┌──────────▼──────────┐
   │                │   vLLM 서버          │
   │                │   (기본 모델 상주)    │
   │                │                     │
   │                │  ┌───────────────┐  │
   │                │  │ 어댑터 캐시    │  │
   │                │  │ sql-lora      │  │
   │                │  │ code-lora     │  │
   │                │  │ translate-lora│  │
   │                │  └───────────────┘  │
   │                └─────────────────────┘
```

---

## 비용 관리

### 토큰당 비용 산정

LLM 운영 비용의 핵심은 GPU 시간당 처리 토큰 수다.

```python
# 비용 계산 예시
class LLMCostCalculator:
    def __init__(self, gpu_hourly_cost: float, tokens_per_second: float):
        self.gpu_hourly_cost = gpu_hourly_cost  # 예: A100 $3.0/시간
        self.tokens_per_second = tokens_per_second  # 실측값

    def cost_per_million_tokens(self) -> float:
        tokens_per_hour = self.tokens_per_second * 3600
        return (self.gpu_hourly_cost / tokens_per_hour) * 1_000_000

    def estimate_request_cost(
        self,
        prompt_tokens: int,
        completion_tokens: int,
        prefill_tps: float = None,
    ) -> float:
        # 단순화: 완성 토큰 기준
        total_tokens = prompt_tokens + completion_tokens
        cost_per_token = self.gpu_hourly_cost / (self.tokens_per_second * 3600)
        return total_tokens * cost_per_token


# A100 80GB, Llama-2-70B 기준
calc = LLMCostCalculator(
    gpu_hourly_cost=3.0,       # $/시간
    tokens_per_second=25.0,    # decode TPS (실측)
)
print(f"백만 토큰당 비용: ${calc.cost_per_million_tokens():.2f}")
# 백만 토큰당 비용: $33.33

# 요청당 비용
cost = calc.estimate_request_cost(
    prompt_tokens=500,
    completion_tokens=200,
)
print(f"요청당 비용: ${cost:.5f}")
```

배칭이 비용에 미치는 영향이 크다. 배치 크기를 높이면 GPU 활용률이 올라가 토큰당 비용이 낮아진다.

### Spot GPU 활용

AWS, GCP, Azure 모두 Spot/Preemptible GPU 인스턴스를 제공한다. 온디맨드 대비 60-90% 저렴하다. 대신 중단 가능성이 있다.

LLM 서빙에서 Spot을 쓰려면 체크포인트 없이 요청 레벨 재시도를 구현한다.

```python
# Spot 중단 대응 패턴
import asyncio
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    reraise=True,
)
async def call_llm_with_retry(
    client: httpx.AsyncClient,
    prompt: str,
    endpoint: str,
) -> str:
    """Spot 인스턴스 중단 시 자동 재시도"""
    try:
        response = await client.post(
            endpoint,
            json={"prompt": prompt, "max_tokens": 512},
            timeout=30.0,
        )
        response.raise_for_status()
        return response.json()["text"]
    except (httpx.ConnectError, httpx.RemoteProtocolError) as e:
        # 인스턴스 중단으로 인한 연결 오류
        raise  # tenacity가 재시도 처리
```

멀티 리전 또는 멀티 클라우드에 인스턴스를 분산하면 Spot 중단이 동시에 발생할 가능성이 낮아진다.

```yaml
# Kubernetes Deployment: Spot + On-demand 혼합
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-serving
spec:
  replicas: 4
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 70
            preference:
              matchExpressions:
              - key: node.kubernetes.io/capacity-type
                operator: In
                values: ["spot"]
          - weight: 30
            preference:
              matchExpressions:
              - key: node.kubernetes.io/capacity-type
                operator: In
                values: ["on-demand"]
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        resources:
          limits:
            nvidia.com/gpu: "1"
```

### 비용 절감 전략 요약

**모델 양자화**: FP16 → INT8 → INT4 순으로 메모리가 절반씩 줄어든다. 품질 손실을 측정하고 허용 범위 내에서 적용한다.

**요청 배칭**: 지연 허용 가능한 태스크는 배치로 묶어 처리한다. 실시간 응답이 필요한 태스크와 분리 운영한다.

**모델 선택**: 태스크 복잡도에 따라 모델을 나눈다. 분류, 요약처럼 단순한 태스크는 7B 모델로도 충분한 경우가 많다. 70B를 모든 곳에 쓸 필요는 없다.

**Prefix Caching 활성화**: 시스템 프롬프트가 긴 애플리케이션에서 효과가 크다. RAG 패턴에서 문서 캐싱에도 적용된다.

---

## 참고 자료

- [vLLM 공식 문서](https://docs.vllm.ai/) — PagedAttention, Continuous Batching, LoRA 서빙 가이드
- [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180) — vLLM 핵심 논문
- [QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314) — 4비트 양자화 파인튜닝 방법론
- [Text Generation Inference (TGI) 문서](https://huggingface.co/docs/text-generation-inference) — Hugging Face TGI 설정 및 운영
- [PEFT 라이브러리](https://huggingface.co/docs/peft) — LoRA, QLoRA 구현 레퍼런스
