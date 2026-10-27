---
title: "282M 코드 LLM을 밑바닥부터: V100S 한 장으로 69억 토큰을 완주하기까지"
date: 2026-10-23 10:00:00 +0900
categories: [AI, LLM]
tags: [pytorch, transformer, rope, gqa, swiglu, fp16, pretraining]
math: true
image:
  path: /assets/img/posts/sangyong-llm/training-curve-light.png
  alt: 학습·검증 손실 곡선
---

파이썬 코드를 생성하는 282M 파라미터 트랜스포머를 직접 구현해서 Tesla V100S 한 장으로 6,879,527,547 토큰을 1에포크 학습시켰다.

| 항목 | 값 |
|---|---|
| 파라미터 | 282,641,408 |
| 학습 토큰 | 6,879,527,547 (1에포크, 13,121스텝) |
| 최저 검증 손실 | 0.8190 (iter 13,000), 퍼플렉시티 2.268 |
| 처리량 | 6,513 tok/s (80.9초/스텝) |
| 피크 VRAM | 25.01 GB / 32 GB |
| 벽시계 | 15.1일 (실제 계산 12.5일) |
| 코드 / 테스트 | 5,119줄 / 3,481줄, 162항목 통과 |

{% include embed/video.html src='/assets/img/posts/sangyong-llm/explainer.mp4' poster='/assets/img/posts/sangyong-llm/explainer-poster.jpg' title='캐시가 있는 상태에서 질의 여러 개를 넣으면 is_causal=True가 마스크를 왼쪽 위로 정렬해 에러 없이 틀린 값을 낸다. 그래서 이 경로를 막았다' loop=true %}

## 1. 무엇을 직접 만들고 무엇을 빌렸나

연산 계층에서는 PyTorch의 텐서 연산, autograd, CUDA 런타임, 융합 어텐션 커널 `F.scaled_dot_product_attention` 네 가지를 빌렸고, 옵티마이저 `torch.optim.AdamW`와 `GradScaler`도 가져다 썼다.

`transformers`, `tokenizers`, `datasets`, `accelerate`, `peft`는 쓰지 않았으니 아래 목록은 전부 직접 짰다.

- 바이트 단위 BPE 토크나이저 (학습, 인코딩, 디코딩, 직렬화)
- RoPE, RMSNorm, GQA 어텐션, SwiGLU, KV 캐시 증분 생성
- 데이터 필터, 중복 제거, 병렬 토큰화
- 학습 루프 (기울기 누적, 체크포인트, 재개)
- 정밀도 선택, 평가 채점기, 재시작 워치독

어텐션 커널을 빌린 건 범위 때문이다. 수식은 몇 줄이지만 메모리 효율적인 융합 커널을 직접 쓰는 건 별개의 프로젝트이고, 이번 목표는 모델과 학습 파이프라인이었다.

## 2. 모델 크기는 계산으로 정했다

| | 282m | 53m |
|---|---|---|
| d_model | 1024 | 640 |
| layers | 24 | 10 |
| heads | 16 (KV 4, GQA 4:1) | 10 (KV 2, GQA 5:1) |
| d_ff | 2752 (SwiGLU) | 1728 (SwiGLU) |
| context | 2048 | 1024 |
| 파라미터 | 282,641,408 | 53,507,200 |
| 기준 HW | V100S 32GB | RTX 4050 Laptop 6GB |

개발 환경이 둘이라 프리셋도 둘이다. 53m은 노트북 RTX 4050 6GB의 VRAM 상한에서, 282m은 서버 데이터 양에서 역산했다. Chinchilla 법칙은 대략 파라미터 1개당 토큰 20개를 계산 최적으로 본다. 설계 당시 기준인 6.6B 토큰이면 약 330M이 최적점이고, 같은 데이터를 53M에 쓰면 6배 과잉이라 수익이 크게 줄어든다. 그 근처에서 고른 게 282M이고, 최종 토큰화 결과는 6.88B였다.

![모델 구조도](/assets/img/posts/sangyong-llm/architecture-light.png){: .light }
![모델 구조도](/assets/img/posts/sangyong-llm/architecture-dark.png){: .dark }
_디코더 전용, Llama 계열 구성. pre-norm 블록 24개_

### 282.6M은 어디에 있나

![파라미터 구성](/assets/img/posts/sangyong-llm/params-light.png){: .light }
![파라미터 구성](/assets/img/posts/sangyong-llm/params-dark.png){: .dark }
_전체 파라미터의 71.8%가 FFN이다_

파라미터라고 하면 어텐션을 먼저 떠올리기 쉬운데 무거운 쪽은 FFN이다. 레이어 하나만 봐도 FFN 8.45M, 어텐션 2.62M으로 76% 대 24%다.

- 어텐션이 작은 건 GQA 때문이다. K, V를 4헤드만 두니 두 행렬이 1024×1024에서 1024×256으로 줄어든다. MHA였다면 레이어마다 1.57M, 전체로 37.7M이 더 붙는다.
- 임베딩이 5.9%뿐인 건 가중치를 공유해서다. `lm_head`가 임베딩과 같은 텐서를 쓰는데, 묶지 않았다면 16.8M이 더 붙었을 것이다.
- RMSNorm은 50,176개로 전체의 0.02%다.

이 숫자는 `model.Transformer`를 실제로 만들어 센 값이다. 공유 텐서를 두 번 세지 않도록 `id(p)`로 중복을 거른다.

```python
def num_params(self, non_embedding: bool = False) -> int:
    seen = set()
    total = 0
    for p in self.parameters():
        if id(p) in seen:
            continue
        seen.add(id(p))
        total += p.numel()
    if non_embedding:
        total -= self.tok_emb.weight.numel()
    return total
```

## 3. 블록 하나 뜯어보기

블록은 pre-norm 구조로, 정규화를 잔차 안쪽에 둬서 잔차 경로 `x = x + ...`가 항등 함수로 남는다. 덕분에 24층을 쌓아도 초기 신호가 감쇠하지 않는다.

```python
class Block(nn.Module):
    def forward(self, x, cos, sin, cache=None):
        h, new_cache = self.attn(self.attn_norm(x), cos, sin, cache)
        x = x + h
        x = x + self.ffn(self.ffn_norm(x))
        return x, new_cache
```

### RMSNorm: 제곱합은 float32에서

LayerNorm에서 평균 빼기를 없앤 버전이다.

$$
\mathrm{RMSNorm}(x) = \frac{x}{\sqrt{\frac{1}{d}\sum_i x_i^2 + \epsilon}} \odot g
$$

```python
class RMSNorm(nn.Module):
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # 정규화는 float32에서 한다. bf16으로 하면 제곱합에서 정밀도가 무너진다.
        dtype = x.dtype
        x = x.float()
        x = x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)
        return (x.to(dtype)) * self.weight
```

입력을 반정밀도로 받아도 제곱합이 무너지지 않도록 이 함수 안에서만 float32로 올렸다가 되돌린다. `model.to(torch.bfloat16)`처럼 모델 전체를 반정밀도로 돌리는 경우에 대비한 방어다. 지금 저장소의 학습과 추론 경로에서는 임베딩 출력이 fp32이고 `x = x + h`도 fp32로 승격되니, 잔차 스트림이 이미 fp32라서 달라지는 건 없다.

### RoPE: 학습할 위치 파라미터가 없다

위치 $m$의 벡터에서 앞 절반의 $i$번째와 뒤 절반의 $i$번째 차원을 한 쌍으로 묶어 각도 $m\theta_i$만큼 회전시킨다.

$$
\theta_i = 10000^{-2i/d_{head}}, \qquad
\begin{pmatrix} x'_{1} \\ x'_{2} \end{pmatrix} =
\begin{pmatrix} \cos m\theta_i & -\sin m\theta_i \\ \sin m\theta_i & \cos m\theta_i \end{pmatrix}
\begin{pmatrix} x_{1} \\ x_{2} \end{pmatrix}
$$

q와 k를 이렇게 돌리면 내적 $q_m \cdot k_n$에는 두 각도의 차 $(m-n)\theta_i$만 남는다. 그래서 어텐션 점수에는 절대 위치 대신 **상대 거리**가 들어간다.

```python
def build_rope_cache(seq_len, head_dim, theta, device, dtype=torch.float32):
    inv_freq = 1.0 / (
        theta ** (torch.arange(0, head_dim, 2, device=device).float() / head_dim)
    )
    pos = torch.arange(seq_len, device=device).float()
    freqs = torch.outer(pos, inv_freq)  # (seq_len, head_dim/2)
    return torch.cos(freqs).to(dtype), torch.sin(freqs).to(dtype)


def apply_rope(x, cos, sin):
    # 앞 절반과 뒤 절반을 복소수의 실수부/허수부처럼 보고 회전시킨다.
    x1, x2 = x.chunk(2, dim=-1)
    cos = cos[None, None, :, :].to(x.dtype)
    sin = sin[None, None, :, :].to(x.dtype)
    return torch.cat([x1 * cos - x2 * sin, x1 * sin + x2 * cos], dim=-1)
```

cos/sin 표는 `max_seq_len`만큼 한 번 계산해서 `register_buffer(..., persistent=False)`로 둔다. 체크포인트에 저장되지 않으니 파일 크기에 안 잡히고 로드할 때 다시 만든다.

테스트로는 두 성질을 고정했다. 회전이니 벡터 길이가 변하지 않아야 하고, 같은 거리만큼 떨어진 두 쌍은 절대 위치가 달라도 내적이 같아야 한다.

### GQA: KV 캐시를 4분의 1로

Q는 16헤드, K와 V는 4헤드라서 Q 헤드 4개가 KV 헤드 1개를 공유한다.

```python
self.wq = nn.Linear(cfg.d_model, cfg.n_heads * self.head_dim, bias=False)     # 1024 -> 1024
self.wk = nn.Linear(cfg.d_model, cfg.n_kv_heads * self.head_dim, bias=False)  # 1024 -> 256
self.wv = nn.Linear(cfg.d_model, cfg.n_kv_heads * self.head_dim, bias=False)  # 1024 -> 256
```

생성할 때 쌓이는 KV 캐시 크기를 직접 계산해 보면 효과가 보인다.

$$
\underbrace{2}_{K,V} \times \underbrace{24}_{layers} \times \underbrace{4}_{kv\ heads} \times \underbrace{64}_{head\_dim} \times \underbrace{2048}_{tokens} \times \underbrace{2\,\mathrm{B}}_{fp16} = 48\,\mathrm{MiB}
$$

MHA였다면 KV 헤드가 16개니 192MiB가 된다. 어텐션을 호출할 때는 KV 헤드를 Q 헤드 수만큼 직접 복제하지 않고 `enable_gqa=True`로 그대로 넘긴다.

```python
out = F.scaled_dot_product_attention(
    q, k, v, is_causal=(T > 1), enable_gqa=(self.n_kv_heads != self.n_heads)
)
```

### SwiGLU: d_ff = 2752의 출처

```python
def forward(self, x):
    return self.w_down(F.silu(self.w_gate(x)) * self.w_up(x))
```

게이트가 붙으면서 행렬이 2개에서 3개로 늘어나니, 보통 FFN의 `4 × d_model`을 그대로 쓰면 파라미터가 1.5배가 된다. 그래서 2/3를 곱해 행렬 3개의 총량을 행렬 2개일 때와 맞추고 64의 배수로 올렸다.

$$
4096 \times \tfrac{2}{3} \approx 2731 \;\rightarrow\; 2752 = 64 \times 43
$$

53m 프리셋의 1,728도 같은 규칙이다 ($2560 \times 2/3 \approx 1707 \rightarrow 1728$).

### 초기화: 잔차로 나가는 투영만 줄인다

```python
for name, p in self.named_parameters():
    if name.endswith("wo.weight") or name.endswith("w_down.weight"):
        nn.init.normal_(p, mean=0.0, std=0.02 / math.sqrt(2 * cfg.n_layers))
```

블록마다 잔차에 값을 두 번(어텐션 출력 `wo`, FFN 출력 `w_down`) 더하니 24층이면 48번이다. 독립적인 값을 더하면 분산이 쌓이므로 이 두 투영의 표준편차를 $1/\sqrt{2L}$만큼 줄여 합의 분산을 층 수와 무관하게 맞췄다.

## 4. KV 캐시: 조용히 틀리느니 죽는다

생성은 프롬프트 전체를 한 번에 넣어 캐시를 채우는 **프리필**과, 그 뒤 한 토큰씩 넣는 **디코드** 두 단계로 나뉜다.

```python
if past_k is not None:
    # 캐시가 있는데 질의가 여러 개면 인과 마스크 정렬이 어긋난다.
    if T > 1:
        raise ValueError(
            f"KV 캐시가 있는 상태에서 T={T} 질의는 지원하지 않는다. "
            "프리필은 캐시 없이 한 번에, 이후는 T=1로 진행할 것."
        )
    k = torch.cat([past_k, k], dim=2)
    v = torch.cat([past_v, v], dim=2)
```

캐시에 과거 100개가 있고 새 질의가 3개면 마스크는 3×103이다. 질의 $i$는 키 $0..100+i$를 봐야 하니 하삼각이 오른쪽 아래로 정렬돼야 한다. 그런데 `is_causal=True`는 질의와 키 길이가 다르면 왼쪽 위로 정렬한 하삼각을 쓴다. 그러면 새 질의 $i$가 과거 키 $0..i$만 보게 되고, 에러 하나 없이 **틀린 값**이 나온다. 어차피 쓰지 않는 경로라 지원하지 않고, 대신 들어오면 바로 죽게 했다.

디코드 단계는 `T == 1`이고 새 토큰 하나가 캐시된 과거를 전부 봐야 하니 마스크를 끈다. 테스트에서는 "캐시로 한 토큰씩 생성한 결과"와 "통짜 forward 결과"가 같은지 비교한다.

## 5. V100에서 bf16은 함정이다

`torch.cuda.is_bf16_supported()`는 V100에서도 `True`를 반환하므로 이 값만 믿으면 bf16을 고르게 된다.

![정밀도별 실측 TFLOPS](/assets/img/posts/sangyong-llm/precision-light.png){: .light }
![정밀도별 실측 TFLOPS](/assets/img/posts/sangyong-llm/precision-dark.png){: .dark }
_V100S, 4096×4096 행렬곱 실측_

| 정밀도 | TFLOPS | fp32 대비 |
|---|---|---|
| fp32 | 13.2 | 1.0배 |
| fp16 | 88.8 | 6.7배 (텐서코어) |
| bf16 | 10.0 | 0.76배 (에뮬레이션) |

V100은 sm_70인데 bf16 텐서코어는 Ampere(sm_80)부터 들어왔다. sm_70에서 bf16은 소프트웨어 경로로 돌아서 fp32보다도 느리다. "지원한다"는 말을 믿었다면 환경 검사는 통과하고 학습만 fp16의 8.9분의 1 속도로 돌았을 것이다.

그래서 판정 근거를 compute capability로 바꿨다. 아래 스니펫은 CPU 분기와 명시 지정 분기를 빼고 근거 문구도 줄인 것이다.

```python
# bf16 텐서코어가 들어온 세대. 이 미만은 에뮬레이션이라 쓰면 안 된다.
BF16_TENSOR_CORE_MAJOR = 8

def resolve(device: str = "cuda", override: str = "auto"):
    major, minor = torch.cuda.get_device_capability()
    name = torch.cuda.get_device_name()
    # override(fp32/bf16/fp16) 명시 지정 분기 생략
    if major >= BF16_TENSOR_CORE_MAJOR:
        return torch.bfloat16, f"{name} sm_{major}{minor}, bf16 텐서코어 있음"
    return torch.float16, f"{name} sm_{major}{minor}, bf16 텐서코어 없음. fp16을 쓴다"
```

환경 검증 스크립트 `verify_env.py`도 같은 원칙을 따라, 지원 여부 대신 실측 TFLOPS가 fp32보다 빠른지로 판정한다.

## 6. fp16을 고르면 클리핑 순서가 걸린다

fp16은 지수부가 5비트라 작은 기울기가 0으로 언더플로한다. 그래서 `GradScaler`가 손실에 큰 수(scale)를 곱해 역전파하고 옵티마이저 스텝 직전에 다시 나눈다. 지수부가 fp32와 같은 bf16에는 스케일러가 필요 없다.

여기서 걸리는 게 기울기 클리핑이다.

```python
def clip_and_step(scaler, optimizer, model, grad_clip: float):
    scaler.unscale_(optimizer)                       # 1. 먼저 scale을 나눈다
    gnorm = torch.nn.utils.clip_grad_norm_(model.parameters(), grad_clip)  # 2. 그다음 자른다
    scale_before = scaler.get_scale()
    scaler.step(optimizer)
    scaler.update()
    stepped = not scaler.is_enabled() or scaler.get_scale() >= scale_before
    return gnorm, stepped
```

1과 2의 순서가 바뀌면 이렇게 된다.

1. 기울기가 아직 scale배 부풀려진 상태라 노름도 $\text{scale} \cdot \lVert g \rVert$다.
2. 그걸 1.0으로 자른다. 계수 $\text{clip} / (\text{scale} \cdot \lVert g \rVert)$가 곱해진다.
3. 그 뒤 `step`이 내부에서 한 번 더 scale로 나눈다. scale·‖g‖가 clip보다 크면(거의 항상) 최종 노름은 실제 기울기와 무관하게 $\text{clip}/\text{scale}$로 고정된다.

실측(scale 128, grad_clip 1.0)에서 실제 기울기 노름이 0.25에서 0.0078(= 1/128)로 **32배** 줄었다. AdamW가 $m/\sqrt{v}$로 크기를 어느 정도 상쇄하긴 해도 스텝마다 다른 계수로 잘리니 기울기 이력이 왜곡된다. 그런데도 손실은 떨어져서 곡선만 봐서는 알 수 없다. 로그에 찍히는 gnorm은 오히려 128배 부풀어 보여서 "기울기가 큰 구간"으로 잘못 읽힌다. 이 순서는 `test_training.py`에 테스트로 고정해 뒀다.

`stepped`로는 하나를 더 잡는다. 기울기에 inf/nan이 섞이면 스케일러는 그 스텝을 통째로 버리고 scale을 낮추는데, 버린 횟수를 세지 않으면 "돌긴 도는데 진도가 안 나가는" 상태를 알아챌 수 없다. 이번 런에서 버려진 스텝은 13,121스텝 중 0회였다.

### 기울기 누적

배치는 2048토큰 시퀀스 2개이고 이때 피크 VRAM이 25.01GB였다. 이걸 128번 누적해 스텝당 524,288 토큰을 만든다.

```python
for _ in range(tcfg.grad_accum):
    x, y = train_ds.batch(tcfg.batch_size, device)
    with autocast(amp_dtype):
        _, loss, _ = model(x, targets=y)
    # 누적 스텝 수로 나눠야 전체 배치 평균과 같아진다
    scaler.scale(loss / tcfg.grad_accum).backward()
```

`loss / grad_accum`을 빼먹으면 기울기가 128배가 되어 클리핑이 매 스텝 걸리고 누적 등가도 깨진다. 테스트에서는 "배치 2로 4번 누적"과 "배치 8 한 번"의 기울기가 같은지 비교한다.

## 7. 학습 설정과 곡선

| 항목 | 값 | 근거 |
|---|---|---|
| 정밀도 | fp16 + GradScaler | sm_70, bf16 텐서코어 없음 |
| 옵티마이저 | AdamW (β 0.9, 0.95) | |
| weight decay | 0.1 | 노름 가중치·바이어스는 제외 |
| 학습률 | 6e-4 | 500스텝 워밍업 후 코사인으로 6e-5 |
| 기울기 클리핑 | 1.0 | `unscale_()` 뒤 |
| 배치 | 2 × 128 × 2048 = 524,288 토큰 | 32GB에서 역산 |
| 스텝 | 13,121 | `train.bin` 크기에서 자동 계산 |

![학습률 스케줄과 기울기 노름](/assets/img/posts/sangyong-llm/lr-gnorm-light.png){: .light }
![학습률 스케줄과 기울기 노름](/assets/img/posts/sangyong-llm/lr-gnorm-dark.png){: .dark }
_워밍업이 끝나기 직전인 iter 480에서 기울기 노름이 292.6까지 튄다_

워밍업(500스텝)이 끝나기 직전, 학습률이 최고점의 96%에 닿은 iter 480에서 기울기 노름이 292.6, 손실이 5.74로 함께 튀었다. 클리핑 1.0이 이걸 잘랐고, 1,500스텝 안에 0.13 근방으로 돌아온 뒤로는 끝까지 평평했다.

![학습·검증 손실 곡선](/assets/img/posts/sangyong-llm/training-curve-light.png){: .light }
![학습·검증 손실 곡선](/assets/img/posts/sangyong-llm/training-curve-dark.png){: .dark }
_검증 손실은 끝까지 내려가는 중이었다_

검증 손실은 추세상 13,000스텝까지 계속 내려갔고 과적합으로 꺾이지도 않았다. 학습이 끝난 건 수렴해서가 아니라 데이터를 다 써서였다.

iter 4,000과 11,000 근처에서 되튀는 건 성능이 변해서가 아니다. 검증 손실을 잴 때 시드 없이 매번 다른 창 100개(전체의 1.5%)를 뽑아서 생긴 잡음이다. 실측 표준편차가 0.034, 최소~최대 폭이 0.097이니 되튐 대부분은 이걸로 설명된다. 이 버그는 아직 안 고쳤고 SFT 전에 고정 시드로 바꿔야 한다.

## 8. 학습은 두 번 죽었다

![벽시계 대비 진행 스텝](/assets/img/posts/sangyong-llm/timeline-light.png){: .light }
![벽시계 대비 진행 스텝](/assets/img/posts/sangyong-llm/timeline-dark.png){: .dark }
_평평한 구간이 죽어 있던 시간이다_

| | 죽은 시각 | 죽은 지점 | 재개 지점 | 멈춰 있던 시간 |
|---|---|---|---|---|
| 1차 | 09-05 22:57 | iter 5,670 | 5,501 | 23.4시간 |
| 2차 | 09-12 20:45 | iter 11,860 | 11,751 | 38.8시간 |

1차 때는 트레이스백도 OOM도 재부팅 흔적도 없었다. 공용 서버라 `/var/log` 권한이 없어서 원인은 밝히지 못했고, 원인을 모르니 재발한다고 가정하고 워치독을 만들었다.

워치독은 5분마다 상태를 보고, 프로세스가 없거나 로그가 45분째 안 늘면 `--resume`으로 다시 띄운다. 다만 다음 세 경우에는 다시 띄우지 않는다.

- 로그에 정상 종료 문구 `학습 종료`가 있을 때
- 15분 안에 죽기를 3회 반복할 때. 설정 문제로 보고 손을 뗀다
- `stop` 요청이 있을 때. 사람이 일부러 멈춘 학습을 살리면 안 된다

### 좀비 프로세스 함정

워치독은 학습을 자식 프로세스로 띄우고 `wait()`하지 않는데, `os.kill(pid, 0)`은 **좀비에도 성공한다.** 죽은 학습은 거둬지기 전까지 좀비로 남아 있으니 PID만 확인하면 영원히 "실행 중"으로 보이고, 워치독은 아무것도 하지 않는다.

```python
def _alive(pid: int) -> bool:
    try:
        os.kill(pid, 0)          # 신호 0: 존재 여부만 확인
    except ProcessLookupError:
        return False
    except PermissionError:
        return True              # 남의 프로세스지만 살아 있다
    # 좀비는 신호 0에 응답한다. 상태 문자까지 봐야 죽은 걸 죽었다고 판정한다.
    try:
        stat = Path(f"/proc/{pid}/stat").read_text()
        return stat.rsplit(") ", 1)[1].split(" ", 1)[0] != "Z"
    except (OSError, IndexError):
        return True
```

`/proc/<pid>/stat`의 두 번째 필드는 괄호로 감싼 프로세스 이름인데, 이름에 공백이나 괄호가 들어갈 수 있다. 그래서 앞에서부터 자르지 않고 마지막 `") "` 뒤를 읽는다. 거기서 첫 토큰이 상태 문자이고, 이게 `Z`면 좀비다.

### 2차: 감시자도 같이 죽었다

2차 때는 이 워치독도 소용이 없었다. 학습이 20:45에 죽고 2분 뒤인 20:47에 워치독 자신도 SIGTERM을 받아 "중단 요청"으로 정상 종료했다. 서버는 199일째 가동 중이었고 메모리도 여유가 있었으니, 외부에서 사용자 프로세스를 일괄 종료한 것으로 본다.

학습과 감시자가 같은 신호에 함께 쓸려 나가면 감시자를 하나 더 둔다고 막을 수 있는 게 아니다. 이번 런에서는 그렇게 38.8시간을 잃었다.

체크포인트를 250스텝마다 쓰니(5,500과 11,750 체크포인트에서 재개) 두 번 합쳐 278스텝을 다시 계산했다. 체크포인트는 `.tmp`에 먼저 쓰고 `replace()`로 바꿔 끼우기 때문에 저장 도중 죽어도 이전 파일은 멀쩡하다.

## 9. 손실 곡선으로는 못 잡는 버그들

테스트는 "동작하는가"보다 "어떻게 깨지는가"를 찾으려고 짰다. 10개 스위트, 162항목이다.

그중 가장 중요한 건 **인과 마스크 누설**이다. 마스크가 뚫리면 모델이 정답(다음 토큰)을 보고 맞히니 손실은 아주 예쁘게 떨어지는데 생성만 안 된다. 곡선으로는 절대 알 수 없다.

```python
def c_causal_no_future_leak():
    """미래 토큰을 바꿔도 과거 위치의 출력은 변하면 안 된다."""
    m = _model()
    T = 32
    idx = torch.randint(0, TEST_CFG.vocab_size, (2, T), device=DEVICE)
    with torch.no_grad():
        base, _, _ = m(idx)

    worst = 0.0
    for cut in (8, 16, 24):
        tampered = idx.clone()
        tampered[:, cut:] = (tampered[:, cut:] + 137) % TEST_CFG.vocab_size
        with torch.no_grad():
            other, _, _ = m(tampered)
        diff = (base[:, :cut, :] - other[:, :cut, :]).abs().max().item()
        worst = max(worst, diff)

    assert worst < 1e-5, f"미래 정보가 과거로 샌다. 최대 차이={worst:.3e}"
```

`cut` 이후 토큰을 전부 바꾸고 `cut` 이전 위치의 로짓이 그대로인지 본다. 마스크를 실수로 끄면 여기서 걸린다.

그 밖에 고정한 것들은 이렇다.

| 스위트 | 잡는 것 |
|---|---|
| `test_model.py` | RoPE 상대 위치, KV 캐시 == 통짜 forward(프리필 뒤 `start_pos`가 붙는 경로라 RoPE 슬라이스 오류도 여기서 걸린다), 프리셋 크기 고정 |
| `test_training.py` | 기울기 누적 등가, 단일 배치 과적합, 재개 전후 손실 차이 0.00e+00, fp16 클리핑 순서 |
| `test_tokenizer.py` | 적대적 입력 28종, 무작위 유니코드 1,000건 왕복 |
| `test_eval.py` | 생성 코드가 `sys.exit(0)`으로 테스트를 건너뛰는 우회 차단 |
| `test_watchdog.py` | 좀비 판정, 조기 사망 차단, 재시작 인자 일치 |

`sys.exit(0)` 우회는 실제로 생긴다. 생성된 코드가 종료 코드 0으로 나가 버리면 단언문이 실행되지 않아도 "통과"로 잡힌다. 그래서 채점기는 테스트 코드 뒤에 `print('__HARNESS_TESTS_COMPLETED__')`를 붙여 실행하고, 종료 코드가 0이어도 이 문자열이 없으면 실패로 본다.

## 10. 평가: 이 pass@5를 믿지 말 것

문제 10개를 각각 5번 생성해 별도 프로세스에서 실행하고 채점한다.

![문제 10개 × 5회 생성 결과](/assets/img/posts/sangyong-llm/eval-grid-light.png){: .light }
![문제 10개 × 5회 생성 결과](/assets/img/posts/sangyong-llm/eval-grid-dark.png){: .dark }
_칸 하나가 생성 한 번. 통과는 50칸 중 13칸_

아래 수치는 `best.pt`(iter 13,000)를 **한국어 독스트링 프롬프트**로 잰 값이다.

| 지표 | 값 |
|---|---|
| 문법 유효율 | 96.0% (48/50) |
| pass@5 | 60.0% (6/10) |
| 샘플 단위 통과율 | 26.0% (13/50) |

50칸 중 13칸만 통과했는데 pass@5가 60%인 건 5번 중 한 번만 맞아도 그 문제를 통과로 세기 때문이다. `max_of_list`도 5번 중 4번이 실행 오류였는데 통과로 잡혔다. pass@5가 재는 건 "얼마나 자주 맞히나"가 아니라 "몇 번 시키면 한 번은 맞히나"다.

표본이 너무 작다는 문제도 있다.

| 체크포인트 | 스텝 | pass@5 | 샘플 단위 |
|---|---|---|---|
| `best.pt` | 13,000 | 60.0% | 26.0% |
| `latest.pt` | 13,120 | 50.0% | 28.0% |

120스텝 차이로 pass@5가 10%p 벌어지는데, 문제가 10개라 한 문제가 곧 10%p다. 샘플 단위는 26% 대 28%로 거의 같고, 실제 차이는 이쪽에 가깝다.

### 프롬프트 언어가 측정을 왜곡했다

처음엔 한국어 독스트링으로 평가했는데, 토크나이저가 파이썬 코드로 학습된 탓에 한글을 거의 병합하지 않았다.

| 독스트링 | 토큰 |
|---|---|
| `두 수를 더해 반환한다.` | 31 |
| `Return the sum of two numbers.` | 7 |

같은 뜻에 토큰이 4~7배 들고, 학습 분포에 없는 문자열이라 이어 쓸 단서도 없다. 같은 체크포인트(`latest.pt`, iter 13,120)를 프롬프트만 영어로 바꿔 다시 쟀더니 pass@5가 40%에서 60%, 샘플 단위가 24%에서 36%로 올랐다. pass@5 숫자 자체는 잡음이 크지만 토큰 비용 차이는 잡음 없이 확인되는 근거여서 평가 프롬프트를 영어로 바꿨다.

### 실제 생성

```python
def flatten(nested):
    """Flatten a nested list into a single flat list."""
    return [item for sublist in nested for item in sublist]


def flatten_entities(entities):
    """Flatten a list of entities into a single flat list."""
    return [entity for entity in entities if entity]
```

첫 함수는 맞는데, 그 뒤로 비슷한 이름의 함수를 멈추지 않고 계속 찍어낸다. 프리트레이닝만 한 기반 모델이라 "요청 하나를 끝내고 멈춘다"를 배운 적이 없어서다. 다음 단계인 SFT에서 가르칠 게 바로 이 부분이다.

## 11. 한계와 다음 단계

- 1에포크로 끝났다. 검증 손실은 아직 내려가는 중이었다.
- 의미 정확도가 낮다. 문법은 96%인데 샘플 단위 통과율은 26%다. 파이썬의 형태는 배웠어도 무엇을 계산하라는 건지는 절반쯤만 따라온다.
- GPU는 한 장만 썼다. 서버의 V100S 10장 중 한 장만 쓰기로 한 공용 장비라 12.5일이 걸렸다.
- 검증 배치 시드를 안 박았다. `best.pt`가 실제로 최선이라는 보장이 없으니 쓸 모델은 `latest.pt`다.
- SFT는 아직이다. 데이터셋 2,845/155쌍과 학습 루프는 준비해 뒀고, CPU 초소형 모델로만 검증했다.

이 프로젝트에서 가장 많이 배운 건 모델 구조보다 조용히 틀리는 곳들이었다. bf16 지원 여부, 클리핑 순서, 누적 손실 나누기, 좀비 PID, 마스크 누설, `sys.exit(0)`까지 전부 에러 없이 돌아가면서 결과만 틀린다. 그래서 손실 곡선을 믿는 대신 테스트로 하나씩 고정했다.

코드: [github.com/kasangyong/sangyong_llm](https://github.com/kasangyong/sangyong_llm)
