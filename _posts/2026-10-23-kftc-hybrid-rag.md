---
title: "공정위 의결서에 묻고 답하는 AI 검색: 키워드와 의미 검색을 합친 RAG"
date: 2026-10-23 10:40:00 +0900
categories: [AI, RAG]
tags: [rag, bm25, bge-m3, rrf, reranker, exaone, lora, kiwipiepy, qdrant, fastapi]
math: true
image:
  path: /assets/img/posts/kftc-hybrid-rag/retrieval-ablation-light.png
  alt: 단독 검색과 하이브리드의 Recall@5 비교
---

공정거래위원회 의결서 500건에서 근거 청크 5개를 찾아 답변과 함께 돌려주는 RAG 시스템을 만들었다. 제2회 공정위 AI·데이터 활용 공모전에 팀 Retriever로 출전했고, 본선에 진출해 최종 1~3위로 수상했다.

| 항목 | 값 |
|---|---|
| 코퍼스 | 의결서 500건, 청크 31,879개 (정제·필터 후 27,076개) |
| 검색 | BM25(kiwipiepy) + BGE-M3 dense, RRF(k=60), Cross-encoder 리랭커, 제목 앵커링 |
| 생성 | 유형별 결정적 조립 (LLM은 폴백), EXAONE-3.5-2.4B + LoRA SFT |
| 결과 | 본선 진출, 최종 1~3위 수상 |
| 자체 검색 (사람 GT 35문항) | R@5 / MRR 0.971 / 0.957 → 1.000 / 1.000 (제목 앵커링, 질문에 제목이 포함된 셋) |
| 자체 생성 (정답 청크 제공) | mBERT 0.786 → 0.861 |
| 응답 시간 | LLM 생성 41.9초 → 결정적 생성 1.4초 |

{% include embed/video.html src='/assets/img/posts/kftc-hybrid-rag/explainer.mp4' poster='/assets/img/posts/kftc-hybrid-rag/explainer-poster.jpg' title='제목이 맞지 않는 질문 하나가 BM25·BGE-M3 병렬 검색, RRF 융합, 리랭커를 거쳐 근거 청크 5개와 결정적 답변이 되는 경로' loop=true %}

## 1. 문제와 제약

질문 하나에 `retrieved_chunk_ids` 정확히 5개와 `answer` 하나를 돌려주는 `/predict` 엔드포인트를 Docker 이미지로 제출한다. 걸린 제약은 이렇다.

| 제약 | 내용 | 설계에 준 영향 |
|---|---|---|
| 고정 청크 | 대회가 준 `*_hybrid.json`의 `chunk_id`를 그대로 반환해야 채점 | 재청킹 금지. 청크 내부 텍스트 정제만 한다 |
| 오프라인 | 평가 환경(A100 80GB) 인터넷 차단, 외부 API 금지 | 모든 모델·인덱스를 이미지에 굽는다 |
| 모델 크기 | 생성 모델 8B 이하 | EXAONE-3.5-2.4B |
| 응답 시간 | 문항당 30초 초과 시 해당 문항 0점 | 긴 생성이 위험하다 |
| 반환 개수 | 청크가 5개가 아니면 검색 점수 0점 | 무조건 5개로 채우는 패딩 |

채점식은 대회 도중에 바뀌었다.

$$
\text{초기: } 0.35\,R@5 + 0.15\,\mathrm{MRR} + 0.30\,\mathrm{BERTScore} + 0.20\,F_1
$$

$$
\text{변경 후: } 0.35\,R@5 + 0.15\,\mathrm{MRR} + 0.45\,\mathrm{BERTScore} + 0.05\,F_1
$$

F1이 전 참가팀에서 낮게 측정되자 운영팀이 F1을 20%에서 5%로 내리고 BERTScore를 30%에서 45%로 올렸다. 검색(R@5 + MRR) 비중 50%는 처음부터 끝까지 그대로였다.

## 2. 의결서 데이터 처리

### 입력 구조

의결서 1건은 두 파일이다. `*_hybrid.json`은 청크 배열(`page_content`와 `section`, `chunk_type`, `chunk_id` 메타데이터)이고, `*_metadata.json`은 제목, 공개일자, 피심인별 위반유형·조치유형을 담는다. 로더는 둘을 합쳐 청크마다 문서 메타데이터를 붙인다.

```python
@dataclass
class Chunk:
    chunk_id: str
    page_content: str
    section: str          # "주문" | "이유"
    chunk_type: str       # "text" | "table"
    # ... 생략
    doc_id: str
    doc_title: str
    피심인기업명: list[str]
    위반유형: list[str]
    세부위반유형: list[str]
    조치유형: list[str]
```

`chunk_id`는 `DOC-{uuid}-CH-{번호}` 형식이다. 섹션은 결론(피심인, 시정명령, 과징금)을 담은 **주문**과 사실관계와 법리를 담은 **이유**로 나뉘는데, 이 구분은 뒤에서 검색과 청크 선택, 답변 생성에 모두 쓰인다.

피심인 정보는 피심인마다 한 줄이라 같은 위반유형이 여러 번 나오므로 집합으로 중복을 지우고 빈 문자열은 거른다.

```python
기업명 = [v for v in {str(p.get("피심인기업명", "")) for p in 피심인정보} if v]
위반유형 = [v for v in {str(p.get("위반유형", "")) for p in 피심인정보} if v]
```

### 청크 경계는 건드리지 않는 정제

청크는 PDF를 변환한 텍스트라 페이지 번호(`- 3 -`), 각주 번호(`기준2)`), 페이지 넘김으로 끊긴 문장이 섞여 있다. 경계를 바꾸면 `chunk_id`가 의미를 잃으니 청크 안에서만 고친다.

```python
def clean_chunk(text: str) -> str:
    # 1. 각주 번호 제거: "사업자1)", "기준2)" → "사업자", "기준"
    text = re.sub(r"(\S)\d+\)", r"\1", text)

    # 2. 줄 단위: 페이지번호 단독 줄("- 1 -") 제거 + 그로 인해 끊긴 문장 잇기
    lines = text.split("\n")
    out: list[str] = []
    join_next = False
    for raw in lines:
        s = raw.strip()
        # ... 빈 줄 처리 생략
        if _PAGE_NUM_LINE.match(s):
            join_next = True
            continue
        if join_next and out:
            j = len(out) - 1
            while j >= 0 and out[j].strip() == "":
                j -= 1
            # 직전 줄이 문장종결로 안 끝나고, 현재 줄이 리스트마커로 시작 안 하면 → 이어붙임
            if j >= 0 and not _SENT_END.search(out[j]) and not _LIST_MARKER.match(raw):
                cont = re.sub(r"^\s*[-–•]\s*", "", s)
                out[j] = out[j].rstrip() + cont
                join_next = False
                continue
        join_next = False
        out.append(raw)
    # ... 3~6단계(인라인 페이지번호, 중복 줄, 빈 줄 압축, 공백 정리) 생략
```

페이지 번호 줄을 지우기만 하면 앞뒤 문장이 줄바꿈으로 갈라진 채 남는다. 그래서 직전 줄이 `다.`, `함`, `호`, `항` 같은 종결로 끝나지 않고 현재 줄이 `1.`, `가.`, `①` 같은 목록 기호로 시작하지 않을 때만 두 줄을 잇는다. 단어 중간에 낀 공백(`않 고`)은 법률 용어를 잘못 붙일 위험이 있어 고치지 않았다.

표 청크는 Markdown 표를 `[표]`와 문서 제목을 앞에 붙인 `헤더: 값` 나열로 바꾼다. 그래야 임베딩과 BM25가 열 이름과 값을 같이 본다.

### 길이 필터를 전부 풀었더니 더 나빴다

처음엔 기호를 뺀 글자 수가 80자 미만인 청크를 모두 색인에서 뺐다. 그런데 주문 섹션의 정답 청크는 짧을 때가 있어서, "피심인은 …행위를 다시 하여서는 아니 된다." 한 문장은 68자밖에 안 된다. 그 바람에 자체 평가 100문항 중 11개 문항은 정답 청크가 색인에 아예 없었고 R@5 천장도 0.89 근처에서 막혔다.

| 시도 | 주문 | 이유 | 결과 |
|---|---|---|---|
| 기존 | 80자 | 80자 | 정답 청크 11개 누락 |
| 전면 완화 | 10자 | 20자 | R@5 0.971 → 0.943 하락 |
| **채택** | **10자** | **80자** | 정답 청크 복구, 노이즈 억제 |

반대로 이유 섹션의 짧은 조각까지 다 넣었더니 그 조각들이 top-5를 차지해 정답을 밀어냈다. 그래서 기준을 섹션별로 다르게 둔다.

```python
MIN_LEN_JUMUN = 10
MIN_LEN_OTHER = 80

def keep_chunk(text: str, section: str = "") -> bool:
    min_len = MIN_LEN_JUMUN if section == "주문" else MIN_LEN_OTHER
    return is_valid_chunk(text, min_len=min_len)
```

이 정제·필터는 색인 스크립트, API 서버 기동, 평가 스크립트에 똑같이 적용한다. BM25 캐시와 Qdrant 색인은 미리 만들어 이미지에 넣고, 서버는 기동할 때 청크를 다시 정제해 `id_to_chunk`·앵커·답변 생성에 쓴다. 정제 규칙이 어긋나면 색인된 텍스트와 서버가 아는 청크가 달라진다.

## 3. 하이브리드 검색

### 희소: 형태소 BM25

한국어는 조사가 붙어서 공백으로 자르면 "삼성전자가"와 "삼성전자"가 서로 다른 토큰이 된다. 그래서 kiwipiepy로 형태소를 나누고 일반명사, 고유명사, 동사, 형용사만 남긴다.

```python
_VALID_TAGS: frozenset[str] = frozenset({"NNG", "NNP", "VV", "VA"})

def tokenize(self, text: str) -> list[str]:
    tokens = self._kiwi.tokenize(text)
    return [
        tok.form for tok in tokens
        if tok.tag in _VALID_TAGS and tok.form not in _LEGAL_STOPWORDS and len(tok.form) > 1
    ]
```

`_LEGAL_STOPWORDS`는 "대한", "관한", "따라", "위하여" 같은 기능어 25개다. 인덱스는 `rank_bm25.BM25Okapi` 기본값($k_1=1.5$, $b=0.75$)으로 만들고 pickle로 캐시한다.

### 밀집: BGE-M3 + Qdrant

BGE-M3(1024차원)로 청크를 임베딩해 Qdrant에 코사인 거리로 넣었다. BGE-M3를 고른 건 한국어 성능과 8192 토큰 문맥, 그리고 오프라인으로 로드할 수 있어서다. 외부 임베딩 API는 오프라인 규정에 걸린다. Qdrant는 위반유형, 공개연도, 섹션 페이로드 필터를 쓰려고 골랐는데, 이 필터는 `/api/search`와 `/api/qa`에서만 쓴다.

### 점수 대신 순위로 융합

BM25 점수는 수십에서 수백인데 코사인은 0~1이라 그대로 더하면 BM25가 압도한다. 가중합을 쓰자니 정규화와 가중치 튜닝을 따로 해야 한다. RRF는 점수를 버리고 순위만 쓴다.

$$
\mathrm{RRF}(d) = \sum_{r \in \{\text{BM25},\,\text{Dense}\}} \frac{1}{k + \mathrm{rank}_r(d)}, \qquad k = 60
$$

```python
def reciprocal_rank_fusion(
    *result_lists: list[tuple[str, float]],
    k: int = 60,
) -> list[tuple[str, float]]:
    scores: dict[str, float] = {}
    for results in result_lists:
        for rank, (doc_id, _) in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

`enumerate`가 0부터 세므로 `rank + 1`로 1위부터 맞춘다. 두 검색기에서 모두 1위인 청크는 $2/61 \approx 0.0328$, 한쪽에서만 1위이고 다른 쪽 top-100에 없으면 $1/61 \approx 0.0164$다. $k=60$이면 1위와 10위의 차이가 $1/61 - 1/70 \approx 0.0021$로 작아서, 한 검색기에서만 높은 청크보다 양쪽에서 중간쯤인 청크가 위로 올라온다. 각 검색기는 top-100을 낸다.

![단독 검색 대비 하이브리드 Recall@5](/assets/img/posts/kftc-hybrid-rag/retrieval-ablation-light.png){: .light }
![단독 검색 대비 하이브리드 Recall@5](/assets/img/posts/kftc-hybrid-rag/retrieval-ablation-dark.png){: .dark }
_단독 검색과 하이브리드의 Recall@5 비교(README 4-3)_

하이브리드를 택한 건 법률 문서에 성격이 다른 단서 두 종류가 섞여 있어서다. 회사명, 법 조항, 금액은 글자 그대로 맞아야 하고(BM25), "부당한 공동행위"와 "담합" 같은 표현 차이는 의미로 맞춰야 한다(dense).

### 기각했다가 다시 넣은 리랭커

`bge-reranker-v2-m3-ko`는 (질문, 청크) 쌍을 같이 넣어 점수를 매기는 Cross-encoder다. RRF 상위 `top_k × 4`개를 다시 정렬한다.

| 시점 | 평가셋 | 결과 | 판단 |
|---|---|---|---|
| v5 | 사람 GT 셋 | R@5 0.983 → 0.980, MRR 0.980 → 0.985 | 이미지 +2.3GB 대비 이득 없음, 제외 |
| 2차 제출 전 | 패러프레이즈셋 | MRR 0.862 → 0.964 | 채택 |
| 2차 공식 점수 | 비공개 | R@5 30.45 → 31.35, MRR 10.99 → 11.13 | 공식 점수로 전이됨 |

사람 GT 셋은 MRR이 이미 0.98이라 더 올라갈 자리가 없었고 질문 표현을 바꾼 패러프레이즈셋을 따로 만들고 나서야 효과가 보였다. 같은 기능을 두고도 평가셋 난이도에 따라 결론이 뒤집힌 셈이다.

## 4. 제목 앵커링

### 관찰

대회 표준 형식으로 만든 평가셋 질문은 의결서 제목을 통째로 담고 있다. "(주)에듀윌의 부당한 표시광고 행위에 대한 건 사건의 피심인은?" 같은 식이다. 이러면 검색 순위와 상관없이 제목 문자열만으로 정답 문서를 먼저 고정할 수 있다.

```python
# 제목 앵커링용 정규화, 공백·문장부호·가운뎃점 제거, 한글/영숫자만
_NORM = re.compile(r"[^0-9A-Za-z가-힣]")

def _norm(s: str) -> str:
    return _NORM.sub("", s or "")

# __init__ 중
for c in chunks:
    self._doc_chunks.setdefault(c.doc_id, []).append(c)
    if c.doc_title:
        n = _norm(c.doc_title)
        if len(n) >= 10:
            self._title_to_doc[n] = c.doc_id

def _anchor_doc(self, query: str) -> Optional[str]:
    """질문에 의결서 제목이 통째로 들어있으면 그 doc_id 반환(가장 긴 매칭). 없으면 None."""
    nq = _norm(query)
    best_doc, best_len = None, 0
    for ntitle, doc_id in self._title_to_doc.items():
        if len(ntitle) > best_len and ntitle in nq:
            best_doc, best_len = doc_id, len(ntitle)
    return best_doc
```

여기에는 결정이 세 가지 들어 있다.

- 정규화는 질문을 쓴 사람이 `(주)`의 괄호를 빼거나 가운뎃점을 바꿔도 맞도록 한글과 영숫자만 남긴다.
- 10자 하한을 둔 건 짧은 제목이 다른 질문에 우연히 부분 문자열로 들어갈 수 있어서다.
- "A 사건"과 "A 사건 관련 B의 건"처럼 한 제목이 다른 제목을 포함하면 최장 매칭으로 긴 쪽을 고른다.

### 문서를 찾은 뒤 유형별로 청크 정렬

채점이 청크 단위라 문서만 맞혀서는 부족하다. 자체 평가셋 GT를 보니 사실형 질문(피심인, 과징금, 조치, 결론)은 정답 청크가 주문과 머리 부분(CH-001, CH-002)에 있었고 위반행위 질문은 이유 섹션에 있었다.

```python
def _ordered_doc_chunks(self, doc_id: str, question_type: str) -> list[Chunk]:
    cs = self._doc_chunks.get(doc_id, [])
    if question_type == "violation":
        이유 = [c for c in cs if c.section == "이유"]
        나머지 = [c for c in cs if c.section != "이유"]
        return 이유 + 나머지
    # 사실형/미상: 주문·머리(낮은 CH번호) 먼저, 이미 CH 순 정렬됨
    return cs
```

앵커가 잡히면 그 문서 청크를 앞에 두고, 남는 자리는 하이브리드 결과로 채운다. 앵커가 안 잡히면 원래대로 하이브리드 검색을 탄다.

```python
if self.use_anchor:
    anchored = self._anchor_doc(query)
    if anchored:
        doc_first = self._ordered_doc_chunks(anchored, question_type)
        bm25_res  = self._bm25.search(query, self._bm25_top_k)
        dense_res = self._dense.search(query, self._dense_top_k, filter_params)
        fused     = reciprocal_rank_fusion(bm25_res, dense_res)
        seen = {c.chunk_id for c in doc_first}
        others = [self.id_to_chunk[cid] for cid, _ in fused
                  if cid in self.id_to_chunk and cid not in seen]
        return (doc_first + others)[:top_k]
```

![제목 앵커링 전후 검색 지표](/assets/img/posts/kftc-hybrid-rag/anchor-light.png){: .light }
![제목 앵커링 전후 검색 지표](/assets/img/posts/kftc-hybrid-rag/anchor-dark.png){: .dark }
_`test/eval_retrieval_only.py` 결과_

### 1.00을 그대로 믿으면 안 되는 이유

이 숫자가 왜 나오는지는 평가셋 파일을 열어 보면 알 수 있다. 문항마다 질문에 정규화한 정답 문서 제목이 들어 있는지 세어 봤다.

| 평가셋 | 문항 | 질문에 정답 제목 포함 |
|---|---|---|
| `eval_testset_human.json` | 35 | 35 |
| `eval_testset_human100.json` | 100 | 100 |
| `eval_testset_v2.json` | 100 | 100 |
| `eval_testset_paraphrase.json` | 35 | 0 |

사람 GT 셋은 모든 질문이 제목을 담고 있어 앵커가 매번 정답 문서를 잡고 자체 R@5/MRR은 문서 단위로 매칭하므로(7장) 그 순간 1.00이 나온다. 패러프레이즈셋은 반대로 앵커가 한 번도 발동하지 않았으니 MRR 0.862 → 0.964는 앵커가 아닌 리랭커 경로에서 나온 효과다.

1.00은 결국 "공식 질문도 제목을 담고 있다"는 가정이 맞을 때의 값이다. 이 가정은 공식 질문 형식과 같은 형식으로 만든 평가셋을 근거로 세웠고, 앵커링을 넣은 3차 제출의 점수는 비공개였다. 그래도 대회가 제출작 중 최고점을 채택하는 규칙이라 3차가 나빠도 2차 점수가 남으니 손해 볼 것 없는 시도였다.

앵커가 잡히면 리랭커를 건너뛴다(응답 1~2초). 남는 자리는 RRF 순서로 채운다.

주의할 점이 하나 있다. `_title_to_doc`는 제목을 키로 쓰기 때문에 정규화 후 같은 제목인 의결서가 둘이면 나중에 읽힌 문서만 남는다.

## 5. 결정적 생성

### LLM을 주력에서 뺀 이유

처음 구조는 EXAONE-3.5-2.4B + LoRA에 검색 청크와 유형별 형식 지시를 주는 방식이었는데, 세 가지 문제에 부딪혔다.

- 조기 EOS. 8GB GPU에서 돌린 v3 모델이 긴 답변 중간에 스스로 EOS를 내서 답이 끊겼다. 출력 상한(512)과는 무관했다.
- 속도. 50문항 벤치마크에서 평균 41.9초였다. 문항당 30초를 넘기면 그 문항이 0점이다.
- 입력 폭발. 피심인 수십 곳이 나열된 카르텔 사건은 주문 섹션이 길어서 100문항 중 6문항이 1,000초를 넘겼다(test_0050은 2,029초). 입력을 3,072토큰에서 자르면 내부 점수가 0.8410에서 0.81로 떨어졌다.

입력은 왼쪽을 자른다. 기본값(오른쪽 절단)으로 두면 맨 끝의 `[질문]`과 생성 프롬프트가 잘려 빈 답이 나왔다.

```python
self._tokenizer.truncation_side = "left"
# ...
inputs = self._tokenizer(text, return_tensors="pt", max_length=3072, truncation=True)
```

그런데 따져 보면 이 과제의 답은 원문 안에 이미 있는 사실이다. 피심인과 조치유형은 메타데이터에, 과징금은 주문 섹션에 있다. 자유 생성이 필요 없는 추출형 질의응답이니 질문 유형별로 사실을 코드로 조립하고 LLM은 조립할 재료가 없을 때만 부르도록 바꿨다.

```python
def predict_answer(self, question, chunks, question_type=""):
    from .facts import extract_facts, build_structured_answer, build_generic_answer

    # 유형별 결정적 생성 (메타데이터+섹션 원문 조립)
    det = build_structured_answer(chunks, question_type)
    if det:
        return det

    # 유형 미분류/데이터 부족 → 약한 LLM보다 '결정적 일반 답변'(메타+섹션)을 우선
    generic = build_generic_answer(chunks, question_type)
    if generic:
        return generic

    # 그래도 만들 게 없으면 최후로 LLM 생성
    # ... 컨텍스트 + [확정 사실] 블록 + 형식 지시로 프롬프트 구성 생략
    ans = self._llm.generate(messages)
    # LLM이 쓰레기('|', 빈 문자열 등)를 내면 결정적 폴백
    stripped = ans.strip()
    if len(stripped) < 10 or not any(ch.isalnum() for ch in stripped):
        fb = build_generic_answer(chunks, question_type)
        if fb:
            return fb
    return ans
```

응답 시간은 41.9초에서 1.4초로 줄었다. 답이 원문 사실의 조합이라 원문에 없는 회사명이나 금액이 끼어들 여지도 없다.

### 질문 유형 분류

조립 규칙을 고르려면 질문 유형을 알아야 하는데 `/predict`는 질문 문자열만 받는다. 그래서 LLM 없이 키워드로 분류한다.

```python
_QTYPE_KEYWORDS: list[tuple[str, list[str]]] = [
    ("fine",       ["과징금", "과태료", "얼마", "금액", "부과", "제재금", "벌금", "납부할", "납부해야"]),
    ("measure",    ["조치", "시정", "처분", "처벌", "명령", "제재를", "제재 ", "불이익", "내려진", ...]),
    ("conclusion", ["결론", "의결 결과", "어떻게 결정", "최종 결정", "내린 결론", "마무리", ...]),
    ("violation",  ["주요 위반", "위반 행위", "위반 내용", "어떤 위반", "무슨 위반", "위반유형", ...]),
    ("respondent", ["누구", "누가", "피심인", "기업명", "회사명", "어느 업체", "어느 회사", ...]),
]
```

여기서는 순서가 중요하다. "피심인에게 내려진 조치는?"에도 "피심인"이 들어가므로 respondent를 맨 뒤에 둔다. violation 키워드에 "행위"나 "위반"을 그냥 넣어서도 안 된다. 질문에 들어 있는 제목("…부당한 공동행위에 대한 건")에 걸리기 때문이다. 동의어를 확장하다가 실제로 이 실수를 해서 respondent 질문 16개가 violation으로 분류됐고 바로 되돌렸다. 지금은 "행위"·"위반"을 단독 키워드로는 넣지 않는다.

| | 기존 100문항 | 패러프레이즈 18문항 |
|---|---|---|
| 동의어 확장 전 | 97/100 | 2/18 (16개 LLM 폴백) |
| 확장 후 | 100/100 | 18/18 |

### 유형별 조립

```python
if question_type == "respondent":
    names = list(top.피심인기업명 or [])
    if not names:
        return None
    if len(names) > 8:
        return f"이 사건의 피심인은 {', '.join(names[:8])} 등 {len(names)}개 사업자이다."
    return "이 사건의 피심인은 " + ", ".join(names) + "이다."
```

`top`은 검색 1위 청크(`chunks[0]`)다. 메타데이터는 문서에 붙어 있으므로 1위 청크의 문서가 곧 답의 대상이다. 과징금은 메타데이터에 금액이 없어 주문 섹션에서 뽑아야 했고 버그도 여기서 가장 많이 나왔다.

**버그 1. 다른 금액을 과징금으로 읽었다.** 처음 로직은 주문에서 큰 금액 상위 3개를 뽑았는데, 하도급 사건에서는 감액된 하도급대금과 지연이자가 주문에 같이 나와서 정답 87백만 원 대신 538백만 원을 답했다. 그래서 "과징금…금액"이나 "과징금을…납부하여야"가 있는 줄을 앵커로 삼고 그 뒤 줄만 보도록 바꿨다. 하도급대금, 지연이자, 감액 같은 단어가 든 줄은 건너뛴다.

```python
_NON_FINE_HINT = ("하도급대금", "지연이자", "감액", "차액", "미지급", "대금을", "대금 ", "대금의")

for raw in jumun.split("\n"):
    ln = raw.strip()
    if not ln:
        continue
    is_anchor = (kw in ln) and ("금액" in ln or "납부하여야" in ln)
    if is_anchor:
        in_fine = True
        for m in _find_amounts(ln):
            if ("총" in ln) or ("합계" in ln):
                total = m
            else:
                amounts.append(m)
        continue
    if not in_fine:
        continue
    if any(x in ln for x in _NON_FINE_HINT):
        continue
    # ... 금액 수집, '총/합계' 줄은 total로 생략
if total:
    return [total]
```

**버그 2. 표기 형식과 순서.** 500건 전체에 결정적 생성을 돌려 감사해 보니 과징금 조치인데 금액이 안 나온 문서가 8건이었다. "51백만 원"처럼 한글 단위로 쓴 금액을 쉼표 숫자 정규식이 놓쳤고, "과징금액 → 납부기한 → 납부장소 → 개별 금액" 순서인 의결서에서 "납부기한"에서 블록을 닫아 개별 금액을 놓쳤다. 한글 단위 정규식을 넣고 종료 경계를 없애 5건을 고쳤다. 금액이 별표 표에만 있는 3건은 표 파서 없이 "과징금이 부과되었으며, 구체적인 과징금액은 의결서 별표에 기재되어 있다"로 답한다.

**버그 3. 합산하면 두 배가 된다.** 카르텔 사건은 주문에 사업자별 과징금과 소계가 섞여 있다. 개별 금액을 자동 합산하자 1,973억이 3,947억이 되어 버렸다. 그래서 합산은 버리고 주문에 적힌 총액만 쓴다.

**버그 4. 다른 사건의 사실이 섞였다.** `/predict`는 상위 3개 문서의 주문 청크를 컨텍스트에 보강하는데, 사실 추출까지 이 청크들을 다 보면 1위가 아닌 문서의 금액이 답에 들어간다. 그래서 사실 추출은 1위 문서 청크로만 한정한다.

```python
def top_doc_chunks(chunks: list[Chunk]) -> list[Chunk]:
    """상위 문서(chunks[0].doc_id)에 속한 청크만, 사실 추출의 교차문서 오염 방지."""
    if not chunks:
        return []
    top_id = chunks[0].doc_id
    same = [c for c in chunks if c.doc_id == top_id]
    return same or list(chunks)
```

### 원문 덤프를 빼고 짧게

초기 답변은 리드 문장 뒤에 주문 원문 수백 자를 그대로 붙였는데, 사람이 쓴 정답은 한두 문장 요약이라 긴 법조문이 BERTScore precision과 F1을 깎아 먹었다. 그래서 원문 덤프를 빼고 메타데이터 사실 위주로 다시 썼다. 금액은 `64,000,000원` 대신 `6,400만 원`처럼 한글로 쓰고, 어미는 정답 어투(`~이다`, `~하였다`)에 맞췄다. 행위 내용이 필요한 결론·위반행위 답에는 주문의 첫 "피심인 …" 문장에서 회사 나열과 "다시 하여서는 아니 된다" 명령부를 정규식으로 걷어낸 행위 서술만 붙인다. 조치(measure) 답은 정답이 짧아서 행위 서술을 붙이면 오히려 0.842에서 0.792로 떨어져서 붙이지 않는다.

| 단계 | 생성 품질 (mBERT, 사람 GT 35문항, 정답 청크 제공) |
|---|---|
| 초기 제출본 (원문 덤프) | 0.786 |
| 원문 덤프 제거, 금액 한글화 | 0.844 |
| 평서체 전환 | 0.861 |

## 6. LoRA SFT

결정적 생성으로 바꾸기 전에 EXAONE-3.5-2.4B를 의결서 도메인으로 미세조정했다. 지금 코드에서 LLM을 부르는 곳은 `/api/risk` 리스크 진단, `/api/qa`, `/query`, 그리고 `/predict`의 최후 폴백이다.

### 데이터

`train/generate_sft_dataset.py`가 의결서 메타데이터와 섹션으로 Chat Template 형식 데이터를 만든다.

| 타입 | 건수 | 내용 |
|---|---|---|
| Type1 검색·요약 | 2,092 | 문서마다 피심인, 위반유형, 조치, 과징금(있을 때), 종합요약 질문 |
| Type2 리스크 | 183 | 위반유형별 시나리오 8종 → 유사 의결 사례 기반 진단 |
| Type3 계약 검토 | 124 | 가맹, 하도급, 대리점, 약관 조항 |
| 합계 | 2,399 | |

평가셋 `eval_testset_v2.json`에 쓰인 문서는 학습에서 뺀다.

```python
def load_eval_doc_ids() -> set[str]:
    # ...
    return {c.get("relevant_doc_id", "") for c in cases if c.get("relevant_doc_id")}

# generate_type1 중
if doc_id.replace("DOC-", "") in eval_doc_ids:
    continue  # 평가셋 제외
```

### 설정과 결과

```python
LORA_R       = 16
LORA_ALPHA   = 32
LORA_DROPOUT = 0.05
TARGET_MODULES = ["q_proj", "k_proj", "v_proj", "out_proj",
                  "c_proj", "c_fc_0", "c_fc_1"]
```

EXAONE 구현의 모듈 이름에 맞춰 어텐션의 q/k/v/out 투영과 MLP의 `c_fc_0`, `c_fc_1`, `c_proj`까지 걸었다. 학습 파라미터는 21,995,520개로 전체의 0.91%다.

| 항목 | 값 |
|---|---|
| 에포크 / 유효 배치 | 3 / 4 (배치 1 × 누적 4) |
| 학습률 | 2e-4, 코사인, 워밍업 50스텝 |
| 최대 길이 | 512 |
| 정밀도 | bf16 (4-bit 미사용) |
| train / eval 분할 | 9:1, seed 42 |
| eval_loss | 0.1934 (최종 train_loss 0.1196) |
| eval 토큰 정확도 | 95.03% |
| 학습 시간 | 35분 35초 (RTX 5060 Ti) |

4-bit를 못 쓴 건 환경 탓이다. RTX 5060 Ti(Blackwell, sm_120)가 PyTorch 2.5.1+cu121에서 안 돌아 nightly cu128로 올렸고, 그 조합에서 bitsandbytes가 Windows CUDA 12.8을 지원하지 않았다. 17GB VRAM이면 2.4B를 bf16으로 LoRA 학습하기에 충분해서 양자화 없이 학습했다.

### 이 점수의 한계

eval 토큰 정확도 95%는 학습 데이터와 같은 템플릿으로 만든 답을 얼마나 따라 쓰는지를 잰다. Type1의 답 형식("이 사건의 피심인은 …입니다.")과 자체 평가셋 v2의 GT 형식("피심인은 …입니다.")은 같은 메타데이터에서 거의 같은 틀로 만든다. 그래서 v2 기준으로 잰 SFT 어댑터 점수(100문항 0.8410)도 7장의 자기참조 문제에서 자유롭지 않다.

제출 이미지는 SFT 어댑터(`exaone-3.5-2.4b-sft-all/adapter`)를 폴백 LLM으로 싣고, 5개 유형의 `/predict` 답은 LLM을 거치지 않는다.

## 7. 자체 평가: 0.9362는 허수였다

v3에서 내부 점수가 0.9362까지 올랐는데, 알고 보니 GT를 답변 생성 로직과 같은 방식(메타데이터 + 주문 섹션 템플릿)으로 만들고 있었다. 답과 정답이 같은 틀에서 나오니 닮을 수밖에 없다.

그래서 사람이 자연어로 따로 쓴 GT 35문항(`eval_testset_human.json`)을 만들었다. 질문과 검색은 그대로 두고 정답 표현만 독립적으로 썼다.

![템플릿 GT 대비 자연어 GT 점수](/assets/img/posts/kftc-hybrid-rag/self-reference.png)
_발표 자료. 질문·검색·답변이 같고 GT만 다르다_

| GT | BERT | F1 | Final |
|---|---|---|---|
| 템플릿 GT | 0.932 | 0.823 | 0.928 |
| 자연어 GT | 0.766 | 0.358 | 0.785 |

검색 쪽은 같으니 차이는 생성 지표에서만 나와야 하고 초기 가중치로 검산해 보면 실제로 맞는다.

$$
\Delta\text{Final} = 0.30 \times (0.932 - 0.766) + 0.20 \times (0.823 - 0.358) = 0.0498 + 0.0930 = 0.1428
$$

$0.928 - 0.785 = 0.143$이다. 이후 생성 개선은 전부 자연어 GT로 쟀다(35문항 0.785 → v4 0.839, 100문항 기준 v4 0.833 → v6 0.835).

자체 지표가 어떻게 구현됐는지도 알고 봐야 한다.

```python
def _doc_id(chunk_id: str) -> str:
    """DOC-{uuid}-CH-XXX → DOC-{uuid} (문서 단위 비교용)"""
    parts = chunk_id.rsplit("-CH-", 1)
    return parts[0] if len(parts) == 2 else chunk_id

def mrr(retrieved: list[str], relevant: list[str]) -> float:
    relevant_docs = {_doc_id(r) for r in relevant}
    for rank, chunk_id in enumerate(retrieved, 1):
        if _doc_id(chunk_id) in relevant_docs:
            return 1.0 / rank
    return 0.0
```

- R@5와 MRR은 문서 단위다. MRR은 정답 문서의 첫 청크 순위로, R@5는 top-5 중 정답 문서 청크 수를 정답 청크 수로 나눈 값(상한 1)으로 잰다. 대회 가이드는 정답 청크 기준으로 설명한다.
- 자체 BERTScore는 BGE-M3 코사인 근사(`bert_score_simple`)로, 토큰 단위 BERTScore가 아니라 문장 임베딩 두 개의 코사인 유사도다.

자체 R@5 0.97과 공식 0.87의 차이는 자체 평가셋이 공식보다 쉬운 탓이 크고 정답 문서의 다른 청크도 맞은 것으로 치는 문서 단위 매칭도 차이의 한 원인일 수 있다.

## 8. 공식 채점에서 배운 것

![2차 제출 공식 점수 항목별 득점](/assets/img/posts/kftc-hybrid-rag/official-score-light.png){: .light }
![2차 제출 공식 점수 항목별 득점](/assets/img/posts/kftc-hybrid-rag/official-score-dark.png){: .dark }
_괄호는 만점 대비 비율_

공식 점수는 대회 운영팀이 비공개 질문·정답으로 잰 값이다. 가중치 변경 후 식이라 항목별 만점이 35, 15, 45, 5다.

| 항목 | 만점 | 1차 | 2차 | 2차 비율 |
|---|---|---|---|---|
| Recall@5 | 35 | 30.45 | 31.35 | 0.896 |
| MRR | 15 | 10.99 | 11.13 | 0.742 |
| BERTScore | 45 | 21.37 | 21.29 | 0.473 |
| F1 | 5 | 0.02 | 0.02 | 0.004 |

표에서 세 가지를 읽을 수 있다.

1. 검색은 50점 중 42.48점, 생성은 50점 중 21.31점이다. 생성 쪽 감점은 대부분 BERTScore(45점 중 23.71점 감점)에서 나왔다.
2. 검색 개선은 공식 점수로 전이됐지만 생성 개선은 그러지 못했다. 2차에서 리랭커를 켜자 R@5가 +0.90점, MRR이 +0.14점 올랐다. 같은 차수에 답변을 다시 썼고 자체 mBERT는 0.786에서 0.844로 올랐지만 공식 BERTScore는 21.37에서 21.29로 그대로였다.
3. 공식 BERTScore 0.473과 자체 0.861은 같은 자로 잰 값이 아니다. 공식 채점의 모델과 설정이 비공개라 baseline 보정 여부를 알 수 없었다. 공식 F1 0.004도 운영팀이 "평가데이터 특성상 전 팀이 낮다"고 공지한 값이다.

마지막 라운드 방향은 2번이 정했다. 생성은 공식 정답의 어투를 모르는 상태에서 자체 GT에 맞추는 일이었고, 검색은 자체 개선이 공식 점수로 넘어간다는 게 확인된 레버였다. 그래서 3차 제출은 검색 천장을 노리는 제목 앵커링에 썼다.

## 9. 그 밖의 실패

| 실패 | 증상 | 대응 |
|---|---|---|
| 오프라인에서 즉사 | 이미지에 BGE-M3가 없었다. 로컬은 인터넷이 있어 런타임 다운로드로 우연히 돌았다. `--network none`에서 기동 실패 | 빌드 때 BGE-M3를 HF 캐시에 굽는다 |
| 캐시가 있어도 즉사 | huggingface_hub가 시작 시 업데이트 확인 HEAD 요청을 보낸다 | `HF_HUB_OFFLINE=1`, `TRANSFORMERS_OFFLINE=1`을 다운로드 `RUN` 뒤에 둔다 |
| 컨테이너 30초 자살 | Qdrant 콜드 스타트 로딩이 대기 시간을 넘겼다 | 헬스체크 대기 30초 → 120초 |
| 상호 변경 | "대성산업(주) 사건의 피심인은?"에 의결서 기재명은 "(주)대성합동지주" | 메타데이터 그대로 답하는 한 못 고친다 |
| 표기명 차이 | 앵커링 이전, 에듀윌 건 하나가 다른 표기로 색인되어 BM25와 dense 모두 놓쳤다(R@5 0) | 따로 고치지 않았다 |

오프라인 문제는 마지막 점검에서 잡았다. 이미지가 무엇을 담고 있는지 확인하다 Dockerfile에 BGE-M3가 없는 걸 보고 `docker run --network none`으로 재현했다. 놓쳤다면 질문 임베딩을 못 해 검색 50점이 통째로 날아갈 수도 있었다. 최종 이미지는 `--network none`에서 `/health` 200과 `/predict` 정상 응답을 확인했고, 사람 GT 35문항 전수 실행에서 빈 답, 청크 이상, 크래시가 0건이었다.

```dockerfile
RUN python3 -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-m3')" \
    && rm -rf /root/.cache/huggingface/modules/
# ...
ENV RERANKER_MODEL=/app/models/bge-reranker-v2-m3-ko
ENV USE_4BIT=0
# ...
ENV HF_HUB_OFFLINE=1
ENV TRANSFORMERS_OFFLINE=1
```

평가 환경이 A100 80GB라 제출 이미지는 4-bit를 끄고 bf16으로 띄운다. 이미지에는 Qdrant 바이너리, BM25 캐시, Qdrant 저장소, 의결서, 모델이 전부 들어간다. 앵커링을 넣은 v3a 이미지는 22.41GB였다.

## 10. 본선 데모

![데모 첫 화면](/assets/img/posts/kftc-hybrid-rag/demo-home.png)
_[kftc-rag.vercel.app](https://kftc-rag.vercel.app)_

본선용 데모는 광고 문구나 계약 조항을 붙여 넣으면 비슷하게 제재받은 의결서를 보여 주는 화면이다. 프런트엔드만 정적으로 배포했고 백엔드를 호출하지 않는다. 유사 사례는 실제 검색 엔진 응답을 캡처해 넣었고, 보여 줄 사례는 고르되 답을 손으로 쓰지는 않는다는 원칙을 두었다. 하단 입력창의 자유 질문은 준비된 추천 질문과 매칭해 답한다.

## 11. 한계와 다음 단계

- BERTScore를 직접 최적화할 신호가 없었다. 45% 비중 항목인데 공식 정답과 채점 설정이 비공개라, 생성 개선은 모두 자체 지표 기준의 추측이었다.
- 자체 평가셋이 공식보다 쉽다. 자체 R@5 0.97 대 공식 0.87이다. 문서 단위 매칭은 정답 문서의 다른 청크도 맞은 것으로 쳐서 차이의 한 원인이 될 수 있으니, 자체 점수는 상한으로만 본다. 다음에는 청크 단위 R@5를 같이 낸다.
- 제목 앵커링은 가정에 기대고 있다. 질문에 제목이 없으면 발동하지 않고 일반 검색으로 넘어간다. 앵커링의 공식 점수(3차)는 공개되지 않았다.
- 메타데이터에 의존한다. 상호가 바뀐 피심인처럼 메타데이터가 질문과 어긋나면 답도 어긋난다.
- 별표에만 있는 과징금 3건은 금액을 못 낸다. 표 파서가 있어야 풀린다.
- dense 질의 접두사를 검증하지 않았다. 질의에 영어 지시문 `"Represent this sentence for searching relevant passages: "`를 붙이는데, BGE-M3 모델 카드는 질의에 지시문을 붙이지 않아도 된다고 안내한다. 접두사 유무를 비교한 측정은 없다.

돌아보면 점수를 움직인 건 모델보다 구조였다. 검색은 하이브리드와 앵커링이, 생성은 LLM을 빼고 원문 사실을 조립한 코드가 맡았다. 가장 중요했던 결정은 0.9362를 버리고 0.785에서 다시 시작한 것이고, 공식 점수에서 생성 개선이 전이되지 않는 걸 본 뒤로는 검색에 걸었다.
