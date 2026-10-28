---
title: "질문 유형에 따라 검색 경로를 고르는 지식그래프 검색, 정말 나아졌나"
date: 2026-10-23 11:40:00 +0900
categories: [AI, RAG]
tags: [knowledge-graph, ontology, neo4j, graphrag, rag, evaluation, mcnemar, permutation-test]
math: true
image:
  path: /assets/img/posts/ontology-router/ui-doc-overview.png
  alt: 문서 코퍼스 지식그래프 탐색기
---

온톨로지·지식그래프 학습 문서 9편을 코퍼스로 삼아 Neo4j 지식그래프를 만들었다. 그 위에 검색 전략 6개, 질문 유형을 보고 전략을 고르는 규칙 라우터, 둘을 비교하는 평가 하네스를 직접 짰다. 여러 문서를 거쳐야 답이 나오는 질문에서 그래프가 벡터 검색보다 나은지 알고 싶었다.

| 항목 | 값 |
|---|---|
| 코퍼스 | 문서 9편, 147,960자 |
| 그래프 | Chunk 242 · 엔티티 333 · MENTIONS 2,399 · 도메인 관계 46 |
| 검색 전략 | vector · bm25 · graph · local · hybrid · router |
| v1 골든셋 (코퍼스를 보며 쓴 세트) | router 0.842 > vector 0.684, McNemar p=0.250 |
| v3 held-out | vector 0.737 > router 0.684, McNemar p=1.000 |
| v3 MRR (router − vector) | −0.112, 순열검정 p=0.008 (라우터가 유의하게 나쁨). 집계기 Hit #0을 빼면 −0.086 (p=0.017) |
| 라우터 분류 정확도 | held-out 합산 32/39 (82.1%) |
| 전략 비교에 필요한 문항 수 | 60~120 (채점 문항은 19개) |

{% include embed/video.html src='/assets/img/posts/ontology-router/explainer.mp4' poster='/assets/img/posts/ontology-router/explainer-poster.jpg' title='W10 질문이 관계형으로 분류돼 local로 가고, 진입점 3개 절단 때문에 근거 청크가 후보에서 빠지는 과정' loop=true %}

## 1. 구성

![문서 코퍼스 그래프 탐색기](/assets/img/posts/ontology-router/ui-doc-overview.png)
_실선은 원문 인용이 붙은 관계, 점선은 같은 청크에 함께 나왔다는 이유로 이은 추론 관계_

| 단계 | 파일 | 내용 |
|---|---|---|
| 청킹 | `chunking.py` | 목표 900자, 최대 1600, 겹침 120. 코드블록은 쪼개지 않고 절 경로를 `Chunk.section`에 저장 |
| 추출 | `extract.py`, `seed.py` | 표·용어집·코드스팬·정규식만 쓴다. LLM 없음 |
| 임베딩 | `embed.py` | fastembed로 `intfloat/multilingual-e5-large`(1024차원). E5 계열이라 `query: `/`passage: ` 접두사를 강제 |
| 적재 | `ingest.py` | 약 3초. 자동추출 → 통제어휘 → 시드 → 용어집 → 별칭 병합 순서 |
| 검색 | `retrieve.py`, `router.py`, `aggregate.py` | 6전략, 규칙 라우터, 스키마 기반 `count()` 집계 |
| 평가 | `eval/run_eval.py` | 검색 단위 지표 4개 + 유의성 검정 |
| 화면 | `web/`, `api.py` | React + react-force-graph-2d, FastAPI |

평가 단위는 검색으로 잡았다. 답변 정확도로 재면 검색 품질과 생성 품질이 섞인다. 애초에 궁금한 건 어느 검색이 이기느냐였고, 검색만 재면 LLM API 키도 필요 없었다.

## 2. 온톨로지: 골든셋에서 역산한 스키마

스키마보다 경쟁 질문 22개(v1 골든셋)를 먼저 써서 동결했고, 클래스와 관계는 그 질문에 답하는 데 필요한 것만 만들었다. 구성은 서술형 5, 관계형 5, 다중홉 5, 집계형 4, 답이 없는 부정 3문항이다. 관계형·다중홉·집계형이 14/22로 64%라 그래프를 들일 근거는 된다고 봤다. 다만 그래프를 쓰고 싶어서 관계형 질문을 많이 만들었을 수도 있으니, 그래프가 질 만한 서술형과 부정 문항을 일부러 섞었다.

### 클래스 9종을 7종으로

| 합치거나 바꾼 것 | 판단 근거 |
|---|---|
| `Tool`·`Package`·`Plugin`·`Language` → `Artifact` | 관계 표면이 같다. Neo4j, langchain-neo4j, APOC, Cypher 25가 모두 `LICENSED_AS`/`REPLACED_BY`/`DEPENDS_ON`을 가진다. 종류 차이는 `kind` 속성으로 |
| `Venue` → `Paper.venue` 속성 | 학회가 2개뿐이고 나가는 관계가 없다 |
| `Benchmark` → `Resource{kind:benchmark}` | 강의·도서와 같은 "참조할 외부 자산" 층위 |
| `License`는 클래스로 유지 | 집계 문항이 라이선스로 묶어서 센다. 속성이면 `WHERE`로 거를 수는 있어도 `-[:LICENSED_AS]->(l)`로 그룹핑할 수 없다 |
| `ApiSymbol`은 클래스로 유지 | `add_graph_documents`는 langchain-neo4j의 일부이면서 APOC을 요구한다. 독립 관계를 갖는다 |

기준은 하나였다. 스스로 관계를 갖는 건 클래스로, 단순 값은 속성으로 뒀다. 그렇게 남은 게 도메인 층 6종(`Artifact`, `License`, `Concept`, `ApiSymbol`, `Paper`, `Resource`)과 렉시컬 층 `Chunk`다.

관계는 11종이고 어느 관계든 최소 한 문항에는 대응한다. `RELATED_TO`·`HAS` 같은 이름은 쓰지 않았는데, 한 번 허용하면 애매한 관계가 전부 그리로 몰리기 때문이다. 방향도 고정해서 `REPLACED_BY`는 낡은 것에서 새 것으로(Kùzu → LadybugDB), `CONSTRAINS`는 제약하는 쪽에서 받는 쪽으로(Cypher 25 → apoc 함수) 간다.

두 층 사이는 `MENTIONS`로 이었다. 벡터 검색은 청크로 들어오고 그래프 확장은 엔티티를 따라 걷기 때문에, 뒤에 나올 `local` 전략은 이 다리를 지나간다.

### 관계마다 원문 인용

관계 46개 중 자동 추출된 20개는 전부 `LICENSED_AS`이고, 나머지 26개는 `seed.py`에 손으로 넣었다. 이 코퍼스에서는 화살표 `→`가 URL에도 흐름도에도 대체 관계에도 쓰이고, `대체`·`후계` 같은 키워드는 부정문에도 나온다. 자동으로 뽑으면 오탐이 쏟아지니 판단이 필요한 관계는 시드로 돌리고, 그 대신 시드마다 출처 파일과 원문 문장을 붙이게 했다.

```python
# (type, start_label, start_name, end_label, end_name, props, src_file, src_quote)
SEED_EDGES: list[tuple] = [
    ("REPLACED_BY", "Artifact", "Kùzu", "Artifact", "LadybugDB",
     dict(reason="Apple 인수 후 아카이브", at="2025-10-10"),
     "의사결정_치트시트.md", "후계 포크**: **LadybugDB** (MIT, 활발"),

    ("REPLACED_BY", "Artifact", "langchain-experimental", "Artifact", "langchain-neo4j",
     dict(reason="공식 sunset (이슈 #87)", at="2026-05-22"),
     "온톨로지_지식그래프_구축_실전가이드.md", "`LLMGraphTransformer`는 `langchain-neo4j`로 이전됐습니다"),
    # ... 생략
]
```

`verify_seed.py`는 인용 41건이 실제 파일에 그 문장 그대로 있는지 전부 확인한다. 하지만 이걸로 잡히는 건 지어낸 인용뿐이고, 진짜 문장에 엉뚱한 관계를 붙인 실수는 그대로 통과한다.

화면에서도 두 종류는 섞지 않았다. API가 관계를 내려줄 때 링크마다 `derived` 플래그를 붙이면 프론트가 그걸 보고 실선과 점선을 나눈다.

```python
links[key] = dict(source=r["a"]["id"], target=r["b"]["id"],
                  label=r["rel"], srcQuote=r.get("srcQuote") or "",
                  weight=r.get("w") or 1,
                  derived=r["rel"] == "CO_MENTION")
```

공동언급 층은 같은 청크에 함께 나온 엔티티 쌍을 세서 만든다. 횟수 하한을 1로 두면 7,087개, 3이면 1,539개, 5면 643개가 나온다. 이걸 주장된 관계 46개와 같은 선으로 그리면 46개짜리 그래프가 수천 개짜리처럼 보인다.

![근거 패널](/assets/img/posts/ontology-router/ui-doc-evidence.png)
_MIT 라이선스 노드를 고른 화면. 주장된 관계마다 어느 문서 어느 문장에서 왔는지 그대로 나온다_

## 3. 검색 6전략

| 전략 | 방식 | 역할 |
|---|---|---|
| `vector` | `db.index.vector.queryNodes` 코사인 | dense 기준선 |
| `bm25` | Neo4j 전문 인덱스 | lexical 대조군 |
| `graph` | 질문에서 렉시콘 매칭 → 관계 2홉 확장 → 그 엔티티를 언급한 청크 | 순수 구조. LLM 없는 엔티티 링킹의 한계 진단용 |
| `local` | 벡터로 진입 청크 → 언급 엔티티 → 관계 확장 → 확장 청크, RRF 융합 | MS GraphRAG Local Search 패턴 |
| `hybrid` | `vector` + `graph`를 RRF(k0=60)로 | |
| `router` | 질문 유형을 분류해 위 전략이나 Cypher 집계로 보냄 | |

기준선인 벡터 검색은 이게 전부다.

```python
def vector_search(session, question: str, k: int = TOP_K) -> list[Hit]:
    from .embed import embed_query
    vec = embed_query(question)
    rows = session.run(
        f"CALL db.index.vector.queryNodes('{VECTOR_INDEX}', $k, $v) "
        "YIELD node AS c, score "
        "RETURN c.id AS id, c.file AS file, c.section AS section, "
        "       c.text AS text, score",
        k=k, v=vec,
    ).data()
    return [Hit(r["id"], r["file"], r["section"], r["text"], r["score"], "vector")
            for r in rows]
```

`graph`는 질문 문자열에서 엔티티 이름이나 별칭(3자 이상)을 긴 것부터 찾는다. 그런데 골든셋 쪽에서 질문에 원문 표현을 12자 이상 그대로 쓰지 못하게 막아 뒀다. 벡터가 글자 겹침으로 이기지 못하게 하려던 규칙이었는데, 이 때문에 v1 22문항 중 11개에서 엔티티를 하나도 못 찾았고 다중홉 5개는 전부 실패했다. 실제 시스템이라면 LLM 엔티티 링킹이나 Text2Cypher가 채울 자리다. 그걸 LLM 없이 메워 보려고 만든 게 `local`이다.

```python
def local_search(session, question: str, k: int = TOP_K, hops: int = 2,
                 lex=None) -> list[Hit]:
    seeds = vector_search(session, question, max(3, k // 2 + 1))
    if not seeds:
        return []
    seed_ids = [h.chunk_id for h in seeds]
    seed_rank = {h.chunk_id: i for i, h in enumerate(seeds)}

    rows = session.run(
        """
        // ① 진입 청크가 언급한 엔티티
        MATCH (sc:Chunk)-[:MENTIONS]->(e)
        WHERE sc.id IN $seeds
        // ② 관계로 확장 (MENTIONS 는 렉시컬 다리라 제외)
        CALL (e) {
            WITH e
            MATCH p = (e)-[r*0..2]-(nb)
            WHERE NOT nb:Chunk
              AND all(x IN r WHERE type(x) <> 'MENTIONS')
              AND length(p) <= $hops
            RETURN DISTINCT nb, length(p) AS dist
        }
        // ③ 확장된 엔티티가 언급된 청크
        MATCH (c:Chunk)-[m:MENTIONS]->(nb)
        WITH c, nb, dist, m.count AS cnt
        WITH c,
             // 진입점에서 가까운 엔티티일수록, 많이 언급될수록 높은 점수
             sum((1.0 / (1 + dist)) * log(1 + cnt)) AS gscore,
             collect(DISTINCT CASE WHEN dist > 0 THEN nb.name END)[..4] AS via
        RETURN c.id AS id, c.file AS file, c.section AS section,
               c.text AS text, gscore, via
        ORDER BY gscore DESC LIMIT $lim
        """,
        seeds=seed_ids, hops=hops, lim=k * 4,
    ).data()

    # ④ RRF 로 진입점(벡터 순위)과 확장(구조 점수)을 융합
    k0 = 60
    fused: dict[str, tuple[float, Hit]] = {}
    for cid, i in seed_rank.items():
        h = seeds[i]
        fused[cid] = (1.0 / (k0 + i + 1), h)
    for rank, r in enumerate(rows, start=1):
        add = 1.0 / (k0 + rank)
        # ... 이미 있으면 점수를 더하고, 없으면 확장 청크로 추가 (생략)
    out = sorted(fused.values(), key=lambda t: -t[0])[:k]
    return [Hit(h.chunk_id, h.file, h.section, h.text, sc, h.via) for sc, h in out]
```

k=5일 때 진입 청크는 3개다. 벡터 점수와 그래프 점수는 척도가 달라 점수 대신 순위만 쓰는 RRF로 합쳤다. 확장 청크 점수 $\sum \frac{1}{1+d}\log(1+c)$는 진입 엔티티와의 거리 $d$가 가까울수록, 언급 횟수 $c$가 많을수록 커진다.

## 4. 규칙 라우터

v1 집계형 문항에서는 `local`(0.250)이 `vector`(0.500)에 졌다. "몇 개인가"는 애초에 Cypher `count()`로 풀어야 하는 문제였다. 골든셋 14문항을 손으로 쓴 Cypher로 재현해 보면(`check_graph.py`) 14/14이고, 여기에는 집계형 4문항도 들어 있다. 그래서 질문 유형에 따라 도구를 바꾸는 라우터를 만들었다. LangGraph 조건부 엣지가 하는 일을 정규식 규칙으로 대신하는 셈이다.

```python
def classify(question: str) -> tuple[str, dict]:
    """질문 → (유형, 신호). 규칙 기반, LLM 불필요.

    순서가 중요하다. 아래로 갈수록 약한 신호이므로 위에서 먼저 끊는다.
    """
    sig = signals(question)

    # ① 수량 의문이면 집계형. 가장 확실한 신호
    if sig["count"]:
        return "집계형", sig

    # ② 지시어 + 위치·식별 의문 → 다중홉
    if sig["anaphora"] and sig["locate"]:
        return "다중홉", sig

    # ③ 관형절·치환 연쇄 → 다중홉 (설명 요구가 더 강하면 서술형에 양보)
    if sig["chain"] and sig["chain"] >= sig["explain"]:
        return "다중홉", sig

    # ④ 설명 요구 → 서술형
    if sig["explain"]:
        return "서술형", sig

    # ⑤ 허가 여부·단일 사실 조회 → 관계형
    if sig["permit"] or sig["fact"]:
        return "관계형", sig

    return "서술형", sig      # 기본값은 벡터가 강한 서술형
```

신호는 전부 한국어 문형 정규식으로 잡는다. 수량 의문(`몇\s*(개|편|건|...)`), 지시어(`그`, `해당`)와 위치 의문(`어디`, `어느 `), 관형절·치환 연쇄(`대신`, `대체`, `후계`), 설명 요구(`왜`, `차이`, `이유`), 허가 여부(`써도`, `상업`), 사실 요구(`뭐야`, `알려줘`, `하나만`) 순이다. `개수`라는 명사는 단독으로는 세지 않고 `얼마`·`몇`과 붙을 때만 센다. "표시 개수만 줄이는"처럼 수량을 묻지 않는 문장에도 나오기 때문이다.

유형별로 어느 전략을 쓸지는 v1 측정값을 보고 정했다.

```python
# 측정값(k=5, answer_support)에 근거한 매핑:
#   서술형  vector 1.000 / local 1.000   → vector (동률이면 단순하고 빠른 쪽)
#   관계형  vector 0.800 / local 0.800   → local  (동률, 경로 설명이 붙는 쪽)
#   다중홉  vector 0.400 / local 0.600   → local
#   집계형  검색으로는 최고 0.500        → cypher (구조 집계)
ROUTE = {
    "집계형": "cypher",
    "다중홉": "local",
    "관계형": "local",
    "서술형": "vector",
}
```

집계는 `aggregate.py`가 맡는다. 질문 토큰을 스키마 어휘(레이블, `kind` 값)에 매칭해 `MATCH ... WHERE ... count()`를 조립하는 방식이다. 문항마다 쿼리를 박아 두면 골든셋에 답을 심는 꼴이 되니 어휘 표만 두었다. 집계가 안 되면 `None`을 돌려주고, 라우터는 `local`로 폴백한다.

![검색과 라우터 판정](/assets/img/posts/ontology-router/ui-doc-search.png)
_질문을 넣으면 라우터가 어떤 유형으로 읽었는지와 근거 신호를 같이 보여준다_

## 5. 처음 결과, 그리고 순환

아래는 v1 22문항을 k=5로 돌린 결과다. 지표 `answer_support@k`는 정답의 `must_include` 토큰이 검색 문맥에 전부 있으면 1, 아니면 0이다. LLM이 이 문맥만 보고 답할 수 있었을지를 대신 재려고 잡은 지표다.

| 유형 | n | vector | bm25 | graph | local | hybrid | router |
|---|--:|--:|--:|--:|--:|--:|--:|
| 서술형 | 5 | 1.000 | 0.800 | 0.200 | 1.000 | 1.000 | 1.000 |
| 관계형 | 5 | 0.800 | 0.400 | 0.400 | 0.800 | 0.800 | 0.800 |
| 다중홉 | 5 | 0.400 | 0.000 | 0.000 | 0.600 | 0.400 | 0.600 |
| 집계형 | 4 | 0.500 | 0.250 | 0.000 | 0.250 | 0.500 | 1.000 |
| 전체 | 22 | 0.684 | 0.368 | 0.158 | 0.684 | 0.684 | 0.842 |

라우터가 0.842로 1위였고, 처음엔 그대로 "라우터가 이긴다"고 적었다. 그런데 이 숫자에는 문제가 세 겹 쌓여 있었다.

첫째는 순환이다. 코퍼스 9편은 내가 고친 문서다. 골든셋도 그 문서를 보면서 내가 썼고, 라우터 규칙은 그 22문항을 읽고 나서 골랐다. 규칙 하나하나는 고유명사나 문항 id를 쓰지 않은 일반 문형이어도, 어떤 규칙을 넣을지는 정답지를 보고 정했다. 물음표가 2개 이상이면 다중홉으로 보는 규칙은 v1에서 반례가 나와서 넣지 않았는데, 이렇게 걸러낸 것부터가 정답지를 본 결과다. 그러니 라우터 분류 정확도 22/22(100%)는 상한값 정도로만 읽어야 했다.

둘째는 채점 경로였다. `cypher_search()`는 집계 결과 문장을 Hit #0으로 끼워 넣는다.

```python
head = Hit(chunk_id="agg", file="(cypher 집계)", section=res["label"],
           text=aggregate.as_text(res, question), score=1.0,
           via=f"cypher:count={res['cnt']}")
# ... 집계된 엔티티를 언급한 청크를 근거로 붙임 (생략)
return [head] + ev
```

다른 전략과 같은 척도로 채점하려고 이렇게 감쌌다. 그러자 집계기가 쓴 답 문장이 "검색된 문맥"으로 채점기에 들어갔고, 집계기가 쓴 답을 채점기가 읽고 통과시키는 꼴이 됐다. v1 집계형 4문항은 모두 이 경로로만 통과했고, 코퍼스만 놓고 보면 0/4였다. 이 문장을 빼고 다시 재면 router는 0.632로 vector 0.684보다 낮다. v1의 라우터 우위는 전부 여기서 나왔다.

셋째는 맨숫자 토큰이다. 채점이 부분문자열 매칭이어서 `must_include`의 `'5'`가 "Opus 5"의 5에도 걸린다. held-out 세트 W18에서 집계기가 7개라고 틀리게 답했는데도 1점을 받은 게 이 때문이다.

세 문제 모두 골든셋 검증기에서는 "문제 없음"으로 통과하고 있었다. 그 뒤 `validate_golden_set.py`에 검사 10~12번을 넣어 `must_include` 토큰이 코퍼스에 없는 경우, 맨숫자 토큰, hops≥2인데 근거가 한 청크에 몰린 경우를 잡게 했다. 동결된 세트에 돌리면 v1은 경고 4건, v3는 10건이 나온다.

## 6. held-out 세트 구성

held-out은 두 번 만들었다.

v2는 라우팅 전용 17문항이다. 문항을 먼저 쓰고 동결한 다음, 규칙은 건드리지 않은 채 한 번만 쟀고 계열 일치는 16/17(94.1%)이었다. 원래는 v2에 답을 채워 검색 평가용으로도 올리려 했다. 그런데 라우팅만 재려던 세트라 답을 확인하지 않고 질문부터 썼던 터라, 나중에 확인해 보니 부정 2문항을 뺀 15개 중 코퍼스에 답이 있는 문항이 5개뿐이었다. 그래서 포기했다.

검색용 v3 22문항은 v1과 같은 절차로 처음부터 다시 만들었다.

1. 코퍼스 9편에서 한 곳에 명시된 사실과 원문 인용을 모으고 원문과 다시 대조했다(258개, 중복 제거 후 180개).
2. v1이 이미 쓴 근거를 뺐다.
3. 근거를 보고 답을 쓰고, 그 답에서 질문을 썼다. 질문부터 쓰지는 않았다.
4. 질문에 근거 표현을 12자 이상 그대로 쓰지 않았다.
5. v1이 다룬 주제(2층 그래프, 별칭 통합, Kùzu → LadybugDB 라이선스 등 17개)를 피했다.
6. 부정 3문항은 코퍼스 전체 grep 0건으로 부재를 확인했다.

v3를 동결한 뒤로는 이 세트를 보고 검색기나 라우터를 고치지 않기로 하고, 그 규칙을 세트 파일 머리에 적어 두었다. 고칠 일이 생기면 v4를 새로 만들어야 한다.

![v1과 v3의 전략별 answer_support](/assets/img/posts/ontology-router/v1-vs-v3-light.png){: .light }
![v1과 v3의 전략별 answer_support](/assets/img/posts/ontology-router/v1-vs-v3-dark.png){: .dark }
_held-out에서 router는 0.842에서 0.684로 내려가고 vector는 0.737로 1위가 된다_

| 유형 | n | vector | bm25 | graph | local | hybrid | router |
|---|--:|--:|--:|--:|--:|--:|--:|
| 서술형 | 5 | 0.400 | 0.200 | 0.000 | 0.400 | 0.400 | 0.400 |
| 관계형 | 5 | 0.800 | 0.600 | 0.000 | 0.400 | 0.800 | 0.600 |
| 다중홉 | 5 | 1.000 | 0.400 | 0.200 | 1.000 | 0.800 | 1.000 |
| 집계형 | 4 | 0.750 | 0.500 | 0.000 | 1.000 | 0.750 | 0.750 |
| 전체 | 22 | 0.737 | 0.421 | 0.053 | 0.684 | 0.684 | 0.684 |

라우터는 관계형에서 손해를 봤다. 라우터는 관계형을 `local`로 보내는데, held-out 관계형에서 `local`은 0.400, `vector`는 0.800이었다. v1에서는 둘이 0.800 동률이어서 "경로 설명이 붙는 쪽"이라는 이유로 `local`을 골랐고, 그 선택에 드는 비용은 v1에서 드러나지 않았다. W10이 대표적이다.

> 초안을 만들고 사람이 확인한 다음 내보내는 흐름에 쓰라고 적어둔 함수, 이름 하나만.

정답은 `interrupt()`다. vector는 근거 청크를 찾아 통과했지만, 라우터는 이 질문을 `local`로 보내는 바람에 놓쳤다.

다중홉 행은 비교에 쓰지 못한다. v3 다중홉 5문항 중 4개는 근거 두 개가 한 청크에 들어 있어서 벡터 한 번으로 다 잡혔다. v1에서는 5/5가 서로 다른 청크, 2~4개 파일에 걸쳐 있었다. 사실 수집을 파일당 에이전트 하나로 돌린 탓에 파일을 넘나드는 연쇄가 후보로 올라올 수 없었다. 결국 "다중홉에서 그래프가 이긴다"는 반증되지도 확인되지도 않은 채 미검증으로 남았다.

집계기는 held-out에서 답을 낸 2문항을 모두 틀렸다. W17은 정답이 5인데 61, W18은 정답이 5인데 7이었다. 집계기는 질문을 `(label, kind, property)`로만 옮길 수 있다 보니, "Claude 모델"의 Claude나 "개념 노트가 적어둔 목록"의 문서 범위처럼 그 틀에 들어가지 않는 제약은 버리고 상위집합을 센다. 개수를 틀린 것보다 기권하지 않았다는 쪽이 더 위험하다.

라우터 분류도 v3에서 같이 쟀다. v3로는 라우터를 튜닝하지 않았으니 이게 두 번째 held-out이 된다.

| 세트 | 계열 일치 |
|---|--:|
| v1 (규칙을 고른 세트) | 22/22 (100%), 폐기 |
| v2 (라우팅 전용) | 16/17 (94.1%) |
| v3 (검색용, 튜닝 안 함) | 16/22 (72.7%) |
| v2 + v3 합산 | 32/39 (82.1%) |

오류 7건 중 4건은 관계형을 서술형으로 읽었다. "어느 학회에 붙었나", "몇이야", "뭐라고 부르고", "무슨 라이선스로"처럼 구어체로 들어온 사실 질문을 설명 요구로 분류해 버린다. 규칙 하나만 손보면 4건이 잡힐 가능성이 높지만, v2와 v3를 둘 다 써 버려서 v4 전에는 고치지 않았다.

## 7. 유의성 검정

v3의 vector 0.737과 router 0.684는 문항 수로 따지면 몇 개 차이일까. 부정 3문항은 `must_include`가 없어 `answer_support`가 null이므로 채점 문항은 19개다. 0.737은 14/19, 0.684는 13/19라서 1문항 차이다. v1의 0.842와 0.684도 16/19 대 13/19로 3문항 차이였다. 그런데도 하네스는 오랫동안 평균만 출력했고, 나는 그 대소를 그대로 순위로 옮겨 적었다.

### McNemar 정확검정

두 전략이 같은 문항을 각각 맞혔는지는 짝지은 이진 데이터다. 둘 다 맞히거나 둘 다 틀린 문항은 차이를 전혀 알려 주지 않으니 한쪽만 맞힌 불일치 쌍만 본다. 두 전략이 같다는 귀무가설 아래서 불일치 $n = n_{10} + n_{01}$개는 반반씩 갈려야 하므로 $n_{10} \sim \mathrm{Binomial}(n, 1/2)$이고, 양측 p값은 다음과 같다.

$$
p = \min\left(1,\; 2 \sum_{j=0}^{\min(n_{10},\, n_{01})} \binom{n}{j} \frac{1}{2^n}\right)
$$

표본이 작으니 카이제곱 근사 대신 이항분포를 직접 계산했다.

```python
def mcnemar_exact(a: list, b: list) -> tuple[int, int, float]:
    """이진 지표용 짝지은 정확검정. (a만 맞은 수, b만 맞은 수, 양측 p).

    불일치 쌍만 쓴다. 둘 다 맞거나 둘 다 틀린 문항은 정보가 없다.
    표본이 작아 근사(카이제곱)를 쓰면 안 되므로 이항분포를 직접 계산한다.
    """
    pairs = [(x, y) for x, y in zip(a, b) if x is not None and y is not None]
    n10 = sum(1 for x, y in pairs if x == 1 and y == 0)
    n01 = sum(1 for x, y in pairs if x == 0 and y == 1)
    n = n10 + n01
    if n == 0:
        return n10, n01, 1.0
    k = min(n10, n01)
    p = sum(math.comb(n, j) for j in range(k + 1)) / 2 ** n * 2
    return n10, n01, min(1.0, p)
```

`min(1.0, p)` 캡은 불일치가 양쪽으로 똑같이 갈린 경우를 위한 것이다. 이때 이항 합에 2를 곱하면 1을 넘을 수 있다.

v1은 불일치 3개가 모두 라우터 쪽이어서 $p = 2 \times \binom{3}{0}/2^3 = 0.250$이다. v3는 불일치가 1개뿐이라 $p = 2 \times 1/2 = 1.000$이다. 불일치가 전부 한 방향으로 몰리더라도 5개면 $2/32 = 0.0625$이고, 6개는 돼야 $2/64 = 0.03125$로 처음 0.05 밑으로 내려간다. 이 경계는 `tests/test_significance.py`에 테스트로 박아 두었다.

### MRR 순열검정

0/1 지표는 버리는 정보가 많다. 근거를 1위에 올리든 5위에 올리든 똑같이 1점이다. 그래서 순위 지표인 MRR로 같은 비교를 다시 했다.

$$
\mathrm{MRR} = \frac{1}{|Q|} \sum_{q \in Q} \frac{1}{\mathrm{rank}_q}
$$

$\mathrm{rank}_q$는 상위 k개 안에서 골든셋 근거 인용이 처음 나온 청크의 순위이고, 못 찾으면 그 문항은 0이다.

MRR은 연속값이라 McNemar를 쓸 수 없고, 19문항으로는 분포를 가정하기도 어려워서 짝지은 순열검정을 택했다. 두 전략이 같다면 문항별 차이 $d_i$의 부호는 어느 쪽이든 상관없어야 한다. 그래서 부호를 무작위로 뒤집고 평균을 다시 내는 작업을 20,000번 반복해 귀무분포를 만든다.

```python
def perm_paired(a: list, b: list, iters: int = 20000, seed: int = 3) -> tuple[float, float]:
    """연속 지표용 짝지은 순열검정. (평균차, 양측 p).

    문항별 차이의 부호를 무작위로 뒤집어 귀무분포를 만든다. 분포 가정이 없고
    n 이 작아도 쓸 수 있다. 시드를 박아 재현 가능하게 둔다.
    """
    d = [(x or 0.0) - (y or 0.0) for x, y in zip(a, b)]
    if not d:
        return 0.0, 1.0
    obs = statistics.fmean(d)
    rnd = random.Random(seed)
    hit = 0
    for _ in range(iters):
        m = statistics.fmean([v if rnd.random() < 0.5 else -v for v in d])
        if abs(m) >= abs(obs) - 1e-12:
            hit += 1
    return obs, hit / iters
```

`significance_block()`은 각 전략을 `vector`와 짝지어 `answer_support`는 McNemar로, MRR과 `evidence_recall`은 순열검정으로 비교한 뒤 평균 옆에 p값을 함께 찍는다.

![지표별 router − vector 차이와 p값](/assets/img/posts/ontology-router/router-vs-vector-light.png){: .light }
![지표별 router − vector 차이와 p값](/assets/img/posts/ontology-router/router-vs-vector-dark.png){: .dark }
_주황 막대만 p<0.05. held-out MRR에서 라우터가 벡터보다 0.112 낮다_

| router − vector | v1 | v3 (held-out) |
|---|--:|--:|
| answer_support | +0.158 (p=0.250) | −0.053 (p=1.000) |
| evidence_recall | +0.079 (p=0.500) | −0.184 (p=0.127) |
| MRR | +0.022 (p=0.731) | −0.112 (p=0.008) |

v3 전체를 `vector` 기준으로 보면 이렇다.

| 전략 | support 차 | 불일치 | p | MRR 차 | p | recall 차 | p |
|---|--:|--:|--:|--:|--:|--:|--:|
| bm25 | −0.316 | 8 | 0.070 | −0.187 | 0.131 | −0.395 | 0.009 |
| graph | −0.684 | 13 | <0.001 | −0.395 | <0.001 | −0.711 | <0.001 |
| local | −0.053 | 3 | 1.000 | −0.121 | 0.006 | −0.237 | 0.062 |
| hybrid | −0.053 | 1 | 1.000 | −0.039 | 0.491 | +0.000 | 1.000 |
| router | −0.053 | 1 | 1.000 | −0.112 | 0.008 | −0.184 | 0.127 |

held-out에서 유의하다고 말할 수 있는 건 두 가지이고, 그 밖에 bm25의 `evidence_recall`도 vector보다 낮다(p=0.009).

하나는 순수 구조 검색 `graph`가 나머지 다섯 전략 모두에 유의하게 진다는 것이다(`answer_support` 기준 bm25 대비 p=0.016, 나머지 p<0.001). 이 수치는 위의 vector 기준 표 밖에서 전략 전체 쌍을 비교해 얻었다. 반유출 규칙 때문에 렉시콘이 진입점을 못 잡은 결과이고, `local`을 따로 만든 이유이기도 하다.

다른 하나는 근거를 몇 번째에 올렸는지로 보면 `router`와 `local`이 `vector`보다 유의하게 나쁘다는 것이다(p=0.008, p=0.006). 답할 수 있는 문맥을 가져왔는지로는 구별되지 않던 차이가 순위에서는 잡혔다.

router의 MRR −0.112에는 채점 구조 탓도 일부 섞여 있다. 집계형으로 분류된 W17·W18에서는 집계기 문장이 Hit #0으로 1위를 차지하니 근거 청크가 한 칸씩 밀린다. 원자료로 다시 계산한 결과는 아래와 같다.

| router − vector MRR | 차 | p |
|---|--:|--:|
| 원본 (19문항) | −0.112 | 0.008 |
| 집계기 Hit #0을 빼고 순위를 한 칸씩 당김 | −0.086 | 0.017 |
| W17·W18을 빼고 17문항 | −0.067 | 0.031 |

방향은 그대로지만 차이 크기의 일부는 집계기 Hit #0이 1위를 차지하는 채점 구조에서 나온다.

나머지는 라우터가 관계형·다중홉(그리고 집계 실패 시 폴백)을 `local`로 보내면서 생긴다. 문항별 첫 근거 순위를 vector와 나란히 놓으면 원인이 두 갈래로 나뉜다.

| 원인 | 문항 | vector 순위 → router 순위 |
|---|---|---|
| RRF 밀어내기. 확장 청크가 RRF에서 앞에 끼어 근거 청크가 뒤로 밀림 | W02, W12, W16, W19 | 2→4, 3→4, 2→3, 3→5 |
| 시드 3개 절단. k=5에서 진입 청크가 3개뿐이라 vector 4위 근거가 진입점에서 빠짐 | W10, W13 | 4→못 찾음, 4→못 찾음 |

6절의 W10도 두 번째 경우에 해당한다. vector는 근거를 4위에 올렸지만 `local`은 상위 3개만 진입점으로 쓰기 때문에 그 청크는 처음부터 후보에 없었다. `router`와 `local`의 MRR이 나란히 나쁘게 나온 것도 이 때문이다.

다중비교 문제도 남아 있다. vector 기준 비교는 전략 5개 × 지표 3개로 15개다. 15개 전체에 Bonferroni(기준 0.0033)나 Holm을 적용하면 graph를 뺀 나머지는 local 0.006과 router 0.008까지 전부 탈락하고, MRR 5개만 묶어 Holm을 적용하면 둘 다 남는다. MRR은 `answer_support`가 유의하지 않게 나온 뒤에야 옮겨 간, 사후에 고른 지표이기도 하다. p=0.008은 그걸 감안하고 읽어야 한다.

## 8. 몇 문항이 있어야 했나

검정을 붙인 다음 표본 크기를 거꾸로 계산해 봤다. 불일치율(한쪽만 맞히는 문항 비율)과 우세 비율(불일치 중 한쪽이 이기는 비율)을 가정하고 McNemar 검정력을 3,000회 시뮬레이션했다(α=0.05).

![문항 수에 따른 McNemar 검정력](/assets/img/posts/ontology-router/power-light.png){: .light }
![문항 수에 따른 McNemar 검정력](/assets/img/posts/ontology-router/power-dark.png){: .dark }
_어느 가정에서도 20문항 근처의 검정력은 0.02~0.32다_

이진 지표(`answer_support`, McNemar) 기준이다. 시뮬레이션 코드는 남겨 두지 않아서 결과표만 있다.

| 불일치율 | 우세 비율 | 위 격자에서 검정력 80%를 처음 넘는 문항 수 |
|--:|--:|--:|
| 35% | 85% | 60 |
| 25% | 85% | 80 |
| 15% | 85% | 120 |
| 25% | 75% | 160 |

현실적인 목표는 60~120문항이고, 채점 문항 19개의 3~6배다. 이 계산은 측정 전에 했어야 했다.

v4를 크게 만들면 될 줄 알았는데, 다중홉에서는 그것도 통하지 않았다. 파일을 넘나드는 다중홉 문항을 만들려면 그래프에 서로 다른 파일의 사실을 잇는 경로가 있어야 한다. Neo4j에서 길이 2 경로를 직접 세어 보니 방향 경로 10개 중 양 끝이 공통 파일을 갖지 않는 건 0개였고, 무방향까지 넓혀도 허브 잡음을 빼면 1개였다. 관계 46개 중 20개가 `LICENSED_AS`인데 `License`는 나가는 간선이 없는 종점이라 경로를 만들지 못한다. 현재 관계를 부분표집해 외삽해 보니 후보 60개에 관계 약 356개, 120개에 약 504개가 필요했다. 문항을 더 쓴다고 풀릴 일이 아니었고, 막힌 곳은 그래프 크기였다.

그래서 관계를 규칙으로 뽑을 수 있는 코퍼스로 옮겨 두 번째 그래프를 만들었다. arXiv 논문 50편의 LaTeX 소스에서는 `\cite{...}`가 관계가 되고, 그 명령이 들어간 문장이 근거가 된다. 그래프는 논문 229편(수집 50 + 인용만 된 179)에 관계 634개이고, 관계는 전부 규칙으로 추출했으며 근거 문장이 붙어 있다. 다중홉 경로는 332개다. 수집한 50편끼리만 이으면 간선 14개에 경로가 0개라서, 인용만 된 논문을 그래프 전용 노드로 올렸다.

![인용망 그래프](/assets/img/posts/ontology-router/ui-citation-overview.png)
_같은 탐색기로 본 arXiv 인용망. 관계 유형은 CITES 381 · USES 157 · COMPARES 43 · CONTRADICTS 30 · SUPPORTS 23_

## 9. 한계와 다음 단계

- 원래 질문에는 답하지 못했다. "다중홉에서 그래프가 이긴다"는 v1에서는 순환 때문에, v3에서는 문항 결함 때문에 검증하지 못했다.
- "라우팅이 검색을 개선한다"는 held-out에서 오히려 반대로 나왔고, MRR은 vector보다 유의하게 나빴다(p=0.008). 다만 MRR은 사후에 고른 지표이고, 15개 비교 전체를 보정하면 유의하지 않으며, 집계기 Hit #0을 빼면 차이가 −0.086으로 줄어든다.
- 문서 그래프에 든 지식은 절반 이상을 손으로 넣었다. 관계 46개 중 26개가 수기이고, 내가 쓰지 않은 오픈소스 README 8편에 같은 추출기를 돌리면 관계가 0개 나온다. 이 추출기는 결국 이 코퍼스의 특정 표 모양에 맞춘 파서다.
- 찾아낸 결함을 근거로 검색기, 라우터, 집계기를 고친 건 하나도 없다. 관계형을 `local`로 보내는 판단, "어느 X" 문형, 집계기 기권 로직은 오염되지 않은 v4가 생긴 다음에 고칠 생각이다. 지금 고치면 다음 측정까지 같이 오염된다.
- v4 조건은 미리 정해 뒀다. 60~120문항, 다중홉 근거는 서로 다른 파일에, 개수는 `answer`와 직접 대조(`must_include` 토큰으로 재지 않음), 집계기 문장을 뺀 점수와 집계기 기권율을 같이 낸다.
- 인용망 그래프에는 아직 골든셋이 없다. 다중홉 후보가 332개라 이제는 60~120문항을 만들 수 있고, 원래 질문으로 돌아가려면 이 길로 가야 한다. 인용 태도(반박/지지/사용/비교) 분류는 held-out 정밀도가 gpt-4o로 68.2%, 규칙만으로는 32.5%여서 라벨만 믿지 않고 인용 문장을 늘 같이 보여준다.

돌아보면 이 프로젝트에서 평균 두 개를 나란히 놓고 큰 쪽을 승자로 적는 실수를 세 번 했다. v1의 "라우터 0.842가 이긴다", v1 다중홉의 "그래프 0.600 대 벡터 0.400", 그리고 결함 보고서를 처음 쓸 때 적은 "held-out에서 vector 0.737 > router 0.684이므로 라우터가 진다"가 그렇다. 마지막 건 1문항 차이에 p=1.000이었다. 방향은 살아남았지만 그 결론을 받쳐 줄 근거는 MRR 쪽에 있었고, 그마저 단서가 붙는다. 그래서 지금은 하네스가 평균을 낼 때 p값도 같이 내도록 바꿔 뒀다.
