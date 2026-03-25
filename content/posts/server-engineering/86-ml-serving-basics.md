---
title: "[AI / ML 서빙] 1편 — ML 서빙 기초: 모델이 API가 되기까지"
date: 2026-03-17T13:07:00+09:00
draft: false
tags: ["ML 서빙", "ONNX", "Triton", "TorchServe", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "학습(Training) vs 추론(Inference) 인프라 분리, 모델 직렬화 포맷(ONNX, TorchScript, SavedModel), 서빙 프레임워크 비교(TorchServe, Triton, TF Serving, BentoML), 모델 서버 API 설계(REST vs gRPC)와 배치 추론 vs 실시간 추론까지"
---

ML 모델을 학습시키는 것과 실제 서비스에서 사용하는 것은 완전히 다른 문제다.

연구자가 Jupyter Notebook에서 돌린 모델이 프로덕션 API로 바뀌는 과정에는 직렬화, 런타임 최적화, 배포 인프라, API 설계가 모두 얽혀 있다.

이 글은 서버 엔지니어 관점에서 그 과정 전체를 다룬다.

---

## 1. 학습 vs 추론 인프라: 왜 분리하는가

ML 파이프라인에서 학습(Training)과 추론(Inference)은 리소스 사용 패턴이 근본적으로 다르다.

### 리소스 특성 차이

**학습 단계**는 대규모 GPU 클러스터에서 수 시간~수 일 동안 실행된다. A100 80GB GPU 여러 장을 묶어 분산 학습하고, 체크포인트를 수백 GB 저장한다.

실험 재현성을 위해 데이터셋 버전 관리도 필요하다.

**추론 단계**는 지연 시간(latency)이 핵심이다. P99 레이턴시를 100ms 이하로 유지하면서 초당 수천 건의 요청을 처리해야 한다.

GPU가 없어도 최적화된 CPU 런타임으로 충분한 경우도 많다.

| 항목 | 학습 | 추론 |
|---|---|---|
| GPU 메모리 | 수십~수백 GB | 수 GB |
| 실행 시간 | 수 시간~수 일 | 수 ms~수 초 |
| 확장 방식 | 수직 확장 (더 큰 GPU) | 수평 확장 (인스턴스 복제) |
| 비용 패턴 | 일회성 대용량 | 지속적 소용량 |
| 주요 병목 | 연산 처리량 (TFLOPS) | 메모리 대역폭, 네트워크 I/O |

### 실제 아키텍처 예시

```
[학습 인프라]
데이터 레이크 → 전처리 파이프라인 → 학습 클러스터 (A100 x 8)
                                         ↓
                                   모델 레지스트리 (MLflow / DVC)
                                         ↓
[추론 인프라]
모델 레지스트리 → CI/CD 파이프라인 → 추론 서버 클러스터
                                       ↓
                               오토스케일링 그룹 (CPU/GPU 혼합)
```

학습 클러스터는 온디맨드 GPU 인스턴스로 필요할 때만 켜고, 추론 서버는 Reserved Instance나 Spot Fleet로 상시 운영하는 패턴이 일반적이다.

### 분리하지 않으면 생기는 문제

같은 인프라에서 학습과 추론을 함께 돌리면 학습 잡이 GPU 메모리를 전부 점유해 추론 서버가 OOM으로 죽는다. 학습 주기마다 서비스 장애가 반복된다.

---

## 2. 모델 직렬화 포맷

학습이 끝난 모델을 추론 서버에서 사용하려면 파일로 직렬화해야 한다. 포맷 선택이 이후 서빙 스택 전체에 영향을 준다.

### ONNX (Open Neural Network Exchange)

ONNX는 프레임워크 중립적인 중간 표현(IR)이다. PyTorch, TensorFlow, scikit-learn 등 여러 프레임워크에서 학습한 모델을 단일 포맷으로 내보낼 수 있다.

```python
import torch
import torch.onnx

model = MyModel()
model.load_state_dict(torch.load("checkpoint.pt"))
model.eval()

dummy_input = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    export_params=True,
    opset_version=17,
    do_constant_folding=True,
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={
        "input": {0: "batch_size"},
        "output": {0: "batch_size"},
    },
)
```

ONNX 파일은 protobuf 형식으로 저장된다. 내부에 계산 그래프(노드)와 가중치가 함께 들어간다.

```python
import onnx

model = onnx.load("model.onnx")
onnx.checker.check_model(model)

# 입출력 형태 확인
for input in model.graph.input:
    print(f"Input: {input.name}, shape: {[d.dim_value for d in input.type.tensor_type.shape.dim]}")
```

**장점**: 런타임 독립적, ONNX Runtime으로 CPU/GPU/NPU 모두 지원, Triton과 궁합이 좋음.

**단점**: 동적 제어 흐름(if/while이 모델 안에 있는 경우) 변환이 까다롭다. 일부 커스텀 연산자는 ONNX opset에 없어 별도 구현이 필요하다.

**실전 문제**: `dynamic_axes`를 설정하지 않으면 배치 크기가 고정된다. 추론 서버에서 배치 크기를 바꾸면 런타임 오류가 난다.

### TorchScript

TorchScript는 PyTorch 모델을 Python 인터프리터 없이 실행 가능한 형태로 컴파일한다. `torch.jit.trace` 또는 `torch.jit.script` 두 가지 방식이 있다.

```python
import torch

model = MyModel()
model.eval()

# trace 방식: 샘플 입력을 따라가며 그래프 기록
# 동적 분기가 없을 때 적합
traced = torch.jit.trace(model, torch.randn(1, 3, 224, 224))
traced.save("model_traced.pt")

# script 방식: Python 코드를 정적으로 분석
# 조건문, 루프가 있을 때 사용
@torch.jit.script
def forward(x: torch.Tensor) -> torch.Tensor:
    if x.sum() > 0:
        return model(x)
    return torch.zeros(1)
```

```python
# 로드 및 추론
loaded = torch.jit.load("model_traced.pt")
with torch.no_grad():
    output = loaded(torch.randn(1, 3, 224, 224))
```

**장점**: PyTorch 생태계와 완벽 호환. C++ LibTorch로 Python 없이 서빙 가능.

**단점**: PyTorch 버전에 종속적이다. 버전이 다른 서빙 환경에서 로드하면 호환성 오류가 난다.

### SavedModel (TensorFlow)

TensorFlow의 기본 직렬화 포맷이다. 가중치와 계산 그래프, 메타데이터를 디렉토리 구조로 저장한다.

```python
import tensorflow as tf

model = tf.keras.applications.ResNet50(weights="imagenet")

# 저장
model.save("saved_model/resnet50")

# 로드
loaded = tf.saved_model.load("saved_model/resnet50")
infer = loaded.signatures["serving_default"]

import numpy as np
input_data = tf.constant(np.random.randn(1, 224, 224, 3).astype(np.float32))
result = infer(input_data)
```

```
saved_model/resnet50/
├── assets/
├── fingerprint.pb
├── saved_model.pb       # 그래프 정의
└── variables/
    ├── variables.data-00000-of-00001
    └── variables.index
```

**장점**: TF Serving과 원클릭 통합. TensorFlow Lite, TensorFlow.js로 파생 변환이 쉽다.

**단점**: TensorFlow 생태계 전용이다. PyTorch 모델은 ONNX를 경유해야 한다.

### GGUF (GPT-Generated Unified Format)

LLM 서빙 특화 포맷이다. llama.cpp 생태계에서 표준으로 자리 잡았다. 양자화(Quantization) 정보를 포맷 내에 포함한다.

```bash
# Hugging Face 모델을 GGUF로 변환
git clone https://github.com/ggml-org/llama.cpp
pip install -r requirements.txt

python convert_hf_to_gguf.py \
    /path/to/llama-3-8b \
    --outtype f16 \
    --outfile llama-3-8b-f16.gguf

# 추가 양자화 (모델 크기 절반으로)
./llama-quantize llama-3-8b-f16.gguf llama-3-8b-q4_k_m.gguf Q4_K_M
```

Q4_K_M 양자화는 가중치를 4비트로 줄인다. 모델 크기가 16비트 대비 약 1/4로 줄고, 추론 속도가 빨라지지만 정확도가 소폭 하락한다.

**장점**: CPU만으로 LLM 추론 가능. 메모리 효율적.

**단점**: 범용 프레임워크에서 직접 지원하지 않는다. llama.cpp나 Ollama 등 특화 런타임이 필요하다.

---

## 3. 서빙 프레임워크 비교

### TorchServe

PyTorch 공식 서빙 솔루션이다. Meta와 AWS가 공동 개발했다.

```bash
# 모델 아카이브 생성
torch-model-archiver \
    --model-name resnet50 \
    --version 1.0 \
    --model-file model.py \
    --serialized-file model_traced.pt \
    --handler image_classifier \
    --export-path model-store

# 서버 시작
torchserve \
    --start \
    --model-store model-store \
    --models resnet50=resnet50.mar \
    --ts-config config.properties
```

`config.properties` 설정:

```properties
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082
number_of_netty_threads=32
job_queue_size=1000
default_workers_per_model=4
default_response_timeout=120
```

커스텀 핸들러로 전처리/후처리를 모델과 함께 패키징한다.

```python
from ts.torch_handler.base_handler import BaseHandler
import torch

class MyHandler(BaseHandler):
    def preprocess(self, data):
        # 입력 데이터를 텐서로 변환
        inputs = []
        for row in data:
            raw = row.get("data") or row.get("body")
            tensor = torch.frombuffer(raw, dtype=torch.float32).reshape(1, 3, 224, 224)
            inputs.append(tensor)
        return torch.cat(inputs)

    def postprocess(self, inference_output):
        return inference_output.tolist()
```

**성능 특성**: 단일 PyTorch 모델 서빙에 최적화되어 있다. 멀티 모델 동시 서빙 시 모델 간 GPU 메모리 격리가 약하다.

**선택 기준**: PyTorch 모델만 서빙하고, 간단한 REST API로 빠르게 배포할 때 적합하다.

### Triton Inference Server (NVIDIA)

NVIDIA가 개발한 고성능 추론 서버다. ONNX, TensorRT, TorchScript, TensorFlow SavedModel을 모두 지원한다.

```
model-repository/
├── resnet50/
│   ├── config.pbtxt
│   └── 1/
│       └── model.onnx
└── bert/
    ├── config.pbtxt
    └── 1/
        └── model.pt
```

`config.pbtxt` 설정:

```protobuf
name: "resnet50"
platform: "onnxruntime_onnx"
max_batch_size: 64

input [
  {
    name: "input"
    data_type: TYPE_FP32
    dims: [ 3, 224, 224 ]
  }
]

output [
  {
    name: "output"
    data_type: TYPE_FP32
    dims: [ 1000 ]
  }
]

dynamic_batching {
  preferred_batch_size: [ 8, 16, 32 ]
  max_queue_delay_microseconds: 5000
}

instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [ 0 ]
  }
]
```

```bash
# Docker로 실행
docker run --gpus all \
  -p 8000:8000 \
  -p 8001:8001 \
  -p 8002:8002 \
  -v $(pwd)/model-repository:/models \
  nvcr.io/nvidia/tritonserver:24.01-py3 \
  tritonserver --model-repository=/models
```

Python 클라이언트로 gRPC 추론:

```python
import tritonclient.grpc as grpcclient
import numpy as np

client = grpcclient.InferenceServerClient(url="localhost:8001")

input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)

inputs = [grpcclient.InferInput("input", input_data.shape, "FP32")]
inputs[0].set_data_from_numpy(input_data)

outputs = [grpcclient.InferRequestedOutput("output")]

response = client.infer(
    model_name="resnet50",
    inputs=inputs,
    outputs=outputs,
)

result = response.as_numpy("output")
print(result.shape)  # (1, 1000)
```

**성능 특성**: GPU 활용도가 가장 높다. 동적 배칭, 모델 파이프라인(앙상블), TensorRT 최적화를 지원한다. 멀티 모델 서빙에서 GPU 메모리를 세밀하게 제어할 수 있다.

**선택 기준**: 높은 처리량이 필요하거나, 여러 모델을 하나의 서버에서 관리해야 할 때 적합하다. 설정이 복잡하므로 초기 세팅 비용이 있다.

### TF Serving

TensorFlow 공식 서빙 서버다. SavedModel 포맷과 긴밀하게 통합된다.

```bash
docker run -p 8501:8501 \
  -v $(pwd)/saved_model:/models/resnet50 \
  -e MODEL_NAME=resnet50 \
  tensorflow/serving:2.14.0
```

REST API로 추론:

```bash
curl -X POST http://localhost:8501/v1/models/resnet50:predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[[...]]]}'
```

`model.config`로 멀티 모델 설정:

```protobuf
model_config_list {
  config {
    name: "resnet50"
    base_path: "/models/resnet50"
    model_platform: "tensorflow"
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
}
```

**성능 특성**: TensorFlow 모델에서는 안정적이고 예측 가능하다. 버전 관리와 A/B 테스트가 내장되어 있다.

**선택 기준**: TensorFlow 스택을 이미 쓰고 있을 때 가장 간단한 선택이다. PyTorch 환경에서는 오버헤드가 크다.

### BentoML

프레임워크 독립적인 ML 서빙 플랫폼이다. Python 코드로 서빙 로직을 정의한다.

```python
import bentoml
import numpy as np
from PIL import Image

@bentoml.service(
    resources={"gpu": 1},
    traffic={"timeout": 10},
)
class ImageClassifier:
    model_ref = bentoml.models.get("resnet50:latest")

    def __init__(self):
        import onnxruntime as ort
        self.session = ort.InferenceSession(
            self.model_ref.path_of("model.onnx"),
            providers=["CUDAExecutionProvider"],
        )

    @bentoml.api
    def predict(self, image: Image.Image) -> np.ndarray:
        img_array = np.array(image.resize((224, 224))).astype(np.float32)
        img_array = img_array.transpose(2, 0, 1)[np.newaxis, :]

        outputs = self.session.run(
            None,
            {"input": img_array},
        )
        return outputs[0]
```

```bash
# 모델 저장
bentoml models import resnet50.onnx

# 서빙 시작
bentoml serve service:ImageClassifier --port 3000

# 컨테이너 빌드
bentoml build
bentoml containerize image_classifier:latest
```

**성능 특성**: 개발 편의성이 높다. 프레임워크 전환이 자유롭다. 대신 Triton 대비 raw 처리량은 낮다.

**선택 기준**: 빠른 프로토타이핑, 또는 팀이 Python에 익숙하고 운영 복잡도를 낮추고 싶을 때 적합하다.

---

## 4. 모델 서버 API 설계

### REST vs gRPC 트레이드오프

| 항목 | REST (HTTP/JSON) | gRPC (HTTP/2 + protobuf) |
|---|---|---|
| 직렬화 오버헤드 | 높음 (JSON 파싱) | 낮음 (binary protobuf) |
| 레이턴시 | 상대적으로 높음 | 낮음 |
| 처리량 | 중간 | 높음 |
| 디버깅 | 쉬움 (curl, Postman) | 불편 (grpcurl 필요) |
| 브라우저 지원 | 기본 지원 | gRPC-Web 필요 |
| 스트리밍 | 제한적 | 양방향 스트리밍 지원 |

일반적으로 서비스 내부 통신(서버-서버)은 gRPC, 외부 공개 API는 REST를 선택한다.

### 입출력 직렬화: numpy → protobuf

gRPC 기반 서빙에서 numpy 배열을 protobuf로 직렬화하는 방법이다.

```protobuf
// inference.proto
syntax = "proto3";

message InferRequest {
  string model_name = 1;
  repeated InferTensor inputs = 2;
}

message InferTensor {
  string name = 1;
  repeated int64 shape = 2;
  string dtype = 3;       // "FP32", "INT64", ...
  bytes data = 4;         // raw bytes
}

message InferResponse {
  repeated InferTensor outputs = 1;
}
```

```python
import numpy as np
import grpc
import inference_pb2
import inference_pb2_grpc

def numpy_to_tensor(name: str, array: np.ndarray) -> inference_pb2.InferTensor:
    return inference_pb2.InferTensor(
        name=name,
        shape=list(array.shape),
        dtype="FP32",
        data=array.astype(np.float32).tobytes(),
    )

def tensor_to_numpy(tensor: inference_pb2.InferTensor) -> np.ndarray:
    dtype_map = {"FP32": np.float32, "INT64": np.int64}
    arr = np.frombuffer(tensor.data, dtype=dtype_map[tensor.dtype])
    return arr.reshape(tensor.shape)

channel = grpc.insecure_channel("localhost:50051")
stub = inference_pb2_grpc.InferenceStub(channel)

input_array = np.random.randn(1, 3, 224, 224).astype(np.float32)
request = inference_pb2.InferRequest(
    model_name="resnet50",
    inputs=[numpy_to_tensor("input", input_array)],
)

response = stub.Infer(request)
output = tensor_to_numpy(response.outputs[0])
```

### 헬스체크 API

추론 서버는 세 가지 상태를 구분해서 노출해야 한다.

```python
from fastapi import FastAPI, HTTPException
from enum import Enum

app = FastAPI()

class ModelStatus(str, Enum):
    READY = "READY"
    LOADING = "LOADING"
    UNAVAILABLE = "UNAVAILABLE"

@app.get("/health/live")
def liveness():
    """프로세스가 살아 있는지. kubelet이 이 엔드포인트로 컨테이너 재시작 여부를 결정한다."""
    return {"status": "ok"}

@app.get("/health/ready")
def readiness():
    """트래픽을 받을 준비가 됐는지. 모델 로딩 중이면 503을 반환한다."""
    if model_registry.is_ready():
        return {"status": "ok"}
    raise HTTPException(status_code=503, detail="model not loaded")

@app.get("/v1/models/{model_name}")
def model_metadata(model_name: str):
    """모델 메타데이터. 버전, 입출력 형태, 상태를 반환한다."""
    model = model_registry.get(model_name)
    if not model:
        raise HTTPException(status_code=404)

    return {
        "name": model_name,
        "versions": model.versions,
        "platform": model.platform,
        "inputs": [
            {"name": t.name, "shape": t.shape, "dtype": t.dtype}
            for t in model.inputs
        ],
        "outputs": [
            {"name": t.name, "shape": t.shape, "dtype": t.dtype}
            for t in model.outputs
        ],
        "status": ModelStatus.READY,
    }
```

Kubernetes 배포 시 헬스체크 설정:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 30    # 모델 로딩 시간 확보
  periodSeconds: 10
  failureThreshold: 6
```

`initialDelaySeconds`를 너무 짧게 설정하면 모델이 아직 로딩 중인데 트래픽이 들어와 오류가 발생한다. 모델 크기에 따라 30~120초까지 설정이 필요하다.

---

## 5. 배치 추론 vs 실시간 추론

### 레이턴시 vs 처리량

GPU는 병렬 연산에 특화되어 있다. 요청 하나를 즉시 처리(배치 크기=1)하면 GPU 활용도가 낮다. 여러 요청을 묶어 처리하면(배치 크기=N) 처리량이 크게 늘지만 레이턴시가 늘어난다.

```python
import time
import torch
import numpy as np

model = torch.jit.load("model_traced.pt").cuda()
model.eval()

def benchmark(batch_size: int, n_runs: int = 100):
    inputs = torch.randn(batch_size, 3, 224, 224).cuda()

    # 워밍업
    for _ in range(10):
        with torch.no_grad():
            model(inputs)

    torch.cuda.synchronize()
    start = time.perf_counter()

    for _ in range(n_runs):
        with torch.no_grad():
            model(inputs)

    torch.cuda.synchronize()
    elapsed = time.perf_counter() - start

    latency_ms = elapsed / n_runs * 1000
    throughput = batch_size * n_runs / elapsed

    print(f"Batch {batch_size:3d}: latency={latency_ms:.1f}ms, throughput={throughput:.0f} req/s")

for bs in [1, 4, 8, 16, 32, 64]:
    benchmark(bs)
```

```
Batch   1: latency=4.2ms,  throughput=238 req/s
Batch   4: latency=5.1ms,  throughput=784 req/s
Batch   8: latency=6.8ms,  throughput=1176 req/s
Batch  16: latency=10.3ms, throughput=1553 req/s
Batch  32: latency=18.7ms, throughput=1711 req/s
Batch  64: latency=35.2ms, throughput=1818 req/s
```

배치 크기 1에서 64로 늘리면 처리량이 7배 이상 늘지만 레이턴시도 8배 늘어난다.

### 동적 배칭 (Dynamic Batching) 원리

동적 배칭은 서버 측에서 짧은 시간 동안 들어온 요청을 모아 하나의 배치로 묶어 처리하는 기법이다.

```python
import asyncio
import time
from collections import deque
from dataclasses import dataclass
from typing import List

@dataclass
class Request:
    data: np.ndarray
    future: asyncio.Future

class DynamicBatcher:
    def __init__(self, model, max_batch_size=32, max_delay_ms=5.0):
        self.model = model
        self.max_batch_size = max_batch_size
        self.max_delay_ms = max_delay_ms
        self.queue: deque[Request] = deque()
        self.lock = asyncio.Lock()
        self._processing = False

    async def infer(self, data: np.ndarray) -> np.ndarray:
        loop = asyncio.get_event_loop()
        future = loop.create_future()

        async with self.lock:
            self.queue.append(Request(data=data, future=future))
            if not self._processing:
                self._processing = True
                asyncio.create_task(self._process_batch())

        return await future

    async def _process_batch(self):
        await asyncio.sleep(self.max_delay_ms / 1000)

        async with self.lock:
            batch_requests = []
            while self.queue and len(batch_requests) < self.max_batch_size:
                batch_requests.append(self.queue.popleft())
            self._processing = bool(self.queue)
            if self.queue:
                asyncio.create_task(self._process_batch())

        if not batch_requests:
            return

        # 배치 추론
        batch_input = np.stack([r.data for r in batch_requests])
        batch_output = self.model(batch_input)

        # 결과 분배
        for i, req in enumerate(batch_requests):
            req.future.set_result(batch_output[i])
```

Triton은 이 기능을 `config.pbtxt`에서 `dynamic_batching` 블록으로 설정한다. `max_queue_delay_microseconds`가 최대 대기 시간이다.

### 유스케이스별 선택 기준

**실시간 추론이 적합한 경우**
- 사용자가 결과를 즉시 기다리는 서비스 (검색, 추천, 챗봇)
- P99 레이턴시 SLA가 엄격한 경우 (< 100ms)
- 요청이 일정하게 들어오고 트래픽 예측이 가능한 경우

**배치 추론이 적합한 경우**
- 오프라인 처리: 대규모 데이터셋 임베딩 생성, 이미지 일괄 분류
- 비동기 파이프라인: 데이터 수집 후 야간에 처리
- 비용 절감이 우선인 경우 (GPU 시간 최소화)

```python
# 오프라인 배치 추론 예시 (Apache Spark + ONNX Runtime)
from pyspark.sql import SparkSession
import onnxruntime as ort
import numpy as np
import pandas as pd

def batch_predict(model_path: str):
    def predict_udf(iterator):
        session = ort.InferenceSession(model_path, providers=["CPUExecutionProvider"])

        for batch in iterator:
            images = np.stack(batch["image_data"].values)
            outputs = session.run(None, {"input": images})

            yield pd.DataFrame({
                "id": batch["id"],
                "prediction": outputs[0].tolist(),
            })

    return predict_udf

spark = SparkSession.builder.getOrCreate()
df = spark.read.parquet("s3://bucket/images/")

from pyspark.sql.functions import pandas_udf
result = df.mapInPandas(batch_predict("model.onnx"), schema="id string, prediction array<float>")
result.write.parquet("s3://bucket/predictions/")
```

---

---

## 참고 자료

- [ONNX Runtime Documentation](https://onnxruntime.ai/docs/) — ONNX 런타임 최적화 및 실행 공급자 설정
- [NVIDIA Triton Inference Server User Guide](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/) — 동적 배칭, 모델 앙상블, 성능 튜닝 가이드
- [TorchServe Documentation](https://pytorch.org/serve/) — 커스텀 핸들러 작성 및 배포 가이드
- [BentoML Documentation](https://docs.bentoml.com/) — 프레임워크 독립적 서빙 및 컨테이너 빌드
- [Efficient Large-Scale Inference — GPU Batching Strategies](https://developer.nvidia.com/blog/maximizing-deep-learning-inference-performance-with-nvidia-model-analyzer/) — NVIDIA 모델 분석기를 이용한 배칭 전략
- [llama.cpp GGUF Format Specification](https://github.com/ggml-org/gguf) — GGUF 포맷 구조 및 양자화 방식 상세 설명
