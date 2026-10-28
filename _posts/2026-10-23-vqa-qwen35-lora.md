---
title: "사진을 보고 분리배출을 답하는 AI: 작은 모델을 고른 이유"
date: 2026-10-23 11:00:00 +0900
categories: [AI, Multimodal]
tags: [vqa, qwen, vlm, lora, qlora, unsloth, bf16, tta]
image:
  path: /assets/img/posts/vqa-qwen35-lora/thumb-poster.jpg
  alt: 재활용품 사진 한 장을 보고 4지선다의 정답 글자 하나를 고르는 VQA 소개 애니메이션
---

재활용품 사진과 4지선다 질문을 받아 정답 글자 하나를 내는 VQA 모델을 Qwen3.5-4B에 LoRA를 붙여 만들었다. SSAFY AI 챌린지에 4인 팀으로 참가했다(포트폴리오 기록, 저장소 미기재).

| 항목 | 값 |
|---|---|
| 모델 | `unsloth/Qwen3.5-4B`, 양자화 없이 BF16 + LoRA r 32 |
| 학습 파라미터 | 77,512,704 / 4,616,778,240 (1.68%, `best.ipynb` 출력 기준, 같은 4B r 32 설정) |
| 학습 데이터 | train.csv 5,073개 중 90% (4,565개) |
| val_loss | ~0.02 (에폭 1) |
| 제출 정확도 | 0.973 |
| 비교 실험 | 9B 4-bit QLoRA val_loss 0.1387 (조건 상이, 정확도 미제출) |
| GPU | RTX 5060 Ti 1장, 17.1 GB |
| 순위 | 193팀 중 3위 (대회 결과, 저장소 미기재) |

저장소에는 개인별 분담 기록이 없어서 아래 실험은 팀 단위로 적는다.

{% include embed/video.html src='/assets/img/posts/vqa-qwen35-lora/intro.mp4' poster='/assets/img/posts/vqa-qwen35-lora/intro-poster.jpg' title='재활용품 VQA 소개: 한 장의 사진, 한 글자의 답' %}

## 1. 문제와 데이터

입력은 이미지 한 장과 한국어 질문, 선지 a~d이고 출력은 소문자 한 글자다. 평가 지표는 정확도다.

| 파일 | 행 수 | 컬럼 |
|---|---|---|
| `train.csv` | 5,073 | id, path, question, a, b, c, d, answer |
| `dev.csv` | 4,413 | 위 + answer1 ~ answer5 (정답 5개) |
| `test.csv` | 5,074 | answer 없음 |

```
train_0001.jpg,train/train_0001.jpg,사진 속 흰색 텀블러의 재질은 무엇인가요?,유리,금속,종이,플라스틱,b
```

### 질문 유형은 dev에서 개수가 70%를 넘는다

질문 문장에 들어간 키워드로 유형을 나눠 봤다. "몇"이나 "개수"가 있으면 개수, "색"이 있으면 색상, "재질"·"소재"가 있으면 재질 순으로 판정하는 근사 분류다. 이 순서 탓에 "사진 속 흰색 텀블러의 재질은 무엇인가요?" 같은 재질 질문이 색상으로 잡히기도 해서 경계는 거친 편이다.

![질문 유형 분포](/assets/img/posts/vqa-qwen35-lora/question-types-light.png){: .light }
![질문 유형 분포](/assets/img/posts/vqa-qwen35-lora/question-types-dark.png){: .dark }
_train은 개수와 재질이 비슷하고, dev는 개수가 71.6%다_

train에서 개수 문제는 1,755개(34.6%)인데 dev에서는 3,159개(71.6%)다. 브랜드·라벨을 묻는 문항은 train 34개, dev 6개뿐이다. 2장의 규칙으로 dev 902개를 더하면 그중 717개가 개수 문제여서 학습 데이터의 개수 문제 비중이 34.6%에서 41.4%로 올라간다.

### dev의 정답 5개는 한 번도 전부 일치하지 않는다

dev에는 문항마다 답이 5개 달려 있는데, README는 이것을 "레이블러 5명"으로, `plan.md`는 "5개 앙상블"로 적었다. 사람이 단 답인지 모델이 낸 답인지는 저장소만으로는 확정할 수 없다.

![dev 라벨 동의도](/assets/img/posts/vqa-qwen35-lora/dev-agreement-light.png){: .light }
![dev 라벨 동의도](/assets/img/posts/vqa-qwen35-lora/dev-agreement-dark.png){: .dark }
_최다 답이 받은 표 수. 4,413문항 중 5표 전원 일치는 0건이다_

| 최다 답의 표 수 | 문항 | 비율 |
|---|---|---|
| 5 | 0 | 0% |
| 4 | 902 | 20.4% |
| 3 | 2,606 | 59.1% |
| 2 이하 | 905 | 20.5% |

빈 답 칸도 118칸 있어서 5칸이 모두 찬 문항만 따로 세면 4표 891개, 3표 2,539개, 2표 871개다. 질문 유형별 평균 최다 표 수는 개수 3.00, 재질 3.00, 색상 3.12로 비슷해서 불일치가 특정 유형에 몰려 있지는 않다.

## 2. dev를 어디까지 학습에 넣을까

규칙은 "5개 중 4개 이상 같은 답"이고, 아래는 `option_d_qwen35_4b_best.py`에 있는 구현이다.

```python
def majority_vote(row):
    votes = [str(row[c]).strip().lower() for c in ans_cols if pd.notna(row[c])]
    if not votes:
        return None
    top, count = Counter(votes).most_common(1)[0]
    return top if count >= 4 else None  # 4/5 이상 동의한 샘플만 포함
dev_df["answer"] = dev_df.apply(majority_vote, axis=1)
dev_df = dev_df[dev_df["answer"].notna()]  # 애매한 샘플(3/5 이하) 제거
```

이 조건이면 902개가 남는다. 빈 칸은 건너뛰고 세기 때문에 답이 4개만 달렸는데 4개가 모두 같은 11문항도 902개 안에 들어 있다. 3표 2,606개는 노이즈로 보고 뺐다. 3표 다수결까지 넣으면 문항 수는 3.9배가 되지만 다섯 중 둘이 반대한 답을 정답으로 가르치는 셈이 된다.

다만 0.973을 낸 모델은 이 규칙을 쓰지 않고 train.csv만으로 학습했다. 규칙을 적용한 실행(Option D)은 다른 변수까지 함께 바꿨다가 발산해서(7장) 편입 효과를 확인하지 못했다.

저장소에는 train 5,073개와 dev 3,694개를 합친 `train_combined_v8767_for_933.csv`도 있다. README에는 "외부에서 추가 합산된 버전"이라는 설명뿐인데, 그 dev 3,694개에는 최다 답이 2표인 문항 546개가 섞여 있다. 어떤 규칙으로 만들었는지 보여 주는 코드도 없어서 여기서는 쓰지 않는다.

## 3. VRAM이 모델 크기를 정했다

| 항목 | 값 |
|---|---|
| GPU | NVIDIA GeForce RTX 5060 Ti (Blackwell) |
| VRAM | 17.1 GB (= 15.93 GiB, `best.ipynb` 출력의 Unsloth 표시 "Max memory: 15.928 GB") |
| 학습 | batch 1, gradient checkpointing (`"unsloth"`) |

노트북 제목의 "16GB"와 출력의 "17.1 GB"는 같은 메모리를 단위만 달리 적었다. `total_memory / 1e9`는 10진 GB이고 Unsloth는 GiB로 찍는다.

BF16 가중치만으로 VRAM이 얼마나 드는지 계산해 보면 고를 수 있는 구성이 금방 좁혀진다. 4B는 `best.ipynb`의 `print_trainable_parameters()` 출력(같은 4B r 32 설정)에서 LoRA 파라미터를 뺀 값으로 계산했고 9B와 NF4는 저장소 주석에 있는 추정치다.

| 구성 | 가중치 크기 | 근거 |
|---|---|---|
| 4B BF16 | 4,539,265,536 × 2 B = 9.08 GB (8.45 GiB) | `best.ipynb` 출력에서 계산 |
| 9B BF16 | ~18 GB | `option_c` 주석, VRAM 초과 |
| 9B NF4 | ~4.5 GB | `option_c` 주석 |

4B는 BF16 그대로 올려도 활성값과 LoRA 옵티마이저 상태를 둘 여유가 남는다. 9B는 BF16으로 올릴 수 없어서 저장소는 4-bit를 택했다(8-bit는 시도 기록이 없다). 코드 주석에도 이 사정이 그대로 적혀 있다.

```python
# 9B BF16 가중치 = ~18 GB → VRAM 초과
# load_in_4bit=True (NF4) : 가중치 ~4.5 GB → 훈련 전체 ~10-13 GB 추정
# ※ Unsloth는 Qwen3.5에서 QLoRA 비권장(품질 손실)이나
#    17 GB VRAM에서 9B BF16은 물리적으로 불가 → 4-bit 불가피
model, processor = FastVisionModel.from_pretrained(
    model_name                 = MODEL_ID,
    load_in_4bit               = True,
    use_gradient_checkpointing = "unsloth",
    max_seq_length             = MAX_SEQ_LEN,
)
```

그러니 고민은 "4B냐 9B냐"보다 "4B를 그대로 쓰느냐, 9B를 4-bit로 깎아 쓰느냐"에 가까웠다. 14B도 4-bit로 시도했지만 끝내지 못하고 중단했다.

4B 쪽 LoRA는 비전과 언어 양쪽의 선형층 전부에 붙였다. 아래 코드는 `best.ipynb`에서 가져왔고 값은 README에 적힌 Option A와 같다(r 32, alpha 64, dropout 0.05).

```python
model = FastVisionModel.get_peft_model(
    model,
    finetune_vision_layers     = True,
    finetune_language_layers   = True,
    finetune_attention_modules = True,
    finetune_mlp_modules       = True,
    r                          = LORA_R,
    lora_alpha                 = LORA_ALPHA,
    lora_dropout               = LORA_DROPOUT,
    bias                       = "none",
    random_state               = SEED,
    target_modules             = "all-linear",
)
```

`best.ipynb` 출력으로 보면 학습되는 파라미터는 77,512,704개로 전체의 1.68%다.

## 4. 4B BF16과 9B 4-bit는 같은 조건의 비교가 아니다

![val_loss 비교](/assets/img/posts/vqa-qwen35-lora/val-loss-light.png){: .light }
![val_loss 비교](/assets/img/posts/vqa-qwen35-lora/val-loss-dark.png){: .dark }
_9B는 에폭을 더 돌려도 0.14 근처에 머물렀다. 4B 값은 README 기록이다_

숫자만 놓고 보면 4B가 7배 가까이 낮지만, 두 실행은 모델 크기와 양자화 말고도 다른 게 많다.

| 항목 | Option A (4B) | Option C (9B) |
|---|---|---|
| 양자화 | 없음 (BF16) | NF4 4-bit |
| LoRA r / alpha / dropout | 32 / 64 / 0.05 | 64 / 128 / 0 |
| 학습률 | 2e-5 | **2e-4** |
| gradient accumulation | 16 | 8 |
| 학습 해상도 상한 | 448² | 512² |
| 추론 해상도 상한 | 1,003,520 px (1280×28×28) | 1,048,576 px (1024²) |
| thinking | 템플릿 기본값 | `enable_thinking=False` |
| max_new_tokens | 5 | 4 |
| 검증 분할 | `df.sample(frac=1, random_state=42)` | `random.Random(42).shuffle(idx)` |
| val_loss | ~0.02 (에폭 1) | 0.1545 → 0.1400 → 0.1387 |
| 제출 정확도 | 0.973 | 미제출 |

검증 분할 줄을 놓치기 쉽다. 둘 다 시드 42로 90/10을 나누는데 섞는 함수가 달라서 검증에 쓰인 508개 중 47개만 겹친다. val_loss를 같은 시험지로 잰 게 아닌 셈이다.

학습률도 10배 차이 난다. 7장의 Option D는 4B BF16에 같은 2e-4를 줬다가 발산했으니, 9B의 높은 val_loss 가운데 어디까지가 4-bit 양자화 몫이고 어디까지가 학습률 몫인지 이 실험으로는 나눌 수 없다.

README는 4B BF16을 고른 근거로 세 가지를 들었다.

- Qwen3.5에 QLoRA(4-bit)가 권장되지 않는다(`option_c` 주석은 Unsloth 기준이라 적었다).
- val_loss 비교: 4B BF16 0.02, 9B 4-bit 0.14.
- 4B는 약 8GB라 BF16 그대로 VRAM에 들어간다.

여기에 해석을 하나 보태자면 9B는 에폭 2에서 최종 기록까지 0.0013밖에 줄지 않았다. 같은 설정으로 더 돌려도 4B 수준까지 내려갈 기미는 없었다.

그래서 "4-bit 양자화가 품질을 크게 떨어뜨린다"는 README의 결론은 이 데이터로 증명되지 않는다. 확인된 건 "이 조건의 9B 4-bit 실행이 이 조건의 4B BF16 실행보다 val_loss가 높았다"까지다.

## 5. 프롬프트와 학습 파이프라인

### 프롬프트

Option A의 시스템 프롬프트는 짧고 출력 형식만 못 박아 둔다.

```python
SYSTEM_INSTRUCT = (
    "당신은 시각적 질의응답을 수행하는 도우미입니다. "
    "보기 중 정확히 하나의 소문자 알파벳으로만 답하세요. 설명은 하지 마세요."
)

def build_mc_prompt(question: str, choices_ordered: list) -> str:
    # 다지선다 프롬프트 생성 (셔플된 선지 순서 적용)
    lines = [f"({ch}) {txt}" for ch, txt in choices_ordered]
    return (
        f"{question}\n"
        + "\n".join(lines)
        + "\n\n정답을 반드시 a, b, c, d 중 하나의 소문자 한 글자로만 출력하세요."
    )
```

나중에 `best.ipynb`에서는 시스템 프롬프트에 재질 구분, 색이 다른 물체는 따로 취급, 개수는 조건에 맞는 것만 세기 같은 확인 순서를 넣었다. dev에 개수 문제가 많은 것과 맞물리는 변경이지만 이 프롬프트로 낸 정확도 기록은 없다.

### 선지 셔플링

정답 글자 분포는 a 1,263, b 1,265, c 1,311, d 1,234로 고른 편이다. 그래도 모델이 "c 자리"나 특정 문구 위치를 외우지 않도록 학습 샘플을 꺼낼 때마다 선지 순서를 섞고 정답 글자도 같이 옮긴다.

```python
if self.shuffle_choices:
    indices = list(range(4))
    random.shuffle(indices)
    displayed_choices = [("abcd"[j], orig_choices[idx][1]) for j, idx in enumerate(indices)]
    orig_to_new       = {orig_choices[idx][0]: "abcd"[j] for j, idx in enumerate(indices)}
# ... 생략
if self.train:
    gold_orig = str(row["answer"]).strip().lower()
    gold_new  = orig_to_new.get(gold_orig, gold_orig)
```

같은 문항도 에폭마다 다른 순서로 보이니 답의 "내용"을 맞혀야 loss가 내려간다. 검증셋은 `shuffle_choices=False`라 원래 순서로만 잰다.

### 레이블 마스킹

loss가 assistant 턴에만 걸리도록 마지막 `<|im_start|>` 앞을 전부 -100으로 지운다.

```python
labels      = enc["input_ids"].clone()
im_start_id = self.processor.tokenizer.convert_tokens_to_ids("<|im_start|>")

for b in range(labels.shape[0]):
    seq       = enc["input_ids"][b]
    positions = (seq == im_start_id).nonzero(as_tuple=True)[0]
    # 마지막 <|im_start|> (= assistant 턴 시작) 이전을 모두 -100
    if len(positions) > 0:
        labels[b, : positions[-1].item()] = -100
    else:
        labels[b, :] = -100

pad_id = self.processor.tokenizer.pad_token_id
if pad_id is not None:
    labels[enc["input_ids"] == pad_id] = -100
```

마스킹하지 않으면 시스템 프롬프트, 질문, 선지, 수백 개의 이미지 토큰이 전부 loss에 들어가서 정답 글자 한 개의 신호가 그 평균 속에 묻힌다.

README는 이것을 "답 알파벳 토큰만 학습"이라고 적었지만 코드는 조금 다르다. 슬라이스가 `positions[-1]` 자신은 남기기 때문에 `<|im_start|>`, 역할 이름 `assistant`, 줄바꿈, 정답 글자, 그리고 턴을 닫는 토큰까지가 레이블로 남는다. 템플릿에 따라서는 빈 think 블록 토큰이 들어갈 수도 있다. 정답 글자를 뺀 나머지는 모델이 거의 확정적으로 맞히는 토큰이어서 평균 loss를 끌어내린다. 그래서 val_loss 0.02를 정답 글자 하나의 교차 엔트로피로 읽으면 실제보다 좋아 보인다. 다만 4B와 9B는 같은 마스킹을 쓰니 이 희석은 양쪽에 비슷하게 걸린다.

### 학습 루프

```python
for step, batch in enumerate(pbar, start=1):
    batch = {k: v.to(device) for k, v in batch.items() if isinstance(v, torch.Tensor)}

    with torch.amp.autocast("cuda", dtype=torch.bfloat16):
        loss = model(**batch).loss / GRAD_ACCUM

    loss.backward()
    running += loss.item()

    if step % GRAD_ACCUM == 0:
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
        scheduler.step()
```

이미지 크기가 제각각이라 batch는 1로 두고 16번 누적해 실효 배치 16을 만든다. BF16은 지수부가 FP32와 같아서 FP16과 달리 `GradScaler`를 쓸 필요가 없다. 옵티마이저는 AdamW(weight decay 0.01)이고 스케줄은 전체 스텝의 10%를 워밍업한 뒤 코사인으로 내린다.

Option A에는 누적 나머지 버그가 있었다. 4,565 = 16 × 285 + 5여서 에폭이 끝날 때 마이크로배치 5개의 기울기가 쌓인 채 남는데, 다음 에폭을 시작하는 `zero_grad`가 이걸 그냥 버린다. 스케줄러도 계획한 286스텝 중 285스텝만 밟는다. 남은 기울기를 에폭 끝에 한 번 더 적용하는 flush는 저장소에서 Option D에 처음 나오고(`option_c`에는 없다) `best.ipynb`가 이어받았다.

```python
# 에폭 끝 나머지 배치 flush
if step % GRAD_ACCUM != 0:
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
    optimizer.step()
    optimizer.zero_grad(set_to_none=True)
    scheduler.step()
```

이렇게 해야 스케줄러 스텝 수가 `ceil(len / GRAD_ACCUM)`으로 계산한 총 스텝과 맞는다. 남은 기울기도 16으로 나눈 값이라 4,565개 기준으로 나머지 5개만 쌓인 그 스텝의 기울기 크기는 꽉 찬 스텝의 약 5/16이다.

## 6. 추론: 선지 순서 5가지 다수결

{% include embed/video.html src='/assets/img/posts/vqa-qwen35-lora/explainer.mp4' poster='/assets/img/posts/vqa-qwen35-lora/explainer-poster.jpg' title='예시 문항 하나를 선지 순서 5가지로 다섯 번 묻고, 원래 글자로 되돌린 다수결로 답을 정하는 추론 경로' loop=true %}

추론 해상도 상한은 학습(448²)보다 높은 1,003,520 px로 올린다. 학습 때 VRAM을 아끼려고 낮췄던 해상도를 메모리 여유가 있는 추론에서는 되돌린다.

학습에서 순서를 섞었던 것처럼 추론에서도 같은 문항을 선지 순서만 바꿔 5번 묻는다. 각 답은 원래 글자로 되돌린 다음 다수결로 정한다.

```python
def infer_one(row, img, label_order: list) -> str:
    orig_labels = ["a", "b", "c", "d"]
    orig_texts  = [str(row["a"]), str(row["b"]), str(row["c"]), str(row["d"])]

    displayed       = [("abcd"[j], orig_texts[i]) for j, i in enumerate(label_order)]
    display_to_orig = {"abcd"[j]: orig_labels[i] for j, i in enumerate(label_order)}
    # ... 프롬프트 구성, chat template 적용 생략
    with torch.no_grad():
        out_ids = model.generate(
            **inputs,
            max_new_tokens = MAX_NEW_TOKENS,
            do_sample      = False,
            eos_token_id   = processor.tokenizer.eos_token_id,
        )
    new_ids  = out_ids[:, input_len:]
    out_text = processor.batch_decode(new_ids, skip_special_tokens=True)[0]
    pred     = extract_choice(out_text)
    return display_to_orig.get(pred, pred)


TTA_ORDERS = [
    [0, 1, 2, 3],   # 원본 순서
    [1, 2, 3, 0],   # 순환 1
    [2, 3, 0, 1],   # 순환 2
    [3, 0, 1, 2],   # 순환 3
    [3, 2, 1, 0],   # 역순 (동점 방지용)
]
# ... 생략
final = Counter(votes).most_common(1)[0][0] if votes else "a"
```

| 순서 | (a) 자리 | (b) 자리 | (c) 자리 | (d) 자리 |
|---|---|---|---|---|
| 원본 | 선지 a | 선지 b | 선지 c | 선지 d |
| 순환 1 | b | c | d | a |
| 순환 2 | c | d | a | b |
| 순환 3 | d | a | b | c |
| 역순 | d | c | b | a |

순환 4개를 쓰면 각 선지가 네 자리에 한 번씩 놓인다. 위치 편향이 있더라도 모든 선지가 똑같이 덕을 보거나 손해를 본다. 다섯 번째 역순은 동점을 막으려고 넣었다.

### 홀수라고 동점이 안 나지는 않는다

README와 코드 주석에는 "홀수(5×)라 동점 없음"이라고 적혀 있지만 선택지가 4개라 틀린 말이다. 5표는 2·2·1로도 갈린다. 이때 `Counter.most_common`은 표 수가 같으면 먼저 들어온 답을 앞에 두므로 동점이면 투표 목록에 먼저 등장한 답이 이긴다. 표는 원본 순서부터 넣으니 원본 순서의 답이 동점 후보에 있으면 원본 답이 이긴다. 정해진 규칙대로 깨지니 결과가 흔들리지는 않지만 의도했던 "동점 방지"와는 다르다. 동점이 실제로 몇 번 났는지는 투표 기록을 저장하지 않아서 모른다.

### 답 추출과 조용한 기본값

```python
def extract_choice(text: str) -> str:
    # 모델 출력에서 a/b/c/d 추출 (thinking 태그 제거 포함)
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)
    text = text.strip().lower()
    lines = [ln.strip() for ln in text.splitlines() if ln.strip()]
    if not lines:
        return "a"
    last = lines[-1]
    if last in {"a", "b", "c", "d"}:
        return last
    for tok in last.split():
        if tok in {"a", "b", "c", "d"}:
            return tok
    for ch in reversed(text):
        if ch in {"a", "b", "c", "d"}:
            return ch
    return "a"
```

Qwen3.5에는 thinking 모드가 있어서 `<think>` 블록부터 지운다. 그런데 `max_new_tokens`가 5라서 thinking이 켜진 채 생성이 `<think>` 안에서 잘리면 닫는 태그가 없어 정규식이 아무것도 못 지울 수 있다. 그러면 마지막 줄이 생각 중인 문장이 된다. 그 문장이 한국어라 a~d 글자가 하나도 없으면 마지막 `return "a"`로 빠지고(`think`라는 단어에도 a~d가 없다), 영문 글자가 섞여 있으면 그중 맨 뒤의 a~d를 집는다. 학습과 추론 양쪽에 `enable_thinking=False`를 명시하면(Option C·D 스크립트에 이미 있었고 `best.ipynb`가 이어받았다) 이 위험을 막을 수 있다.

출력이 비면 `"a"`를 돌려준다. `infer_one`에서 예외가 나도 `except Exception: pass`로 그 표만 버리고 5표가 다 사라지면 역시 `"a"`다. 두 경로 모두 로그를 남기지 않아서 제출 예측 분포를 대신 봤다. 0.973 제출 파일의 분포는 저장소에 없으니 아래는 `best.ipynb` 실행의 분포다.

![선지 글자별 비율](/assets/img/posts/vqa-qwen35-lora/answer-dist-light.png){: .light }
![선지 글자별 비율](/assets/img/posts/vqa-qwen35-lora/answer-dist-dark.png){: .dark }
_0.973 제출이 아니라 `best.ipynb` 출력에 남은 제출 5,074건의 분포. a는 24.8%다_

a가 1,259건(24.8%)으로 train 정답의 a 비율 24.9%와 거의 같다. 이 실행에서 기본값 `"a"`가 대량으로 터졌다면 a만 튀었을 것이다. 그래도 분포로 본 간접 확인일 뿐이라 기본값이 몇 번 쓰였는지 세는 카운터는 따로 두는 게 맞다.

## 7. 평가 방법과 실패한 실행들

### 평가는 두 가지뿐이었다

| 지표 | 무엇으로 | 한계 |
|---|---|---|
| val_loss | train의 10% 홀드아웃, 원래 선지 순서, assistant 턴 토큰 평균 | 정답 글자 외 토큰이 섞여 희석됨. 실행마다 검증셋이 다를 수 있음 |
| 정확도 | 테스트 예측 제출 | 제출 횟수가 제한된 외부 지표 |

코드에는 검증셋 정확도를 재는 부분이 없다. 검증 508문항에 TTA 추론을 돌려 정확도를 냈다면 실행끼리 같은 단위로 비교할 수 있었을 테고 9B도 제출 없이 정확도를 볼 수 있었다. 508문항이면 한 문항이 약 0.2%p다.

### Option D는 여러 변수를 한꺼번에 바꾸고 발산했다

0.973을 넘겨 보려고 Option A에서 한꺼번에 바꾼 항목은 다음과 같다.

| 항목 | Option A | Option D |
|---|---|---|
| 모델 | `unsloth/Qwen3.5-4B` | `unsloth/Qwen3.5-4B-Instruct` |
| 데이터 | train.csv | train + dev 4표 이상 902개 |
| 이미지 증강 | 없음 | 좌우 반전, ColorJitter, 회전 ±10° |
| 시스템 프롬프트 | 일반 | 재활용 도메인 특화 |
| LoRA r / alpha | 32 / 64 | 64 / 128 |
| LoRA dropout | 0.05 | 0 |
| 학습률 | 2e-5 | 2e-4 |
| gradient accumulation | 16 | 8 |
| 학습 해상도 상한 | 448² | 768² |
| 추론 해상도 상한 | 1,003,520 px (1280×28×28) | 1,638,400 px (1280²) |
| 에폭 | 3 | 5 (patience 3) |
| thinking | 템플릿 기본값 | `enable_thinking=False` |
| max_new_tokens | 5 | 4 |

val_loss가 크게 나와서 중단했다(README에 수치는 남아 있지 않다). README는 학습률을 원인으로 꼽았다. 검증된 2e-5에서 10배 올린 값이 가장 유력한 원인이긴 하지만 위 표처럼 학습률 말고도 12개 항목이 함께 바뀌어서 분리 실험 없이는 확정할 수 없다. 게다가 검증셋에 dev 문항이 섞이니 val_loss부터 Option A와 같은 시험지가 아니다.

좌우 반전은 이 과제에서 따로 따져 봐야 하는 증강이다. 질문이나 선지에 "오른쪽"·"왼쪽"이 들어간 문항이 train에 53개, dev에 69개 있다("마우스 오른쪽에 있다", "왼쪽 검은색 봉투 안" 같은 것). 이미지를 뒤집고 정답 글자는 그대로 두면 이런 문항에서는 틀린 정답을 가르칠 수 있고 라벨에 적힌 글자도 거울상이 된다.

### 이후 개선판(`best.ipynb`)

Option A의 학습률 2e-5와 r 32를 그대로 두고 학습 해상도를 768²로 올렸고 시스템 프롬프트도 5장의 도메인 특화판으로 바꿨다. `enable_thinking=False`는 Option C·D에, 누적 나머지 flush는 Option D에 이미 있었고 `best.ipynb`가 둘 다 이어받았다. README 기준으로는 학습이 아직 끝나지 않았다.

노트북에 남은 출력은 코드와 맞지 않는다. 출력은 "Train: 5977", "Epoch 1/1", val_loss 0.0428을 보여 주는데 지금 `train.csv`는 5,073행이고 코드의 `NUM_EPOCHS`는 3이다. 다른 데이터 파일과 에폭 설정으로 돌렸던 출력이 남아 있고 이 실행의 정확도 기록은 없다. 그래서 0.0428은 Option A의 0.02와 비교하지 않는다.

### 기록이 코드를 따라가지 못한 곳

| 위치 | 기록 | 실제 |
|---|---|---|
| `option_a_qwen35_4b_bf16.ipynb` | 4B BF16, 0.973 | 모델 셀이 `Qwen3.5-14B-Instruct`, `load_in_4bit=True`로 덮어써져 있다 |
| README, Option D | 학습 해상도 1024² | 코드는 768² |
| README, 설치 | transformers 4.51.3, torch nightly 필수 | Option A 노트북은 `"transformers>=5.0.0"`을 설치하고 출력이 transformers 5.3.0, torch 2.12.0.dev20260325+cu128(nightly). `best.ipynb` 출력은 transformers 5.3.0, torch 2.10.0+cu128(정식) |
| `option_c`, `option_d` 마스킹 | `-100 if len(pos) > 0 else ...` 폴백 | 우변 폴백을 계산한 뒤 대입 대상의 `pos[-1]`에서 IndexError가 나므로 폴백은 동작하지 않는다. assistant 턴이 항상 있어서 실제로 걸리지는 않았다 |
| README, dev 편입 | 2/5 이하 904개 | `dev.csv`로 세면 905개(최다 2표 903, 1표 1, 답 없음 1) |

지금 저장소의 Option A 노트북으로는 0.973 설정이 그대로 재현되지 않고, 위 표의 Option A 값도 README 기록이다. 실험할 때마다 노트북 셀을 고쳐 다시 쓰다 보니 이렇게 됐다. 실행 하나에 설정 파일 하나를 두고 출력과 함께 남겨 뒀어야 했다.

## 8. 한계와 다음 단계

- 9B 비교는 학습률, LoRA 크기, 해상도, 검증셋이 모두 달라서 양자화 효과를 분리하지 못했다. 같은 학습률과 같은 검증셋으로 9B NF4를 한 번 더 돌려야 결론이 선다.
- 검증 정확도 없이 val_loss만으로 실행을 골랐다. 검증셋 TTA 정확도와 기본값 `"a"` 사용 횟수를 함께 찍어야 한다.
- dev 편입 효과는 아직 확인하지 못했다. 규칙은 세웠지만 그 규칙을 적용한 실행이 발산했다. 학습률을 되돌린 상태에서 dev 902개만 더하는 단일 변수 실험이 남아 있다.
- README에 적힌 향후 계획은 `best.ipynb` 3에폭 결과 확인, 체크포인트 앙상블, Qwen2.5-VL-7B-Instruct 시도, `train_combined_v8767` 전체로 재학습이다.

코드: [github.com/kasangyong/vqa](https://github.com/kasangyong/vqa)
