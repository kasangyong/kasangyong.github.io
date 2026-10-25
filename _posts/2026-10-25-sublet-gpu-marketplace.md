---
title: "Sublet: 남의 GPU에서 돌린 추론을 TOPLOC 지문과 머클 커밋으로 검증하고 컨트랙트로 정산하기"
date: 2026-10-25 11:00:00 +0900
categories: [Blockchain, Distributed Systems]
tags: [gpu, toploc, eip-712, solidity, merkle-tree, fastapi, docker, cuda, llm-inference]
image:
  path: /assets/img/posts/sublet-gpu-marketplace/architecture-light.png
  alt: Sublet 잡 흐름과 권한 분리
---

유휴 GPU를 노드로 등록해 남의 LLM 추론 잡을 돌리고, 그 결과에 활성화 지문과 서명과 머클 루트를 붙여 스마트 컨트랙트가 정산하게 하는 플랫폼 Sublet을 만들었다. 다른 사람의 노트북이 명령 한 줄로 노드가 되어 내 잡을 GPU로 처리하는 데까지 실제로 돌렸다.

| 항목 | 값 |
|---|---|
| 구성 | coordinator, checker, attestor, provider-agent(상주 프로세스 4개) + 검증 CLI `sublet-verify` + 웹 콘솔 + Solidity 컨트랙트 3개와 라이브러리 1개 |
| 코드 | Python 11,216줄(`*/src`), Solidity 1,217줄 |
| 테스트 | pytest 테스트 함수 578개, forge 87건, Slither 0 findings |
| 전 구간 자동 실행 | 잡 제출 하나가 배정·격리 실행·증거 서명·TOPLOC 재검증·어테스트·`settle`까지 사람 손 없이 간다 (로컬 Anvil) |
| 판정 신뢰 구조 | 컨트랙트는 2-of-3 어테스터 서명을 요구하지만, 지금은 한 attestor 프로세스가 임계값 이상의 키를 혼자 들고 채운다(로컬 기본은 3개 전부). 실질적으로 checker 키 하나와 attestor 프로세스 하나를 믿는다 |
| TOPLOC 신호 | 정직한 재실행 0, 정밀도 격하 11~12, 프롬프트 변조 111, 모델 교체 128 (`exp_mismatches`) |
| 다른 사람 GPU | RTX 4050 Laptop에서 CPU 2.5 tok/s, GPU 약 44 tok/s |
| 미해결로 선언한 것 | R1 이종 백엔드 검증(정직한 실행도 최대 42), R4 잡당 온체인 비용(정상 경로 중앙값 합 338,678 gas, 배치로 묶어도 약 3.3%만 준다) |
| 기간 | 2026-08-24 설계 ~ 09-02, 커밋 83개 |

## 1. 주장하는 것과 주장하지 않는 것

기존 GPU 임대 플랫폼은 정산 근거가 플랫폼의 집계뿐이다. 공급자는 왜 이 금액인지, 수요자는 그 작업이 정말 수행됐는지를 스스로 확인할 수 없다. Sublet은 이걸 푸는 게 목표지만 "무신뢰(trustless) 검증"이라고는 주장하지 않는다. 설계 문서에서 가장 중요하게 둔 절이 보장 목록과 미보장 목록이다.

| 보장하려는 것 | 메커니즘 |
|---|---|
| 청구된 작업이 선언된 모델·설정으로 수행됐다 | TOPLOC 활성화 지문 |
| 청구된 작업량이 산출물과 맞는다 | 토큰 단위 과금 |
| 결과가 사후에 바뀌지 않았다 | 순서까지 커밋하는 머클 루트, 증거 해시를 온체인 판정에 묶음 |
| 사용자가 플랫폼에 묻지 않고 확인할 수 있다 | `sublet-verify` (코디네이터에 접속하지 않음) |
| 정산을 한 주체가 조작하지 못한다 | m-of-n 어테스터 서명 + 분쟁 창 |
| 부정이 손해다 | 담보 + 단계적 슬래싱 |

| 보장하지 않는 것 | 이유 |
|---|---|
| 공급자가 워크로드 내용을 못 본다 | 공급자는 자기 기계의 root이고 GPU는 평문으로 연산한다. GPU TEE 없이는 불가능해서 v1은 공개 모델, 비민감 입력만 받는다 |
| 청구 시간의 진실성 | 공급자 프로세스가 만드는 값이라 반증할 수 없다. 그래서 시간을 과금 기준에서 뺐다 |
| 학습 작업의 정합성 | Verde급 refereed delegation이 필요하다. v1은 추론만 |

원안은 "GPU 사용 시간을 검증 가능한 값으로 기록"이었다. 그런데 NVML 사용률이나 전력 로그도 결국 공급자 기계가 만드는 값이라 외부에서 반박할 방법이 없고, 정직한 공급자에게도 느린 척해서 시간을 늘릴 유인이 생긴다. 그래서 과금 단위를 결과 본문에서 다시 셀 수 있는 `prompt_tokens + completion_tokens`로 바꾸고 시간은 SLA 판정에만 남겼다. 같은 이유로 사용자가 올리는 것도 선언적 명세로 제한했다. 임의 코드는 격리도 어렵고 "정상 결과"의 정의가 없어서 지문을 걸 곳이 없다.

```json
{
  "runtime": "hf-inference@0.1.0",
  "model": "Qwen/Qwen2.5-0.5B-Instruct",
  "dtype": "bfloat16",
  "sampling": { "temperature_milli": 0, "max_tokens": 16, "top_p_milli": 1000, "seed": 42 },
  "verification": { "mode": "toploc", "redundancy": 0, "strict_determinism": false },
  "max_price_uusdc_per_1k_tokens": 200
}
```

위 표는 목표이고 지금 구현은 몇 칸이 비어 있다. 작업량 재계수는 산술 일관성 검사에 머물러 있고(7절), `sublet-verify`의 TOPLOC 단계는 SKIPPED이며 m-of-n은 배선이 한 프로세스로 닫혀 있다(2절). 컨트랙트에도 신뢰 가정과 어긋나는 함수가 둘 있다(8절).

## 2. 권한을 셋으로 나눈 잡 흐름

![잡 흐름과 권한 분리](/assets/img/posts/sublet-gpu-marketplace/architecture-light.png){: .light }
![잡 흐름과 권한 분리](/assets/img/posts/sublet-gpu-marketplace/architecture-dark.png){: .dark }
_checker와 attestor는 서로 직접 통신하지 않고 코디네이터의 `/work/*` 엔드포인트를 거친다_

| 컴포넌트 | 하는 일 | 하지 않는 일 |
|---|---|---|
| coordinator | 노드 레지스트리, 잡 큐, lease, 배정, 판정 보관 | 판정 |
| checker | 구조 검증, 같은 명세로 재실행해 TOPLOC 대조, 판정에 서명 | 집행 |
| attestor | checker 서명을 직접 복구해 대조, 어테스터 서명으로 온체인 제출, 분쟁 창 뒤 `settle` | 판정 변경, 지급액 결정 |
| 컨트랙트 | 에스크로, 담보, 슬래싱, 분쟁 | 판정 |

한 프로세스가 배정·판정·집행을 다 하면 "중앙 집계를 믿어야 한다"는 원래 문제로 돌아간다. 그래서 경계를 HTTP로 긋고 판정에는 checker 서명을 받는다. attestor는 코디네이터가 넘겨준 `checker_id`를 믿지 않고 서명에서 주소를 복구한다. 코디네이터가 저장된 판정을 고치면 서명이 깨져 체인까지 못 간다.

### m-of-n은 아직 컨트랙트에만 있다

로컬 배포는 어테스터 3명, 임계값 2로 설정한다. 그런데 어테스터끼리 서명을 모으는 경로가 없다. attestor 데몬은 자기 프로세스가 든 키 전부로 서명을 만든다. 임계값에 모자라면 그 판정을 올리지 않고 다음 주기에 다시 본다.

```python
        signatures = collect_signatures(verdict, self._cfg.keys, self._chain.deployment.domain)
        if len(signatures) < self._chain.deployment.attestor_threshold:
            log.error(
                "%s 서명이 부족하다 (%d/%d), 다른 어테스터의 서명이 필요하다. "
                "지금 구조에는 어테스터 간 서명 수집 경로가 없다.",
                # ... 인자 생략
            )
            return False
```

그래서 로컬에서는 한 프로세스가 임계값 이상의 키를 혼자 든다. `--local-keys`로 띄우면 로컬 어테스터 키 3개를 전부 받아 셋 다로 서명한다. 데몬은 기동할 때 그 상태를 "m-of-n이 1-of-1과 같아진다"고 경고한다. 게다가 판정을 올려 줄 checker 주소 목록(`checker_allowlist`)의 기본값이 비어 있고, 비어 있으면 누구의 판정이든 올린다. 정리하면 지금 실제로 믿는 대상은 checker 서명키 하나와 attestor 프로세스 하나다. 서명 수집 경로를 만들고 키를 서로 다른 주체에 나눠 줘야 표의 "한 주체가 조작하지 못한다"가 성립한다.

### 잡이 흐르는 순서

1. 프롬프트를 8개씩 샤드로 나눈다(`PROMPTS_PER_SHARD = 8`). 이 크기가 주인이 GPU를 되찾을 때 잃는 작업의 상한이다.
2. 스케줄러가 후보를 거른다. 오프체인 상태, 하트비트(45초), 주인의 GPU 사용률(40% 이상이면 제외), 동시 실행 수, 내놓은 VRAM, 그리고 온체인 `isActive`를 함께 본다.
3. lease를 발급한다. Redis `SET NX`와 DB 부분 유니크 인덱스 두 겹으로 이중 배정을 막는다. 이중 배정은 곧 이중 정산이기 때문이다.
4. 에이전트가 격리 컨테이너에서 추론하고, 지문을 만들고, 증거에 서명해 돌려준다.
5. checker가 코디네이터에서 일감을 받아 판정을 서명해 올리고, attestor가 `/work/pending-attestations`에서 가져가 체인에 올린 뒤 분쟁 창(로컬 배포 1시간)이 지나면 `settle`을 부른다.

온체인 `isActive`를 배정에서 보는 줄은 나중에 들어갔다. 처음엔 코디네이터가 체인을 보지 않아서 담보가 900으로 깎여 `Suspended`가 된 노드(최소 1,000)도 계속 잡을 받았다.

```python
if not candidate.onchain_active:
    # 담보가 최소치 아래이거나 위조로 제외된 노드다. 잡을 주면 슬래싱이
    # 실질적 제재가 되지 못한다.
    return "온체인 레지스트리에서 비활성 (담보 미달 또는 제외)"
```

### 격리 컨테이너

남의 PC에서 남의 잡을 돌리니 격리 설정은 한 함수에 모아 두고 여기만 보면 무엇을 막는지 읽히게 했다.

```python
config: dict[str, Any] = {
    "image": image,
    # --- 격리 ---------------------------------------------------------
    "network_mode": "none",  # egress 차단의 근거. allowlist보다 강하다.
    "read_only": True,  # rootfs 쓰기 차단
    "cap_drop": ["ALL"],
    "security_opt": ["no-new-privileges"],
    "volumes": volumes,  # 호스트 바인드 마운트 0개
    # /tmp: 읽기전용 rootfs에 임시 디렉터리가 없으면 torch가 못 돈다.
    "tmpfs": {"/tmp": f"rw,noexec,nosuid,size={limits.tmp_size_mib}m"},
    # --- 자원 상한 -----------------------------------------------------
    "mem_limit": f"{limits.memory_mib}m",
    "pids_limit": limits.pids,
    "detach": True,
}
```

설계 문서는 egress allowlist였는데 `network_mode=none`으로 올렸다. allowlist는 규칙을 잘못 쓰면 뚫리고, 네트워크 스택이 없으면 뚫을 규칙 자체가 없다. 대신 입력을 `docker cp`로 넣어야 하는데, `--read-only` 컨테이너의 tmpfs는 시작 시점에야 생겨서 시작 전 복사가 안 됐다. 그래서 `/work`를 잡별 named volume으로 바꿨다. 모델 볼륨은 읽기 전용이라 컨테이너 안에서 모델을 바꿀 수 없다. GPU 패스스루가 `/dev/nvidia*`를 노출하는 건 알려진 약점이고 gVisor 승격은 아직이다.

## 3. 서명되는 구조에 float을 넣지 않는다

증거 검증은 노드가 서명한 바이트를 checker와 `sublet-verify`가 1바이트도 다르지 않게 재구성할 수 있어야 성립한다. 그래서 정규 직렬화 단계에서 float을 아예 거부한다.

```python
def _check(value: Any, path: str) -> None:
    """서명 대상에 들어가면 안 되는 타입을 사전에 거른다."""
    if isinstance(value, bool) or value is None or isinstance(value, int | str):
        return
    if isinstance(value, float):
        raise CanonicalizationError(
            f"{path}: float은 서명 대상에 쓸 수 없다 (플랫폼 간 표현 차이가 서명 불일치를 만든다). "
            f"단위를 이름에 박은 정수를 쓸 것. 예: sm_util_ppm, power_mw"
        )
    # ... dict, list 재귀 생략
```

float의 텍스트 표현이 플랫폼마다 달라지면 서명이 어긋난다. 증상은 "정직한 노드의 서명이 거부됨"으로 나타난다. 슬래싱이 붙은 시스템에서는 억울한 처벌로 이어진다. 이 때문에 실수값은 전부 단위를 이름에 박은 정수다. 명세의 `temperature_milli`, 텔레메트리의 `sm_util_ppm`, `power_mw_mean`, TOPLOC 통계의 `mant_err_median_micro`가 그렇다.

증거 묶음에는 잡 ID, 샤드 번호, 노드 주소, 명세 해시, 런타임 이미지 digest, 결과 머클 루트, TOPLOC 지문, 토큰 수, 텔레메트리, nonce가 들어간다. 이 묶음의 keccak 해시(`evidence_hash`)를 EIP-712로 서명한다.

```python
EVIDENCE_TYPES: dict[str, list[dict[str, str]]] = {
    "Evidence": [
        {"name": "jobId", "type": "bytes32"},
        {"name": "shardIdx", "type": "uint32"},
        {"name": "nodeId", "type": "address"},
        {"name": "evidenceHash", "type": "bytes32"},
    ]
}
```

EIP-712를 쓴 건 같은 서명을 컨트랙트에서도 검증해야 해서다. 노드 신원은 OS 키스토어에 둔 개인키에서 파생한 이더리움 주소다. 이 주소가 곧 `node_id`이자 온체인 레지스트리 조회 키다. 오프체인 증거의 식별자와 온체인 키가 같아야 `sublet-verify`가 코디네이터 없이 대조할 수 있다.

`runtime_digest`에 처음엔 이미지 태그를 넣고 있었다. 태그는 움직이니 카탈로그와 대조하는 의미가 없어서 digest로 바꿨다. Python이 만든 서명을 Solidity가 복구하는지는 Phase 0부터 교차 언어 벡터로 고정했고 CI가 커밋된 벡터와 재생성 결과가 같은지까지 본다.

## 4. 머클 트리: 순서까지 커밋한다

결과 본문은 오프체인에 두고 루트만 증거에 넣는다. 처음엔 OpenZeppelin 방식으로 형제 쌍을 정렬해서 해싱했는데, 테스트가 문제를 잡았다. 정렬하면 형제끼리 자리를 바꿔도 같은 루트가 나온다. 그러면 노드가 프롬프트 0의 답을 프롬프트 1의 답으로 바꿔 내도 탐지되지 않는다. 결과의 순서는 곧 입력 프롬프트와의 대응이라 커밋 대상이어야 한다.

```python
_LEAF = b"\x00"
_NODE = b"\x01"

# ... _leaf_hash, _node_hash 생략

def _levels(leaves: list[bytes]) -> list[list[bytes]]:
    levels = [leaves]
    while len(levels[-1]) > 1:
        current = levels[-1]
        nxt: list[bytes] = []
        for i in range(0, len(current), 2):
            if i + 1 < len(current):
                nxt.append(_node_hash(current[i], current[i + 1]))
            else:
                # 홀수 잔여는 그대로 승격한다. 복제(duplicate-last)는 서로 다른
                # 리프 집합이 같은 루트를 갖는 경로를 만들므로 쓰지 않는다.
                nxt.append(current[i])
        levels.append(nxt)
    return levels
```

결정은 세 가지다.

- 쌍을 정렬하지 않는다. 대가로 검증 쪽에 `index`와 `total`이 필요한데, 둘 다 레코드 위치와 샤드 프롬프트 수라 검증자가 이미 안다.
- 리프에 `0x00`, 내부 노드에 `0x01`을 붙여 second-preimage를 막는다.
- 홀수 잔여는 복제하지 않고 그대로 올린다. 마지막 리프를 복제하면 `[a, b, c]`와 `[a, b, c, c]`가 같은 루트를 갖는다.

포함 증명 검증은 `index`와 `total`로 홀수 승격 레벨을 재현하고 증명에 남는 항목이 있으면 위조로 본다.

```python
    # ... 입력 범위 검사와 total == 1 분기 생략
    acc = _leaf_hash(record)
    idx, n, consumed = index, total, 0
    while n > 1:
        has_sibling = not (idx == n - 1 and n % 2 == 1)
        if has_sibling:
            if consumed >= len(proof):
                return False
            sibling = bytes.fromhex(proof[consumed].removeprefix("0x"))
            consumed += 1
            acc = _node_hash(acc, sibling) if idx % 2 == 0 else _node_hash(sibling, acc)
        idx //= 2
        n = (n + 1) // 2

    # 증명에 남는 항목이 있으면 위조 시도로 본다.
    return consumed == len(proof) and ("0x" + acc.hex()).lower() == root.lower()
```

머클 루트는 온체인에 따로 올리지 않는다. 루트가 들어간 증거 묶음의 해시가 판정 서명(`Verdict.evidenceHash`)에 들어가고, 그 판정이 `attestVerdict`로 체인에 기록된다. 결과 한 글자를 바꾸면 루트가 바뀌고 루트가 바뀌면 증거 해시가 온체인 값과 달라진다.

배치 정산(`attestVerdictBatch`)의 판정 머클도 컨트랙트에서 같은 규약(정렬 없음, 홀수 승격, `0x00`/`0x01` 접두사)으로 구현했다. 규약이 갈리면 오프체인에서 만든 증명이 온체인에서 통과하지 않는다.

## 5. TOPLOC: 결과 텍스트만으로는 모델을 증명할 수 없다

결과 텍스트가 그럴듯해도 그게 선언한 모델로 계산된 건지는 알 수 없다. 0.5B로 계산하고 1.5B라고 청구하거나, bf16이라고 해 놓고 다른 정밀도로 돌려도 텍스트만 봐서는 모른다. 출력 해시 비교도 쓸 수 없다. 같은 temperature 0이라도 배치 크기에 따라 수치 경로가 달라져서 정직한 노드끼리도 출력이 갈린다.

TOPLOC(Prime Intellect)은 추론 중간 활성화의 지역민감해시를 지문으로 남긴다. 32토큰당 258바이트다. 비트 일치를 요구하는 대신 지수부 불일치 수와 가수부 오차 통계로 판정해서 수치 경로 차이에 어느 정도 견딘다.

채택할 때 기대한 장점 하나는 검증 비용이었다. 설계 문서에는 TOPLOC 논문을 근거로 검증이 원 추론보다 최대 100배 빠르다고 적었고 그래서 전 잡 검증이 경제적으로 가능하다고 봤다. 지금 구현은 그 이점을 아직 실현하지 못했다. checker의 재검증 경로는 원 추론과 같은 `run_inference`를 불러 `model.generate`로 전체를 다시 생성한 뒤 그 활성화로 지문을 대조한다. 재검증 비용이 원 추론과 같은 수준이고 GPU 잡을 검증하려면 checker에도 GPU가 있어야 한다. 이게 6절의 R1 문제와 그대로 이어진다.

런타임은 vLLM 대신 transformers 기반 `hf-inference`로 직접 만들었다. 활성화를 forward hook으로 바로 잡을 수 있어서다. 캡처 지점은 마지막 정규화 층 출력이다. 모델 전체를 통과한 뒤라 모델 교체, 정밀도 격하, 프롬프트 변조가 전부 여기에 반영된다.

```python
TOPLOC_DECODE_BATCH = 32
TOPLOC_TOPK = 128

def _capture_activations(model: Any) -> tuple[list[Any], Any]:
    activations: list[Any] = []

    def hook(_module: Any, _inputs: Any, output: Any) -> None:
        import torch

        activations.append(output.detach()[0].to(torch.bfloat16).cpu())

    handle = model.model.norm.register_forward_hook(hook)
    return activations, handle
```

`toploc==0.0.6`은 bfloat16 활성화만 받는다. float32를 주면 "interpolate has hardcode prime"으로 거부해서 잡의 dtype과 무관하게 지문용 활성화는 bf16으로 바꾼다. 설계 문서가 참조한 `exp_intersections` 필드도 이 버전에는 없었다. 실제 노출되는 건 `exp_mismatches`, `mant_err_mean`, `mant_err_median` 셋이다. 문서를 믿고 짠 스키마는 실측 뒤에 고쳤다.

checker는 "선언대로 했다면 이런 활성화가 나와야 한다"를 계산한다. 노드가 실제로 무엇을 썼든 명세의 모델·dtype·프롬프트·샘플링으로 같은 런타임 이미지에서 재실행하고 그 활성화로 노드의 지문을 대조한다.

```python
    model_path = resolve_model_path(shard.model)
    _records, summary, activations = run_inference(shard, model_path)

    params = payload.get("params", {})
    results = toploc.verify_proofs_base64(
        activations,
        list(payload["proofs"]),
        int(params.get("decode_batching_size", TOPLOC_DECODE_BATCH)),
        int(params.get("topk", TOPLOC_TOPK)),
        skip_prefill=bool(params.get("skip_prefill", False)),
    )
```

부정을 직접 주입해 신호가 갈리는지 쟀다.

![TOPLOC 지문 대조](/assets/img/posts/sublet-gpu-marketplace/toploc-signal-light.png){: .light }
![TOPLOC 지문 대조](/assets/img/posts/sublet-gpu-marketplace/toploc-signal-dark.png){: .dark }
_점선이 판정 임계값. 파란 막대는 6절의 문제다_

| 경우 | `exp_mismatches` | `mant_err_median` |
|---|---|---|
| 정직한 재실행 | 0 | 0.000 |
| 정밀도 격하 (float16) | 12 | 4.000 |
| 정밀도 격하 (float32) | 11 | 4.000 |
| 모델 교체 (0.5B → 1.5B) | 128 | 센티넬(클램프) |
| 프롬프트 변조 | 111 | 21.000 |
| seed 변조 (greedy) | 0 | 0.000 |

seed 변조가 0인 건 정상이다. greedy 디코딩에서 seed는 출력에 영향이 없으니 애초에 부정으로 볼 이유가 없다. 이걸 구분하지 않으면 검증기가 정직한 노드를 잡는다.

정밀도 격하 행은 한 번 놓쳤던 부분이다. 런타임이 CPU에서 선언 dtype을 조용히 float32로 바꾸고 있었는데, 그러면 정밀도 격하를 주입해도 양쪽이 다 float32로 돌아서 TOPLOC이 차이를 못 본다. 부정 주입 테스트가 이걸 잡았다. 지금은 선언된 dtype을 그대로 쓰고 지원 안 되는 조합이면 시끄럽게 실패한다.

모델이 바뀌어 활성화 차원이 다르면 toploc이 가수부 오차로 `18446744073709551616`(2^64) 같은 센티넬을 준다. 그대로 저장하면 정수 필드가 넘치므로 1e6으로 클램프한다. 어느 임계보다도 크니 판정에는 영향이 없다.

### 판정: 가장 나쁜 조각을 따른다

임계값은 코드에 박지 않고 `config/verification_thresholds.yaml`에서 읽는다. 지금 값은 지수부 불일치가 3을 넘으면 SUSPECT, 8을 넘으면 FAIL이고, 가수부 오차 중앙값은 1.0과 3.0이다. 정직한 값 0과 가장 약한 부정 신호 11 사이에 뒀다. 파일이 없으면 기본값으로 넘어가지 않고 멈춘다.

```python
def judge_all(
    measurements: list[TopLocMeasurement], thresholds: ThresholdSet
) -> tuple[Judgement, str]:
    if not measurements:
        return Judgement.SUSPECT, "지문이 없어 검증할 수 없다"

    worst = Judgement.PASS
    worst_reason = ""
    order = {Judgement.PASS: 0, Judgement.SUSPECT: 1, Judgement.FAIL: 2}
    for idx, measurement in enumerate(measurements):
        verdict, reason = judge_measurement(measurement, thresholds)
        if order[verdict] > order[worst]:
            worst, worst_reason = verdict, f"지문 조각 {idx}: {reason}"
    return worst, worst_reason
```

평균을 내면 한 조각의 명백한 부정이 나머지 정상값에 희석되니 최악의 조각을 따른다. 판정에는 통과/실패 외에 원시 통계와 `thresholds_version`도 남긴다. 임계가 바뀌면 과거 판정을 새 값으로 재평가할 수 있어야 해서다.

checker가 가장 신경 쓰는 건 검증 불능과 부정을 구분하는 일이다. 일시적 장애를 부정으로 기록하면 정직한 노드가 담보를 잃는다.

| 상황 | 판정 |
|---|---|
| 저장소에 못 닿음, 컨테이너가 안 뜸 | 판정하지 않고 재시도 |
| 재시도 한도 초과 | SUSPECT (FAIL 아님) |
| 지문이 없음 | SUSPECT, 재시도해도 생기지 않는다 |
| 서명·머클·토큰 합계 등 구조 검증 실패 | FAIL, 자기모순인 증거는 재실행할 것이 없다 |

SUSPECT는 슬래싱하지 않고 온체인에도 올리지 않는다. 올리면 잡이 지급 0으로 확정되는데, 확인 안 된 의심으로 공급자에게 불이익을 줄 수는 없다.

에이전트도 지문 없는 증거에는 서명하지 않는다. GPU 런타임 이미지가 낡아 toploc이 없었던 적이 있는데, 잡은 정상 완료됐고 에이전트가 지문이 빈 증거를 서명해 냈다. "유효하지만 검증 불가능한" 증거가 통과했다. 지금은 그 경우를 `runtime_error`로 보고하고 테스트가 CPU와 GPU 두 이미지를 모두 확인한다.

## 6. R1: 정직한 실행이 부정보다 더 이상해 보였다

위 신호 분리는 같은 기계, 같은 백엔드에서 잰 값이다. 개인 PC의 잡다한 GPU가 전제인 시스템에서는 다른 하드웨어끼리의 정상 편차가 문제다. 캘리브레이션을 한 08-26에는 GPU가 개발기 한 대(RTX 4050 Laptop)뿐이어서 CPU/GPU 백엔드 차이를 이종 환경의 대리로 썼다. 하니스로 생성 런타임과 검증 런타임을 따로 지정해 정직한 실행만 30건 모았다(Qwen2.5-0.5B, bf16, greedy, max_tokens 16).

![백엔드별 정직한 실행의 지문 편차](/assets/img/posts/sublet-gpu-marketplace/calibration-light.png){: .light }
![백엔드별 정직한 실행의 지문 편차](/assets/img/posts/sublet-gpu-marketplace/calibration-dark.png){: .dark }
_원 크기는 같은 값의 개수. 백엔드가 다르면 프롬프트에 따라 1, 3, 42가 나온다_

| 생성 → 검증 | n | `exp_mismatches` 최대 | 평균 |
|---|---|---|---|
| cpu → cpu | 6 | 0 | 0.00 |
| gpu → gpu | 3 | 0 | 0.00 |
| gpu → cpu | 12 | 42 | 15.33 |
| cpu → gpu | 9 | 42 | 15.33 |

같은 백엔드면 0, 다르면 프롬프트별로 1, 3, 42가 나왔고 방향과 무관하게 재현됐다. 판정은 임계값 초과(`>`) 비교라 1과 3은 PASS이고 42는 FAIL 임계 8을 넘는다. 42가 나오는 프롬프트가 들어 있는 샤드는 최악 조각 규칙 때문에 통째로 FAIL을 받는다. 임계값을 42 위로 올리면 11~12인 정밀도 격하를 통째로 놓친다. 정직한 이종 실행이 실제 부정보다 더 크게 어긋나니 신호가 분리되지 않는다.

이게 실제로 살아 있는 결함이었다. checker 기본값이 `with_gpu: False`라서 노드가 무엇으로 돌렸든 CPU 런타임 하나로 검증했다. 그대로 담보를 켰다면 정직한 GPU 노드의 샤드 중 42가 나오는 프롬프트를 포함한 것들이 FAIL로 슬래싱 대상이 됐을 것이다.

임계값은 그대로 두고 배선을 고쳤다. checker가 증거의 `telemetry.gpu_name`을 보고 노드의 백엔드에 맞춰 재실행하고, 맞출 수 없으면 판정하지 않는다.

```python
        needs_gpu = bundle.telemetry.ran_on_gpu
        if needs_gpu and not self._cfg.with_gpu:
            # 지문 파라미터가 없을 때와 같은 부류다: 검증할 수 없는 것을 통과로도
            # 실패로도 만들지 않는다. SUSPECT 는 슬래싱하지 않는다 (§7.6).
            # ... 경고 로그 생략
            return judge_shard(bundle, structural, None, self._thresholds)

        shard = self._shard_spec(item)
        image = self._cfg.gpu_image if needs_gpu else self._cfg.cpu_image
        measurements = verify_toploc(
            shard,
            list(bundle.toploc_proofs),
            params,
            image,
            limits=ContainerLimits(memory_mib=6144, timeout_seconds=1800),
            with_gpu=needs_gpu,
        )
```

`judge_shard`에 측정값 `None`을 넘기면 SUSPECT가 된다.

### 슬래싱 위험이 지급 문제로 바뀌었다

이 수정은 정직한 GPU 노드가 슬래싱되는 길을 막았지만 다른 비용을 만들었다. 08-31에 다른 사람 노트북(GPU 노드)에 GPU 없는 checker를 붙여 판정 5건을 냈더니 PASS 2건, SUSPECT 3건이었고, SUSPECT 3건이 전부 GPU 잡이었다. 공급자는 GPU로 일하고 그 잡의 과금 단위는 0이다. 정산을 켜면 이쪽이 먼저 아프다. 5절에서 적었듯 지금의 재검증은 전체 재생성이라, 이 문제는 "GPU를 가진 checker가 필요하다"로 이어진다.

| 선택지 | 대가 |
|---|---|
| checker를 GPU 기계에 둔다 | 가장 단순하다. 플랫폼이 GPU 비용을 진다 |
| 중복 실행으로 노드끼리 교차 검증 | 확장되지만 "다른 GPU끼리 같은 백엔드인가"에 먼저 답해야 한다 |
| SUSPECT를 부분 지급 | 검증 없이 돈이 나간다. 이 프로젝트의 명제와 충돌한다 |

아직 고르지 않았다.

남은 질문은 "서로 다른 GPU는 같은 백엔드인가"다. RTX 4050과 A100이 서로에게 0에 가까우면 GPU끼리 교차 검증이 되고, 42에 가까우면 checker가 같은 GPU 모델까지 맞춰야 한다. 08-27에 RTX 4070 Laptop에서 잡을 돌리긴 했지만(11절) 그 기계로 교차 캘리브레이션을 하지는 않았다. 그래서 임계 파일의 `is_provisional: true`를 그대로 두고 checker가 기동할 때 경고하게 했다. 내리는 조건은 두 종류 이상의 GPU에서 정직 실행 표본 200회 이상, 위양성률 0.1% 미만으로 잡아 뒀다.

## 7. 과금: 토큰 단위와 에스크로 상한

### 상한을 절대 과소평가하지 않는다

사용자는 잡을 제출할 때 에스크로를 예치한다. 실제 토큰 수는 토크나이저를 돌려야 아는데, 코디네이터가 모델별 토크나이저를 들고 있으면 "선언적 명세만 다룬다"는 경계가 깨진다. 그래서 과소평가하지 않는 상한을 쓴다.

```python
CHAT_TEMPLATE_MARGIN_TOKENS = 64

def max_billable_units(prompts: list[str], max_tokens: int) -> int:
    if not prompts:
        return 0
    prompt_bound = sum(len(p.encode("utf-8")) for p in prompts)
    template_bound = len(prompts) * CHAT_TEMPLATE_MARGIN_TOKENS
    completion_bound = len(prompts) * max_tokens
    return prompt_bound + template_bound + completion_bound
```

바이트 수준 BPE(GPT-2, Qwen 계열)에서는 모든 토큰이 최소 1바이트로 복호화되니 UTF-8 바이트 수가 토큰 수의 상한이다. 문자 수로는 안 된다. 한글은 한 글자가 여러 토큰이 되기도 한다. 채팅 템플릿이 붙이는 토큰은 프롬프트당 64로 넉넉하게 잡았다. 실측값이 아닌 가정이라 런타임을 추가할 때 확인할 항목으로 적어 뒀다.

예치액은 올림, 지급액은 내림이다.

```python
def required_escrow(max_units: int, price_per_1k_units: int) -> int:
    if max_units <= 0 or price_per_1k_units <= 0:
        return 0
    return -(-max_units * price_per_1k_units // 1000)
```

컨트랙트의 지급 계산이 내림이라 예치를 내림으로 잡으면 1단위 모자라는 경우가 생기고 그때 예치가 거부된다. 과다 예치는 정산에서 환불되지만 과소 예치는 공급자가 일하고 못 받는 것이라 한쪽으로만 틀리게 했다. 라이브 E2E에서 지불 지갑 잔액이 정확히 56 uUSDC 줄어든 것으로 실제 예치를 확인했다(279단위 × 200 / 1000 = 55.8, 올림).

### 재계수는 아직 산술 일관성까지다

설계상 정산 대상은 checker가 결과 본문을 선언된 모델의 토크나이저로 재토크나이즈해서 다시 센 값이다. 지금 구현은 거기까지 가지 않았다. checker는 레코드 수가 프롬프트 수와 맞는지, 레코드별 토큰 수의 합이 묶음의 합계와 맞는지, 머클 루트가 본문에서 재계산되는지까지만 본다. PASS면 `billable_units`는 노드가 신고한 합계다.

```python
    # 정산 대상은 checker가 재계수한 값이다. 노드 청구값을 그대로 쓰지 않는다.
    billable = bundle.units.total if outcome is Outcome.PASS else 0
```

주석은 설계 의도를 말하는데 실제 값은 `bundle.units.total`이다. 합계만 999,999로 부풀린 청구는 구조 검증에서 FAIL로 잡힌다. 레코드별 값까지 일관되게 부풀리면 통과한다. 이 한계는 숨기지 않고 테스트로 고정해 뒀다.

```python
def test_known_limitation_uniformly_inflated_claim_passes() -> None:
    """레코드별 값까지 함께 부풀리면 통과한다. 알려진 한계를 테스트로 고정한다.

    이 테스트가 실패하도록 만드는 것이 Phase 2의 목표다. 그때 이 테스트는
    "재계수가 부풀림을 잡는다"로 뒤집혀야 한다.
    """
```

`UnitsConsistency.is_authoritative`도 항상 `False`를 돌려주고, `sublet-verify`는 "작업 단위 권위 재계수"를 SKIPPED로 보고한다. 지금 부풀림의 천장은 컨트랙트가 쥐고 있다. 다음 절의 `maxUnits` 커밋 때문에 어떤 청구도 예치 시점의 상한을 넘지 못한다.

## 8. 컨트랙트 정산

컨트랙트는 `SubletSettlement`(에스크로·판정·분쟁·정산), `ProviderRegistry`(담보·슬래싱), `SubletCatalog`(런타임·모델 카탈로그) 세 개와 EIP-712 타입을 담은 라이브러리 `SubletTypes`다. 설계 문서는 에스크로, 정산, 분쟁을 세 컨트랙트로 나눴는데 하나로 합쳤다. 셋이 같은 잡 상태 머신을 공유해서, 나누면 상태 전이마다 컨트랙트 간 호출이 생기고 그 경계가 재진입과 상태 불일치의 표면이 된다. 담보를 만지는 권한은 잡 자금을 만지는 권한과 달라서 레지스트리는 분리했다.

### 지급액을 서명에 묶었다

전 구간을 실제로 돌려 보다가 찾은 결함이다. `attestVerdict`의 어테스터 서명은 잡 ID, 샤드, 노드, 증거 해시, 판정, 작업량 6개 필드만 덮고 지급액은 서명 밖 파라미터로 받았다. 호출자 제한도 없어서 어테스터 서명을 손에 넣은 누구든 `payout = escrow`로 먼저 부를 수 있었다. 돈이 호출자에게 가지는 않지만, 공급자에게 에스크로 전액이 지급되고 사용자 환불이 0이 된다. 배치 경로는 지급액을 리프에 담아 서명에 묶고 있었으니 두 경로가 서로 다르게 짜여 있었다. 지금은 가격과 작업량 상한을 예치 시점에 커밋하고 지급액은 서명된 작업량으로 계산한다.

```solidity
function _payoutFor(uint64 units, uint128 pricePer1kUnits) private pure returns (uint256) {
    return (uint256(units) * uint256(pricePer1kUnits)) / 1000;
}

function _resolvePayout(Job storage job, uint64 billableUnits) private view returns (uint128) {
    if (billableUnits > job.maxUnits) {
        revert UnitsExceedCommitment(billableUnits, job.maxUnits);
    }
    uint256 payout = _payoutFor(billableUnits, job.pricePer1kUnits);
    if (payout > job.escrow) revert PayoutExceedsEscrow(uint128(payout), job.escrow);
    return uint128(payout);
}
```

상한을 넘으면 잘라내지 않고 되돌린다. 조용히 상한까지만 주면 checker가 부풀린 사실이 어디에도 남지 않는다. `fundJob`도 `maxUnits`까지 일했을 때의 금액을 에스크로가 못 덮으면 `EscrowCannotCoverMaxUnits`로 거부한다. 공급자가 일하고 못 받는 구간이 있다면 실행 전에 알아야 하기 때문이다.

### 어테스터 서명은 정렬해서 받는다

```solidity
        address previous = address(0);
        for (uint256 i = 0; i < signatures.length; ++i) {
            address signer = SubletTypes.recoverSigner(digest, signatures[i]);
            if (signer <= previous) revert UnsortedOrDuplicateSigner();
            if (!isAttestor[signer]) revert NotAnAttestor(signer);
            previous = signer;
        }
```

서명자 주소 오름차순을 요구하면 중복 서명을 O(n)에 거를 수 있다. 정렬 없이 중복을 허용하면 한 어테스터가 임계값을 혼자 채운다(오프체인 배선의 한계는 2절). `recoverSigner`는 서명 malleability를 막으려고 s를 하위 절반으로 제한한다. attestor는 서명키와 가스를 내는 sender 키를 분리했다. 같은 키면 가스가 마르는 순간 승인 권한도 멈춘다.

### 분쟁, 정산, 탈출구

| 함수 | 하는 일 | 신뢰 모델과의 차이 |
|---|---|---|
| `raiseDispute` | 분쟁 창 안에서 에스크로의 20%(로컬 배포 `DISPUTE_BOND_BPS = 2000`)를 걸고 이의 제기. 담보가 없으면 정상 결과에 계속 이의를 걸어 정산을 막을 수 있다 | |
| `resolveDispute` | 인용되면 사용자에게 전액 + 보증금, 노드 슬래싱. 기각되면 지급 + 보증금을 공급자에게 | 주석은 "어테스터 임계값과 같은 권한"을 요구한다고 쓰지만 코드는 `isAttestor[msg.sender]` 한 명만 확인한다. 어테스터 한 명이 분쟁 결과를 정할 수 있다 |
| `settle` | 분쟁 창이 지나면 누구나 호출. 공급자만 부를 수 있으면 공급자가 잠수했을 때 사용자 돈이 갇힌다 | |
| `refundUnattested` | 판정이 오지 않은 잡의 에스크로를 지불자가 회수 | 시간 조건이 없다. `Funded` 상태면 언제든 부를 수 있어서, 노드가 일하는 도중에 지불자가 에스크로를 빼 가면 공급자는 일하고 못 받는다. 에이전트도 실행 전에 에스크로를 확인하지 않는다 |

분쟁 창 마감은 attestor가 로컬 시계 대신 체인 시각으로 판단한다. 시계가 어긋난 만큼 일찍 부르면 `DisputeWindowOpen`으로 되돌려진다. 지급은 전부 pull-payment로 레지스트리에 적립하고 수령자가 `withdraw`한다.

### 단계적 슬래싱과 회복 가능한 정지

```solidity
        uint16 bps;
        if (forgery) {
            bps = SLASH_FORGERY_BPS;
        } else {
            // 첫 확정 위반은 10%. GPU 수치 편차로 억울하게 걸린 노드가
            // 한 번에 퇴출되지 않게 한다 (§7.6).
            bps = node.confirmedViolations == 0 ? SLASH_FIRST_BPS : SLASH_REPEAT_BPS;
            node.confirmedViolations += 1;
        }
```

첫 위반 10%, 재범 50%, 위조 100%다. 슬래싱은 FAIL·위조 판정의 `settle`과 인용된 분쟁에서만 일어난다. 문제는 정확히 `minStake`로 등록한 노드가 첫 10% 슬래싱에 바로 최소치 미달이 된다는 점이다. 그때 영구 퇴출하면 단계적 슬래싱은 의미가 없다. 그래서 담보 미달은 `Suspended`로 두고 담보를 채우면 복귀하게 했다. 영구 `Ejected`는 위조에만 쓴다. 정지 상태도 슬래싱 대상이다. 정지 전에 수행한 잡의 판정이 나중에 도착할 수 있다.

### 대역이 구현과 같은 가정을 공유하면 검사되지 않는다

attestor가 쓰던 `JobState` 상수가 어긋나 있었다. `Settled`를 3으로 적었는데 3은 `Disputed`였다. 정산은 성공하는데 데몬이 그걸 못 알아보고 영원히 재시도했다. 테스트용 체인 대역도 같은 틀린 값을 써서 단위 테스트 15건이 전부 통과했다. 이 일 뒤로 컨트랙트 소스의 enum을 직접 읽어 attestor 상수와 대조하는 테스트를 뒀다. 콘솔이 손으로 하는 ABI 인코딩도 `abi.encodeCall`로 만든 정본과 대조한다(`CalldataParity.t.sol`, `calldata-parity.mjs`). 4바이트 셀렉터를 기억으로 적었다가 두 개를 틀린 뒤의 일이다.

### 가스: R4는 미해결이다

![잡당 온체인 비용](/assets/img/posts/sublet-gpu-marketplace/gas-light.png){: .light }
![잡당 온체인 비용](/assets/img/posts/sublet-gpu-marketplace/gas-dark.png){: .dark }
_`forge test --gas-report` 중앙값. forge 1.7.1, Solidity 0.8.28, optimizer runs 200, OpenZeppelin v5.1.0_

처음 가스를 잰 건 08-24로, 위 지급 결함을 고치며 가격과 `maxUnits` 커밋을 넣기(08-25) 전이었다. 지금 코드로 `forge test --gas-report`를 다시 돌렸다. 저장소가 OpenZeppelin 버전을 고정하지 않아서 v5.1.0으로 받아 쟀다. 중앙값은 테스트 전체 호출 기준이라 `settle`처럼 슬래싱 유무에 따라 경로가 갈리는 함수는 여러 경로가 섞여 있다. 08-24 값도 `forge test --gas-report`로 쟀지만 forge와 OpenZeppelin 버전을 적어 두지 않았고, 배치 값은 테스트 로그의 `gasleft()` 차이라서 지금 값과 바로 비교할 수 없다.

| 호출 | 08-24 (gas-report, 버전 미기록 / 배치는 테스트 로그) | 다시 잰 값 (gas-report 중앙값) |
|---|---|---|
| `fundJob` | 148,960 | 137,983 |
| `attestVerdict` (개별) | 120,052 | 120,524 |
| `settle` | 98,422 | 80,171 |
| 판정, 16잡 배치의 잡당 | 83,780 | 116,601 |

배치의 잡당 값은 fuzz 테스트 호출이 섞이지 않도록 16잡 테스트만 따로 돌려(`--match-test test_batchIsCheaperPerJobThanIndividualAttestation`) 잰 값이다. `applyBatchedVerdict` 16회 중앙값 112,574에 `attestVerdictBatch` 64,438을 16으로 나눈 4,027을 더했다. 개별 어테스트 120,524보다 약 3.3% 적을 뿐이다. 정상 경로(`fundJob` + `attestVerdict` + `settle`) 중앙값 합은 338,678 gas이고 판정을 배치로 바꿔도 334,755 gas다. 서명 검증은 배치당 한 번으로 줄지만 잡마다 자기 상태를 저장하고 갱신하는 비용은 그대로 남는다. 참고로 `--gas-report`를 켜면 "배치가 개별보다 싸야 한다"는 저장소의 단언 `test_batchIsCheaperPerJobThanIndividualAttestation`이 123,318 ≥ 120,052로 실패한다. `--gas-report` 없이 돌리면 같은 테스트가 통과한다.

배치로 묶어도 거의 줄지 않으니 잡 하나마다 트랜잭션을 내는 구조로는 소액 추론의 경제성이 성립하지 않는다. 잡 채널이나 에스크로 묶기 같은 구조 변경 없이는 풀리지 않아서 미해결로 선언했다.

## 9. 자가 온보딩: 노드 키는 서명만 하고 돈은 지갑이 낸다

목표는 "웹에서 토큰 받고 명령 한 줄 실행하면 내 GPU가 노드가 된다"였다.

| 단계 | 흐름 |
|---|---|
| 웹 | 지갑 로그인(SIWE) → "내 GPU 등록" → 일회용 토큰 + 설치 명령 한 줄 |
| 내 PC | Docker 확인 → 등록 → 이미지 빌드 → 모델 캐시 → 벤치마크 |
| 웹 | 대시보드에 노드 등장 → "담보 걸기" → 지갑 서명 2번 → 온체인 Active |

### registerFor

처음 `register`는 노드 신원과 담보 출처를 `msg.sender` 하나로 묶었다. 등록하려면 에이전트가 방금 만든 주소로 사용자가 실제 돈을 보내야 했다. `registerFor`에서는 노드 키가 "이 지갑이 나를 이 정산 주소로 등록해도 좋다"에만 EIP-712로 서명하고 담보는 호출자 지갑에서 나간다. 노드 주소의 토큰 잔액이 0인 채로 등록이 된다.

```solidity
        bytes32 digest = SubletTypes.digest(
            SubletTypes.domainSeparator(block.chainid, address(this)),
            SubletTypes.hashStruct(
                SubletTypes.NodeEnrollment({
                    node: node, payout: payout, specHash: specHash, deadline: deadline
                })
            )
        );
        address recovered = SubletTypes.recoverSigner(digest, consent);
        if (recovered != node) revert BadNodeConsent(recovered, node);
```

동의 서명은 반드시 `payout`을 덮어야 한다. 빼면 남이 내 노드 주소를 자기 정산 주소로 등록해 수익을 가로챌 수 있다. 여기서 실제 버그가 하나 나왔다. 이 코드의 다른 서명은 전부 settlement 주소를 도메인으로 쓰는데, `registerFor`는 `address(this)`, 즉 레지스트리 주소로 다이제스트를 만든다. 습관대로 settlement를 쓴 동의는 온체인에서 `BadNodeConsent`로 되돌아왔다. Solidity 테스트는 레지스트리 주소로 다이제스트를 만들어 전부 통과하고 있었다. 두 쪽이 실제로 만난 건 브라우저가 트랜잭션을 보낸 순간이 처음이었다. `deadline`도 벽시계로 잡았다가 Anvil 체인 시각이 1시간 앞서 있어서 정직한 동의가 만료로 거부됐다. 온체인이 검사하는 값은 체인 시각으로 만든다. 지갑 창을 띄우기 전에는 `eth_call`로 예행 연습을 한다. 만료, 하한 미만, 중복 등록이 사용자의 가스로 드러나면 안 된다.

로컬 Anvil에서 확인한 결과는 `registerFor` status 1, gas 156,571, `stake = 1,000 USDC`, `isActive() = true`였고, 온체인 `specHash`가 에이전트가 자기 명세로 계산한 값과 일치했다.

### SIWE: 메시지를 서버가 만든다

로그인 주체는 지갑 주소 하나이고 사용자 테이블은 없다. 담보를 낸 주소, 정산 받는 주소, 에스크로를 낸 주소가 모두 지갑이라 별도 계정을 얹으면 그 매핑이 틀릴 여지만 생긴다.

흔한 SIWE 구현은 클라이언트가 메시지를 조립하고 서버가 파싱해서 `domain`, `Chain ID`, `nonce`를 하나씩 대조한다. 대조를 하나 빠뜨리면 그게 곧 취약점이다. `domain`을 확인하지 않으면 다른 사이트에서 받은 서명이 여기서 통한다. 그래서 EIP-4361 전문을 서버가 발급하고 nonce로 되찾아 대조했다. 지갑은 텍스트를 그대로 서명하니 한 글자만 달라도 다른 주소가 복구된다. 코드에 `domain`을 비교하는 줄을 두지 않았고, 다른 도메인·다른 체인·한 글자 변조가 각각 401이 되는지를 테스트가 검사한다. 세션은 24시간이고 활동에 따른 연장은 넣지 않았다. 담보를 거는 권한이 붙은 세션이라 만료 시점이 흔들리면 안 된다.

### 등록 토큰과 설치 순서

`POST /enroll`은 세션 인증을 요구하지 않는다. 부르는 쪽은 공급자 기계의 에이전트다. 거기 세션 쿠키가 있다면 그 기계에 로그인 자격이 놓여 있다는 뜻이 된다. 콘솔에서 발급한 토큰(30분, 1회용, 계정당 미소비 5개)이 인증 역할을 하고 노드는 `sublet-node-enrollment / node=… / token=…` 전문에 서명한다. `node_id`만 서명하면 그 서명을 가로채 다른 토큰과 짝지어 남의 노드 신원을 자기 계정에 묶을 수 있다. 여기서는 EIP-712를 쓰지 못했다. 서명 시점에 에이전트가 `chainId`와 `verifyingContract`를 아직 모르고, 그 값을 `/enroll` 응답으로 받기 때문이다. 도메인 분리는 서버가 발급한 1회용 토큰이 대신한다.

설치 순서는 한 번 뒤집었다. 처음엔 clone, 의존성, 이미지 빌드, 모델 캐시를 다 하고 마지막에 등록했다. 토큰 수명은 30분인데 GPU 이미지와 모델 캐시는 느린 회선에서 그보다 오래 걸려서 다 받아 놓고 마지막에 만료로 실패한다. 등록은 키를 만들어 서명을 POST하는 것뿐이고 Docker도 필요 없어서 `--enroll-only`로 맨 앞에 끝낸다. 이 순서는 `tests/test_installers.py`가 고정한다.

다른 기계에서 처음 돌렸을 때 로컬에서는 안 보이던 설치 결함이 7개 나왔다. 몇 가지만 적으면 이렇다.

- 앞서 PowerShell 5.1 때문에 `.ps1`에 BOM을 넣었는데, `irm`으로 받아 scriptblock으로 실행하는 경로에서는 그 BOM 때문에 파싱이 안 됐다. 파일로 받아 자식 프로세스로 실행하게 바꿨다.
- `$ErrorActionPreference = 'Stop'` 아래에서는 네이티브 명령이 stderr에 쓰기만 해도 종료 오류가 나서, Docker 안내 문구가 `NativeCommandError` 스택에 묻혔다.
- 경로 문자열의 `bin\bash.exe`에서 `\b`가 백스페이스(0x08)로 파일에 박혀 화면에는 `binash.exe`로 보였고, 파서는 통과했다.

각각에 설치 스크립트 검사를 넣었다(제어문자 금지, BOM 유무, WSL bash 우선순위, 스크립트 끝 `exit` 금지 등).

## 10. GPU 부분 등록: 8GB 중 4GB만 빌려주기

공급자가 카드 전체를 내놓지 않아도 되게 했다. 브라우저는 VRAM 총량을 신뢰할 수 있게 읽지 못하니 에이전트가 NVML로 재서 절반을 기본값으로 제안한다.

내놓는 양을 최선 노력으로 할지 보장으로 할지가 첫 결정이었다. 처음엔 "선언값과 현재 여유 중 작은 쪽"을 추천했지만 보장으로 확정했다. 보장으로 하면 배정은 선언값만 본다. 주인이 게임을 켠 순간에도 코디네이터는 여전히 그만큼 있다고 믿고 잡을 보낸다. 그 잡은 OOM으로 죽는다. 죽은 건 남의 잡이라 공급자는 그 실패 기록과 미지급을 떠안는다. 그러니 보장을 지키는 장치가 두 겹 필요했다.

| 층 | 무엇을 |
|---|---|
| 예약 홀더 (에이전트) | 드라이버(`nvcuda.dll` / `libcuda.so.1`)를 ctypes로 열어 내놓은 만큼을 `cuMemAlloc`으로 들고 있다. 잡이 돌 때만 놓는다 |
| 자기 제한 (런타임) | 잡 컨테이너가 `set_per_process_memory_fraction`으로 스스로를 묶는다. 초과하면 그 컨테이너가 OOM으로 죽고 주인의 프로그램은 산다 |

호스트가 컨슈머 GPU에서 메모리 상한을 강제할 방법은 없다(MIG는 A100급 기능이다). 두 번째 층이 성립하는 건 잡이 임의 코드와 달리 우리가 만든 런타임이 실행하는 명세이기 때문이다.

```python
            ctx = ctypes.c_void_p()
            driver.check(
                driver.lib.cuCtxCreate_v2(ctypes.byref(ctx), 0, device), "CUDA 컨텍스트 생성"
            )
            self._ctx = ctx
            try:
                free_before, total_mib = self._mem_info(driver)
                if self.target_mib > free_before:
                    raise ReservationError(
                        f"{self.target_mib} MiB 를 확보할 수 없다. 지금 다른 프로그램이 "
                        f"GPU 를 쓰고 있다 (여유 {free_before} / 총 {total_mib} MiB). "
                        "내놓는 양을 줄이거나 그 프로그램을 끄고 다시 시작하라."
                    )
                ptr = ctypes.c_void_p()
                driver.check(
                    driver.lib.cuMemAlloc_v2(
                        ctypes.byref(ptr), ctypes.c_size_t(self.target_mib * _MIB)
                    ),
                    f"{self.target_mib} MiB 예약",
                )
                self._ptr = ptr
                self._held_mib = self._verify_held(driver, free_before)
```

에이전트 호스트에는 torch가 없어서 드라이버 API를 직접 다뤘다. `_v2` 심볼을 쓰는 건 접미사 없는 이름이 32비트 시절 시그니처라 포인터 크기가 달라서다. 확보하지 못하면 조용히 적게 내놓는 쪽으로 물러서지 않고 노드를 올리지 않는다. 약속한 만큼 내놓고 있다고 사용자가 믿게 되는 게 가장 나쁘다.

### 재는 자가 틀렸다

성공 경로를 확인하다가 두 번 틀렸다.

| 시도 | 무엇이 틀렸나 |
|---|---|
| 1차 | "여유가 총량 − 내놓는 양이 되도록" 역산했다. 다른 프로그램이 이미 쓰고 있으면 적게 잡고도 다 잡았다고 보고한다 |
| 2차 | 잡은 뒤 NVML 여유로 확인했다. 멀쩡한 예약을 실패로 판정해 노드가 아예 안 떴다 |

진단 스크립트로 같은 시점을 두 회계로 재 보니 이랬다(RTX 4050 Laptop, Windows).

| 회계 | 요청/예약 | 여유 감소 |
|---|---|---|
| CUDA `cuMemGetInfo` | 2989 MiB 요청 | 2990 MiB |
| NVML | 3070 MiB 예약 | 1361 MiB |

컨텍스트 직후에 이미 CUDA 5075, NVML 5842로 767 MiB가 어긋나 있었고 할당하자 격차가 1710 더 벌어졌다. 예약은 정상이었고 재는 자가 틀렸다. Windows WDDM이 VRAM을 가상화해 다른 프로세스 메모리를 시스템 램으로 밀어내는 게 유력한 설명이지만, 측정한 건 "두 회계가 어긋난다"까지다. 그래서 확인은 CUDA 회계 하나로 통일했고 보장의 범위도 고쳐 적었다. 빌리는 쪽이 약속한 양을 언제나 되찾을 수 있다는 것까지이고, 주인의 앱이 그 메모리를 물리적으로 절대 못 쓴다는 뜻은 아니다.

### 배정은 내놓은 양으로

![내놓은 양 기준 배정](/assets/img/posts/sublet-gpu-marketplace/vram-split-light.png){: .light }
![내놓은 양 기준 배정](/assets/img/posts/sublet-gpu-marketplace/vram-split-dark.png){: .dark }
_같은 노드, 같은 시각에 모델만 바꿔 넣은 두 잡_

스케줄러는 카드 총량 대신 `offered_vram_mib`로 판단하고 모델 추정치에 파편화 여유 256 MiB를 더한다. 다른 사람 노트북(6141 MiB 중 3070 MiB를 내놓음)에 0.5B와 1.5B 잡을 같이 넣으니 0.5B(2304 필요)는 배정되고 1.5B(4864 필요)는 pending으로 남았다. 총량으로 판단했다면 둘 다 배정됐을 것이다.

### 동시 샤드 상한이 하트비트 지연으로 샜다

에이전트는 실행 중인 샤드 수를 오래전부터 하트비트로 보냈는데 스케줄러가 그걸 보지 않았다. 컨테이너마다 모델을 통째로 VRAM에 올리니 6GB 카드는 두 번째 샤드에서 죽는다. 하트비트를 보게 고친 뒤에도 샤드 2개가 같은 노드에 1.02초 간격으로 갔다. 하트비트는 10초마다 오니 두 번 다 0이었다. 코디네이터가 자기가 방금 한 배정을 세지 않은 게 원인이었다.

```python
                # 노드의 자기신고와 코디네이터의 기록 중 **큰 쪽**을 쓴다.
                # 자기신고는 늦게 오고, 기록은 노드가 아직 시작하지 않은 것도
                # 센다. 어느 쪽으로 틀려도 덜 보내는 방향이라 안전하다.
                running_shards=max(
                    agent.running_shards, (active_shards or {}).get(node.node_id, 0)
                ),
```

고친 뒤에는 두 샤드가 73.4초 간격으로 직렬화됐다. 앞 샤드가 끝나고 0.29초 뒤에 다음이 나갔다. 동시 상한은 `내놓은 양 / 모델 요구량`으로 계산하되 당분간 1로 묶어 뒀다. 파편화와 컨텍스트 오버헤드를 실측하기 전에는 "두 개가 들어갈 것"이라는 계산을 믿을 근거가 없다. 두 결함 모두 단위 테스트를 통과한 코드였다.

## 11. 실측: 전 구간 자동 실행과 다른 사람 GPU

### 잡 제출 하나가 온체인 정산까지

로컬 Anvil, 실제 컨테이너, 실제 서명으로 돌렸다. 사람이 한 일은 잡 제출뿐이다.

| 경과 | 잡 | 샤드 | 체인 | 지급 |
|---|---|---|---|---|
| 2s | running | assigned | Funded | 0 |
| 620s | completed | completed | Funded | 0 |
| 642s | completed | completed | Attested | 4 |
| 분쟁 창 1시간 뒤 | | | Settled, PASS | 에스크로 37 중 지급 4, 환불 33 |

금액 단위는 uUSDC(USDC 최소 단위, 소수 6자리)다. 620초가 걸린 건 CPU 추론이어서다. 어테스트에서는 한 attestor 프로세스가 혼자 임계값을 채웠다(2절).

이 실행은 `attempts: 2`였다. 첫 시도가 lease 만료로 실패했는데 같은 노드가 다시 받아 성공했다. 처음엔 일시적 실패(lease 만료)도 노드를 그 샤드에서 영구 제외해서 노드가 한 대인 환경에서 에이전트를 재시작하자 샤드가 영원히 막혔다. 지금은 사라진 노드와 느린 노드를 구분한다.

| lease 만료 시점의 연결 | 사유 | 같은 노드에 다시 주나 |
|---|---|---|
| 끊김 | `lease_expired` | 준다. 전원과 네트워크는 회복된다 |
| 유지 | `too_slow` | 안 준다. 다시 줘도 또 만료된다 |

`too_slow`는 1.5B 샤드가 CPU 노드에 세 번 배정되고 세 번 버려진 걸 보고 넣었다. 컨테이너는 매번 끝까지 돌았고 그 시간은 통째로 낭비였다. 사유에는 "리스 600초 안에 끝내지 못했다 (연결은 유지됐다)"처럼 관찰한 사실만 적고 붙어 있는 노드에는 `lease_revoked`를 보낸다. 이 메시지는 프로토콜에도 있고 에이전트도 처리하는데 코디네이터가 한 번도 보내지 않고 있었다.

### 다른 사람 노트북에서

다른 기계 두 대에서 돌렸다. 첫 번째는 08-27의 RTX 4070 Laptop(드라이버 591.74)으로, 개발기(RTX 4050 Laptop, 591.44)와 GPU도 드라이버도 다른 기계였다. 체인을 켠 상태로 `registerFor` → 에스크로 예치 → 배정 → 11초 만에 완료까지 갔다. 증거에는 TOPLOC 지문 2개, 프롬프트 10토큰, 생성 32토큰, 2145ms, 노드 키 서명이 실렸다. 이때 잡만 돌렸고 이 GPU로 R1 교차 캘리브레이션은 하지 않았다.

두 번째는 다른 사람의 RTX 4050 Laptop이다. 체인을 끈 구성(`CHAIN=off`, 담보·에스크로 없이 프로토콜만)으로 ngrok 고정 주소를 열고 공인 IP가 다른 별개 기계가 명령 한 줄로 노드가 됐다. 같은 노드에서 CPU와 GPU를 나눠 쟀다(Qwen2.5-0.5B, greedy).

| | 프롬프트 | 생성 토큰 | 시간 | 처리량 | 증거의 `gpu_name` |
|---|---|---|---|---|---|
| CPU (`--force-cpu`) | 3 | 72 | 28.6초 | 2.5 tok/s | `cpu-fallback` |
| GPU | 8 | 최대 8,192 | 186.4초 | 약 44 tok/s | `NVIDIA GeForce RTX 4050 Laptop GPU` |

GPU 처리량은 `max_tokens = 1024 × 8`이 전부 생성됐다고 보고 나눈 값이라 상한 쪽 추정이다. 같은 기계의 GEMM 벤치(CPU 103 / GPU 5,532 GFLOPS)는 54배인데 토큰 생성은 약 18배에 그쳤다. 토큰 생성은 연산보다 메모리 대역폭에 묶이고, 0.5B 모델을 배치 없이 한 프롬프트씩 돌리니 하트비트로 본 GPU 사용률도 32~42%에 머물렀다. GPU 실행의 증거에는 TOPLOC 지문 198개와 노드 키 서명이 들어 있다. 어디서 돌았는지가 실제 GPU 이름으로 남는다. 이 노드의 GPU 잡이 GPU 없는 checker에서 SUSPECT, 대금 0이 된 경위는 6절에 적었다.

## 12. 노출면과 접근 제어

공개 주소를 실제로 열어 보고 정리했다. 열기 전에는 안 보이던 구멍이 둘 있었다.

| 경로 | 무엇이 지키는가 |
|---|---|
| `POST /enroll` | 일회용 등록 토큰 (9절) |
| `WS /ws/agent` | 노드 키의 EIP-712 서명. 챌린지에 코디네이터의 도메인이 실려 오고, 에이전트는 자기 도메인과 다르면 연결을 거부한다 |
| `POST /jobs` | 세션 + `OWNER_ALLOWLIST` |
| `GET /jobs`, `/nodes` | 없음 (읽기 전용 메타데이터) |
| `POST /verdicts`, `/work/*` | 없음. 미해결 |

첫 번째 구멍은 `POST /jobs`의 인증 누락이었다. 쿠키 없이 보내 보니 401 대신 422가 왔다. 인증을 지나 본문 검증까지 갔다는 뜻이고, URL을 아는 누구나 등록된 GPU에 일을 시킬 수 있었다. 인증은 FastAPI `Depends`로 걸었다. 함수 본문에서 부르면 FastAPI가 본문 검증을 먼저 해서 인증 안 된 요청이 422와 함께 스키마를 돌려받는다. SIWE는 "이 지갑의 주인이다"만 증명하고 지갑은 누구나 만들 수 있으니 허용 목록도 따로 뒀다. 그런데 그 목록을 `OWNER_ALLOWLIST=0xabc,0xdef`처럼 쉼표로 적으면 pydantic-settings가 `list[str]`을 JSON으로만 읽어서 코디네이터가 `error parsing value`로 죽었다. 보안을 켜려는 순간에 나는 실패였다. 지금은 쉼표 형식을 그대로 받는다.

두 번째는 인프라 포트였다. Docker의 `"5432:5432"`는 모든 인터페이스에 공개되고, 개발용 자격증명은 compose 파일에 적혀 있다. 개발기의 Wi-Fi 주소에서 Postgres, Redis, MinIO, Anvil 포트가 전부 열려 있었고 그 주소는 사설 대역도 아니었다. 전부 `127.0.0.1`에 묶고 외부 접속은 localhost로 붙는 터널만 담당하게 했다. 고치지 않은 보안 항목은 gitignore된 디렉터리에만 적는다.

에이전트가 실패 원인을 올릴 때도 노출을 따졌다. 원래는 `runtime_error` 코드만 남아서 원인("모델 캐시가 비었다")을 찾는 데 몇 시간이 걸렸다. 지금은 첫 오류 줄만, 200자까지, 절대 경로는 파일 이름만 남겨 공급자 기계를 떠나기 전에 지운다. 컨테이너 로그 전체에는 사용자 이름, 디렉터리 구조, 다른 프로세스 정보가 섞여서 그걸 잡 제출자에게 보내면 기계를 빌려준 사람의 정보를 넘기는 셈이 된다. 모델 ID의 슬래시는 경로로 보지 않는다. 그것까지 지우면 진단 가치가 사라진다.

## 13. 한계와 다음 단계

- m-of-n이 실제로는 1-of-1이다(2절). 어테스터 간 서명 수집 경로와 키 분산이 필요하다.
- 컨트랙트의 신뢰 가정에 빈칸이 둘 있다. `resolveDispute`는 주석과 달리 어테스터 한 명만으로 호출된다. `refundUnattested`에는 시간 조건이 없어 지불자가 실행 도중 에스크로를 빼 갈 수 있다.
- R1이 미해결이다. 임계값은 RTX 4050 한 대와 CPU로 잰 잠정값이고 다른 GPU로 교차 캘리브레이션을 하지 않았다. GPU 없는 checker로는 GPU 잡이 SUSPECT, 대금 0이 된다(6절).
- TOPLOC의 비용 이점이 아직 없다. checker는 전체를 다시 생성해서 재검증 비용이 원 추론과 같은 수준이고, 그래서 GPU 잡을 검증하려면 GPU checker가 필요하다.
- R4가 미해결이다. 지급 결함 수정 뒤 다시 잰 정상 경로 중앙값 합이 338,678 gas이고 배치로 판정을 묶어도 334,755 gas로 거의 줄지 않는다. 잡 채널, 에스크로 묶기 같은 구조 변경이 필요하다.
- 작업량 재계수는 산술 일관성까지만 한다. 재토크나이즈는 Phase 2이고, 그때까지 부풀림을 막는 건 컨트랙트의 `maxUnits` 상한뿐이다.
- `sublet-verify`는 서명, 머클 루트, 포함 증명, 토큰 합계, 명세 해시까지 코디네이터 없이 확인하지만 TOPLOC 지문 검증은 SKIPPED다. 온체인 판정 해시와 대조하는 단계도 아직 CLI에 없다.
- 쓰이지 않는 부품이 있다. 커밋-리빌(`commit_reveal.py`)은 구현과 테스트가 있고, replica 비교(`compare_replica`)는 구현만 있다. 둘 다 서비스에 배선되지 않았고, 같은 샤드를 두 노드에 주는 코드가 없어서 명세의 `redundancy`가 아무 일도 하지 않는다. SUSPECT의 2차 재검증 경로도 없다.
- 판정·정산 엔드포인트(`POST /verdicts`, `/work/*`)에 인증이 없다. checker와 attestor는 데몬이라 세션 쿠키가 맞지 않고 기계 인증 설계가 필요하다. `jobs`에 소유자 컬럼이 없어서 허용 계정이 둘 이상이면 서로의 GPU를 쓴다.
- 두 기계에서 각각 돌려 봤지만 같은 시점에 두 노드를 붙여 lease 경합과 동시 배정을 확인한 적은 없다.
- 실제 배포가 없다. 서비스 이미지(477MB, 비특권 uid)는 빌드·기동까지 확인했지만 코디네이터, DB, 저장소가 개발기에 있어서 PC가 꺼지면 노드가 끊긴다. Base Sepolia 배포 스크립트는 준비했고 faucet을 못 받아 배포하지 않았다.
- 격리는 Docker 기본 격리 + 권한 박탈이다. 커널은 공유되고 gVisor 승격은 Phase 4로 남겼다.
- 추론만 된다. 학습 워크로드와 AI 에이전트 작업 관리자는 손대지 않았다.

이 프로젝트에서 반복해서 나온 패턴은 단위 테스트가 초록인 채로 실제 실행에서만 드러나는 결함이었다. 지급액 파라미터, `JobState` 상수, 레지스트리 도메인, NVML 회계, 하트비트 지연, 설치 스크립트의 백스페이스까지 전부 그랬다. 그래서 대역끼리 맞춰 보던 테스트를 소스끼리 대조하는 테스트(EIP-712 벡터, enum, calldata)로 바꾸고 계산을 믿기 전에 재는 자부터 확인하게 됐다.
