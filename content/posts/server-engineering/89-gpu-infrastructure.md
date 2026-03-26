---
title: "[AI / ML 서빙] 4편 — GPU 인프라와 최적화: 추론 성능 끌어올리기"
date: 2026-03-17T13:04:00+09:00
draft: false
tags: ["GPU", "TensorRT", "양자화", "CUDA", "서버"]
series: ["AI / ML 서빙"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 13
summary: "GPU 인스턴스 선택(T4, A10G, A100, H100)과 CPU vs GPU 추론 기준, 모델 최적화(양자화, 프루닝, 디스틸레이션, TensorRT), 동적 배칭과 오토스케일링, K8s GPU 스케줄링(NVIDIA Device Plugin, 타임슬라이싱)까지"
---

추론 서비스의 지연과 비용은 결국 GPU를 얼마나 잘 다루느냐에 달려 있다.

인스턴스 선택을 잘못하면 과금만 나가고, 최적화를 건너뛰면 처리량이 절반으로 줄어든다.

이 글은 GPU 인프라 설계부터 K8s 스케줄링까지, 서버 엔지니어가 실제로 다루는 결정들을 순서대로 짚는다.

---

## 1. GPU 인스턴스 선택

### T4, A10G, A100, H100 비교

AWS 기준으로 주로 사용하는 GPU 인스턴스 네 종류를 정리한다.

| GPU | 메모리 | FP16 TFLOPS | INT8 TOPS | 적합한 용도 |
|-----|--------|-------------|-----------|-------------|
| T4 | 16 GB | 65 | 130 | 소형 모델, 비용 최적화 |
| A10G | 24 GB | 125 | 250 | 중형 모델, 균형형 |
| A100 (40G) | 40 GB | 312 | 624 | 대형 모델, 배치 처리 |
| H100 | 80 GB | 1979 | 3958 | LLM, 최고 처리량 필요 시 |

T4는 g4dn 인스턴스에 탑재된다. 시간당 비용이 낮아 트래픽이 예측 가능하고 모델 크기가 7B 미만이면 실용적이다.

A10G는 g5 인스턴스 계열이다. 24 GB 메모리로 13B 모델까지 FP16으로 올릴 수 있고, T4 대비 처리량이 두 배 가까이 나온다.

A100은 p4d 인스턴스에 포함된다. NVLink로 묶인 다중 GPU 구성에서 모델 병렬 처리에 강점이 있다.

H100은 p5 인스턴스에 탑재된다. Transformer Engine이 내장되어 FP8 연산을 지원한다. LLM 서빙에서 A100 대비 토큰 생성 속도가 3배 이상 차이 나는 경우가 많다.

### CPU vs GPU 추론 판단 기준

GPU가 항상 답은 아니다. 아래 조건 중 하나라도 해당하면 CPU 추론이 합리적이다.

- 모델 파라미터 수가 100M 미만이고 배치 크기가 1인 단순 분류 모델
- 요청 빈도가 초당 10 미만이고 지연 요건이 200ms 이상
- 온프레미스 GPU 자원이 없고 클라우드 비용 제약이 엄격한 경우

GPU가 필요한 신호는 명확하다. 배치 처리량이 CPU 한계를 초과하거나, 실시간 응답 요건이 50ms 미만이거나, 모델 크기가 1B 파라미터를 넘는 경우다.

판단 공식 하나를 기억해 두면 편하다.

```
GPU 비용 / 시간 < CPU 절약 처리량 / 시간 × 요청당 수익
```

수익 기여가 낮은 배치 작업은 CPU로 야간에 처리하고, SLA가 붙은 실시간 추론만 GPU를 쓰는 혼합 전략이 가장 흔하다.

---

## 2. 모델 최적화 기법

### 양자화 (Quantization)

양자화는 파라미터의 수치 정밀도를 낮춰 메모리와 연산량을 줄이는 기법이다.

FP32를 FP16으로 내리면 메모리가 절반, 처리량은 약 1.5~2배 오른다. 대부분의 모델에서 정확도 손실이 0.1% 미만이므로 기본 적용해야 한다.

INT8은 FP16보다 추가로 절반을 더 아낀다. 보정(calibration) 과정이 필요하다.

```python
import torch

# FP16 변환 (기본)
model = model.half().cuda()

# torch.quantization을 이용한 동적 INT8
import torch.quantization

model_int8 = torch.quantization.quantize_dynamic(
    model,
    {torch.nn.Linear},
    dtype=torch.qint8
)
```

Hugging Face transformers에서는 `bitsandbytes`로 4비트 양자화까지 지원한다.

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto",
)
```

4비트 양자화를 적용한 7B 모델은 T4 16 GB에서도 동작한다. 원본 FP16 기준으로는 14 GB가 필요해 T4에 올라가지 않는다.

### 프루닝 (Pruning)

프루닝은 중요도가 낮은 가중치를 0으로 만들어 연산량을 줄이는 방법이다.

비구조적 프루닝(unstructured pruning)은 개별 가중치 수준에서 제거한다. 구조적 프루닝(structured pruning)은 헤드, 레이어 단위로 제거한다. 실제 추론 가속화에는 구조적 프루닝이 더 유효하다.

```python
import torch.nn.utils.prune as prune

# 선형 레이어 가중치의 20% 제거 (L1 기준)
prune.l1_unstructured(model.fc1, name='weight', amount=0.2)

# 프루닝 고정 (mask 제거, 실제 0으로 만들기)
prune.remove(model.fc1, 'weight')
```

프루닝 후에는 반드시 파인튜닝으로 정확도를 회복해야 한다. 프루닝만 적용하고 재훈련을 건너뛰면 성능이 급격히 떨어진다.

### 디스틸레이션 (Knowledge Distillation)

대형 교사(teacher) 모델의 출력을 소형 학생(student) 모델이 모방하도록 훈련하는 방식이다.

BERT-base(110M)를 DistilBERT(66M)로 줄이면 추론 속도가 60% 빨라지고 정확도는 97% 수준을 유지한다.

디스틸레이션 손실 함수 구조는 다음과 같다. 두 항으로 구성된다. **소프트 레이블 손실**(KL divergence)은 교사 모델의 출력 분포를 학생이 따라가도록 하여 클래스 간 유사도 같은 암묵적 지식을 전달하고, **하드 레이블 손실**(CrossEntropy)은 실제 정답 레이블을 기준으로 정확도를 유지한다. `temperature`를 높이면 교사 출력의 확률 분포가 고르게 펴져 정보량이 증가한다. `temperature ** 2` 스케일링은 temperature 변화에 따른 그래디언트 크기 변동을 보정하기 위해 Hinton 등이 제안한 방식이다.

```python
import torch.nn.functional as F

def distillation_loss(student_logits, teacher_logits, labels, temperature=4.0, alpha=0.7):
    # 소프트 레이블 손실 (KL divergence)
    soft_loss = F.kl_div(
        F.log_softmax(student_logits / temperature, dim=-1),
        F.softmax(teacher_logits / temperature, dim=-1),
        reduction='batchmean'
    ) * (temperature ** 2)

    # 하드 레이블 손실 (CrossEntropy)
    hard_loss = F.cross_entropy(student_logits, labels)

    return alpha * soft_loss + (1 - alpha) * hard_loss
```

디스틸레이션은 양자화, 프루닝보다 훈련 비용이 크지만 모델 크기를 50% 이상 줄이는 경우 정확도 유지 면에서 가장 안정적이다.

### TensorRT 변환

TensorRT는 NVIDIA가 제공하는 추론 최적화 런타임이다. 레이어 퓨전, 커널 오토튜닝, 정밀도 최적화를 자동으로 적용한다.

PyTorch 모델을 TensorRT로 변환하는 흐름은 다음과 같다.

```python
import tensorrt as trt
import torch
from torch2trt import torch2trt

model = model.eval().cuda()
dummy_input = torch.ones(1, 3, 224, 224).cuda()

# FP16 모드로 변환
model_trt = torch2trt(
    model,
    [dummy_input],
    fp16_mode=True,
    max_workspace_size=1 << 25  # 32 MB
)

torch.save(model_trt.state_dict(), 'model_trt.pth')
```

ONNX를 경유하는 방법도 널리 쓰인다. PyTorch → ONNX → TensorRT 순서로 변환한다.

```bash
# PyTorch를 ONNX로 변환
python -c "
import torch
model = ...  # 모델 로드
dummy = torch.randn(1, 3, 224, 224).cuda()
torch.onnx.export(model, dummy, 'model.onnx', opset_version=17)
"

# ONNX를 TensorRT 엔진으로 변환
trtexec \
  --onnx=model.onnx \
  --saveEngine=model.trt \
  --fp16 \
  --workspace=512
```

ResNet-50 기준으로 FP32 PyTorch 대비 TensorRT FP16이 3~4배, INT8이 5~6배 빠르다. 변환 후에는 동일 입력으로 출력값 오차를 반드시 확인해야 한다.

---

## 3. 동적 배칭과 큐 설계

### 배칭 전략

GPU 연산은 배치 크기가 클수록 GPU 활용률(utilization)이 올라간다. 요청 하나씩 처리하면 GPU가 대부분의 시간을 유휴 상태로 보낸다.

동적 배칭(dynamic batching)은 일정 시간이나 요청 수 조건이 채워지면 묶어서 처리하는 방식이다.

```python
import asyncio
from collections import deque
import time

class DynamicBatcher:
    def __init__(self, max_batch_size=32, max_wait_ms=20):
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self.queue = deque()
        self.lock = asyncio.Lock()

    async def add_request(self, request):
        future = asyncio.get_event_loop().create_future()
        async with self.lock:
            self.queue.append((request, future))
        return await future

    async def process_loop(self, model):
        while True:
            await asyncio.sleep(self.max_wait_ms / 1000)
            async with self.lock:
                if not self.queue:
                    continue
                batch_items = []
                while self.queue and len(batch_items) < self.max_batch_size:
                    batch_items.append(self.queue.popleft())

            requests = [item[0] for item in batch_items]
            futures = [item[1] for item in batch_items]

            results = model.infer_batch(requests)

            for future, result in zip(futures, results):
                future.set_result(result)
```

Triton Inference Server는 동적 배칭을 설정 파일로 제어한다.

```
# config.pbtxt
dynamic_batching {
  preferred_batch_size: [8, 16, 32]
  max_queue_delay_microseconds: 20000
}
```

`max_queue_delay_microseconds`를 낮추면 지연이 줄지만 배치 크기도 작아진다. 지연 SLA에 맞춰 조정해야 한다.

### GPU 메모리 관리

GPU 메모리는 한 번 할당하면 해제가 느리다. 모델 가중치, 활성화 맵, KV 캐시가 함께 올라가므로 메모리 계획이 필요하다.

LLM에서 KV 캐시는 시퀀스 길이에 비례해 증가한다. vLLM이 구현한 PagedAttention은 KV 캐시를 페이지 단위로 관리해 단편화를 줄인다.

```bash
# vLLM 서버 실행 시 GPU 메모리 비율 지정
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-13b-chat-hf \
  --gpu-memory-utilization 0.90 \
  --max-model-len 4096
```

`gpu-memory-utilization`을 0.90으로 지정하면 가용 메모리의 90%를 KV 캐시와 모델에 할당한다. 0.95 이상으로 올리면 OOM 위험이 있다.

CUDA 메모리 단편화를 줄이려면 추론 전 CUDA 캐시를 명시적으로 비워주는 것이 좋다.

```python
import torch

torch.cuda.empty_cache()
torch.cuda.synchronize()
```

---

## 4. 오토스케일링

### GPU 메트릭 기반 HPA

표준 CPU/메모리 기반 HPA는 GPU 워크로드에 적합하지 않다. GPU 활용률과 큐 깊이를 기반으로 스케일해야 한다.

DCGM Exporter로 GPU 메트릭을 Prometheus에 노출한다.

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace monitoring
```

Prometheus에서 확인 가능한 주요 메트릭은 다음과 같다.

```
DCGM_FI_DEV_GPU_UTIL       # GPU 코어 활용률 (%)
DCGM_FI_DEV_MEM_COPY_UTIL  # 메모리 대역폭 활용률 (%)
DCGM_FI_DEV_FB_USED        # 사용 중인 프레임버퍼 메모리 (MB)
```

Prometheus Adapter를 통해 HPA에 커스텀 메트릭을 연결한다.

```yaml
# prometheus-adapter configmap 일부
rules:
  - seriesQuery: 'DCGM_FI_DEV_GPU_UTIL{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)$"
      as: "gpu_utilization"
    metricsQuery: 'avg(DCGM_FI_DEV_GPU_UTIL{<<.LabelMatchers>>})'
```

HPA 설정에서 GPU 활용률 70%를 기준으로 스케일아웃한다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-server
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: gpu_utilization
        target:
          type: AverageValue
          averageValue: "70"
```

### KEDA를 이용한 큐 기반 스케일링

요청 큐 깊이로 스케일하면 GPU 활용률보다 반응이 빠르다. KEDA(Kubernetes Event-driven Autoscaling)는 외부 메트릭 소스를 직접 지원한다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inference-scaledobject
spec:
  scaleTargetRef:
    name: inference-server
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: inference_queue_depth
        query: sum(inference_request_queue_depth)
        threshold: "10"   # 큐에 요청 10개 이상이면 스케일아웃
```

큐 깊이 임계값 10은 배치 크기와 처리 시간을 바탕으로 결정한다. 배치 크기가 32이고 처리 시간이 100ms라면, 큐가 10개 쌓이는 시간 내에 스케일아웃이 완료되어야 한다.

스케일인(scale-in)은 느리게 가져간다. GPU 파드를 빠르게 줄이면 진행 중인 추론 요청이 중단된다. KEDA `cooldownPeriod`를 300초 이상으로 설정하는 게 일반적이다.

```yaml
spec:
  cooldownPeriod: 300
  pollingInterval: 15
```

---

## 5. K8s GPU 스케줄링

### NVIDIA Device Plugin

K8s에서 GPU를 리소스로 노출하려면 NVIDIA Device Plugin이 필요하다.

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml
```

설치 후 노드에서 GPU 리소스가 보인다.

```bash
kubectl get nodes -o json | jq '.items[].status.allocatable | select(."nvidia.com/gpu")'
# 출력 예: {"nvidia.com/gpu": "1"}
```

파드 스펙에서 GPU 리소스를 요청한다.

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1
```

requests와 limits를 동일하게 지정해야 한다. GPU는 정수 단위로만 할당되며, 부분 할당은 기본적으로 지원되지 않는다.

### 타임슬라이싱 (Time-slicing)

GPU 하나를 여러 파드가 시간 분할로 나누어 쓰는 방식이다. 처리량보다 비용 효율이 중요한 상황에서 쓴다.

Device Plugin ConfigMap으로 타임슬라이싱을 활성화한다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: nvidia-device-plugin
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 4
```

위 설정은 GPU 하나를 4개로 노출한다. 각 파드는 `nvidia.com/gpu: 1`을 요청하지만 실제로는 GPU 시간의 1/4을 얻는다.

타임슬라이싱은 메모리를 나누지 않는다. 4개 파드가 동시에 떠 있으면 GPU 메모리 합산이 GPU 총 메모리를 초과하면 안 된다.

### MIG (Multi-Instance GPU)

A100과 H100에서 지원하는 기능이다. GPU를 물리적으로 독립된 인스턴스로 분할한다. 타임슬라이싱과 달리 메모리와 컴퓨트를 완전히 격리한다.

A100 40 GB를 7개의 MIG 인스턴스로 분할하면 각각 5 GB를 갖는다.

```bash
# MIG 모드 활성화
nvidia-smi -i 0 -mig 1

# 7개 1g.5gb 인스턴스 생성
nvidia-smi mig -cgi 1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb -C

# 인스턴스 확인
nvidia-smi mig -lgi
```

K8s에서는 Device Plugin이 MIG 인스턴스를 별도 리소스로 노출한다.

```yaml
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1
```

MIG 프로파일은 변경 시 GPU 리셋이 필요하다. 운영 중인 클러스터에서 프로파일을 바꾸려면 해당 노드를 drain하고 작업해야 한다.

---

## 6. 멀티 GPU 서빙

### 텐서 병렬 (Tensor Parallelism)

모델의 가중치 행렬을 GPU 여러 개에 쪼개 각 GPU가 일부분을 계산한다. LLM의 Attention과 FFN 레이어가 주 대상이다.

vLLM에서 텐서 병렬은 파라미터 하나로 설정한다.

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-70b-hf \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.85
```

4개의 GPU를 하나처럼 사용해 70B 모델을 올린다. 단일 GPU로는 불가능한 크기다.

### 파이프라인 병렬 (Pipeline Parallelism)

모델의 레이어를 GPU에 순서대로 배분한다. GPU 1이 1~12 레이어, GPU 2가 13~24 레이어를 담당하는 방식이다.

텐서 병렬보다 GPU 간 통신량이 적다. 그러나 파이프라인 버블(bubble)로 인해 GPU 활용률이 낮아지는 구간이 생긴다. 미니배치를 나눠 파이프라인을 채우는 마이크로배칭(micro-batching)으로 버블을 줄인다.

DeepSpeed를 이용한 파이프라인 병렬 설정 예시다.

```python
import deepspeed

engine, _, _, _ = deepspeed.initialize(
    model=model,
    config={
        "train_batch_size": 32,
        "pipeline": {
            "stages": "auto",
            "partition": "type:transformer"
        },
        "fp16": {"enabled": True}
    }
)
```

### GPU 간 통신 최적화

NVLink가 있는 서버에서는 GPU 간 통신이 PCIe 대비 5~10배 빠르다. 인스턴스 선택 시 NVLink 지원 여부를 확인해야 한다.

```bash
# NVLink 연결 확인
nvidia-smi topo -m
```

출력에서 `NV1`, `NV2` 등의 표시가 있으면 NVLink로 연결된 GPU 쌍이다. `PIX`는 PCIe 연결을 의미한다.

NCCL은 분산 GPU 통신 라이브러리다. 환경 변수로 통신 최적화를 제어한다.

```bash
export NCCL_IB_DISABLE=0       # InfiniBand 활성화
export NCCL_NET_GDR_LEVEL=5    # GPU Direct RDMA 최대 레벨
export NCCL_TREE_THRESHOLD=0   # Ring All-reduce 사용
```

A100 클러스터에서 InfiniBand와 GPU Direct RDMA를 함께 쓰면 All-reduce 대역폭이 PCIe 대비 4배 이상 나온다.

---

## 참고 자료

- [NVIDIA TensorRT Documentation](https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/) — TensorRT 최적화 원리와 API 레퍼런스
- [vLLM Documentation](https://docs.vllm.ai/) — PagedAttention, 텐서 병렬, 서빙 설정
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) — GPU 메트릭 수집과 Prometheus 연동
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/) — Device Plugin, 리소스 요청 공식 가이드
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) — A100/H100 MIG 분할 구성 방법
- [KEDA Documentation](https://keda.sh/docs/) — 이벤트 기반 오토스케일링 설정 레퍼런스
