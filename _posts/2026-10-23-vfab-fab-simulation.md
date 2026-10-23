---
title: "SMT2020 팹을 SimPy로 다시 짜고, 기준 엔진과 처리량 57.2 대 57.2를 맞추기까지"
date: 2026-10-23 10:30:00 +0900
categories: [Simulation, DES]
tags: [simpy, discrete-event-simulation, smt2020, pyscfabsim, semiconductor, dispatching, python]
math: true
image:
  path: /assets/img/posts/vfab-fab-simulation/crosscheck-light.png
  alt: VFAB 엔진과 PySCFabSim의 HVLM 사이클타임 비교
---

반도체 팹 테스트베드 SMT2020을 읽어 들이는 파서와, 재진입·배치·셋업·PM·고장을 돌리는 이산사건 시뮬레이터를 SimPy 위에 직접 짰다. 기준 시뮬레이터 PySCFabSim과 입력을 칸 단위로, 출력을 실행 단위로 대조한 뒤 디스패칭 규칙과 장비 정지의 효과를 쟀다.

| 항목 | 값 |
|---|---|
| 팹 모델 | SMT2020 HVLM·LVHM. 라우트 583스텝, 툴그룹 106, 장비 1,043대, PM 292건 |
| 엔진 | SimPy + 직접 짠 장비 단위 디스패처 (`sim/engine.py` 963줄) |
| 입력 대조 | PySCFabSim 사본과 85,582칸 대조, 다른 칸 62개 |
| 엔진 대조 | 고장·PM을 끄면 HVLM 처리량 57.2 대 57.2 로트/일, 사이클타임 24.4 대 25.1일 |
| 디스패치 비교 | HVLM, 워크북 순위 대비: 셋업을 무시하는 규칙 3종이 하루 7.4~8.2로트(13~15%) 덜 만든다 |
| 실행 비용 | 단독 실행은 HVLM 1년, 코어 하나로 약 9분(README). 로드맵은 시뮬레이션 한 번을 20~30분으로 잡았다 |
| 검증 | pytest 1,524개 통과, ruff·mypy 0건 (2026-10-07 기준) |

## 1. 무엇을 시뮬레이션하고 무엇을 안 했나

VFAB는 혼자 만든 반도체 제조 데이터 분석 프로젝트이고, 모듈은 FDC 이상감지(SECOM), 웨이퍼맵 분류(WM-811K), OEE 추정(KAMP), 팹 구조 분석(M3-B), DES 시뮬레이션(M4) 다섯 개와 선택 모듈인 공구 마모 판별(KAMP CNC)이다.

SMT2020은 실측 로그가 아니라 **시뮬레이션 모델 파라미터**다. 라우트, 툴그룹별 장비 대수, 처리시간 분포, 셋업 행렬, PM과 고장 규칙이 엑셀 시트로 들어 있다. 그래서 "실제 팹 데이터를 분석했다"가 아니라 "업계 표준 팹 테스트베드로 모델링했다"가 정확하다. 실물 설비와 동기화되지 않으니 디지털 트윈이라고 부르지도 않는다.

물류는 모델링하지 않았다. SMT2020이 주는 반송 모델이 이 한 줄이 전부다.

| FROM | TO | 분포 | 평균 | offset | 단위 |
|---|---|---|---|---|---|
| Fab | Fab | uniform | 7.5 | 2.5 | min |

차량 대수, 경로, 배차 규칙을 넣으려면 전부 지어내야 하고, 그러면 결과가 테스트베드가 아니라 내 가정 위의 숫자가 된다. 반송은 이 분포 그대로 스텝 사이 지연으로만 넣었다.

## 2. SMT2020을 믿을 수 있게 읽기

### 단위부터 초로

SMT2020은 시간 단위가 열마다 min, hr, day로 섞여 있다. 파서는 단위 열을 읽어 전부 초로 바꾸고, 단위가 없거나 시간 단위가 아니면 거부한다.

```python
TIME_UNITS = {"sec": 1.0, "s": 1.0, "min": 60.0, "hr": 3600.0, "h": 3600.0, "day": 86400.0}

def to_seconds(value: object, unit: object) -> float:
    """``value`` in ``unit`` as seconds. A missing value stays missing;
    a value without a unit, or with a unit that is not a time, is refused."""
    number = float(value) if value is not None else math.nan  # type: ignore[arg-type]
    if math.isnan(number):
        return math.nan
    if not isinstance(unit, str):
        raise ValueError(f"the value {number} has no unit")
    key = unit.strip().lower()
    if key not in TIME_UNITS:
        raise ValueError(f"{unit!r} is not a time unit (known: {sorted(TIME_UNITS)})")
    return number * TIME_UNITS[key]
```

파싱 결과는 `fab_params_{hvlm,lvhm}.json`이라는 계약 파일로 M4에 넘긴다. 읽는 쪽은 `validate_fab_params`로 먼저 검사한다. 형식, 중복·0·음수 같은 크기, 툴그룹·재작업 대상 스텝 같은 참조, 배치·cascade·CQT 규칙의 모순을 막는다. 분을 초로 잘못 적은 값은 계약 검사가 못 잡는다. 그건 단위 열을 읽는 파서의 책임으로 나눴다.

### 기준 엔진의 사본과 칸마다 대조

PySCFabSim(MIT)은 SMT2020을 자기 형식의 탭 구분 파일로 바꿔 들고 있다. 두 사본이 같은 말을 하면 파서와 기준 엔진이 의미를 같게 읽은 것이고, 다르면 어느 한쪽에 오기가 있다. 입력을 먼저 맞추지 않으면 뒤에서 할 엔진 비교가 서로 다른 팹을 비교하게 된다.

```python
def _differs(ours: pd.Series, theirs: pd.Series) -> pd.Series:
    """True where the two differ; two blanks are equal, a blank and a
    value differ, numbers are equal to 1e-6."""
    both_blank = ours.isna() & theirs.isna()
    numeric = pd.api.types.is_numeric_dtype(ours) and pd.api.types.is_numeric_dtype(theirs)
    if numeric:
        same = (ours.astype(float) - theirs.astype(float)).abs() <= 1e-6
    else:
        same = ours.astype("string") == theirs.astype("string")
    return ~(both_blank | same.fillna(False).astype(bool))
```

라우트는 스텝마다 툴그룹, 처리 단위, 처리시간 분포·평균·offset, cascade 간격, 배치 최소·최대, 셋업, LTL, 재작업, 샘플링, CQT까지 16개 열을 짝지어 비교한다. PM은 종류, 간격, 수리 시간, 첫 발생까지 본다. 라우트·PM·툴그룹·셋업·고장·투입·반송 표를 합쳐 85,582칸을 비교했고(HVLM·LVHM), 다른 칸이 62개였다.

| 다른 값 | 칸 수 | 처리 |
|---|---|---|
| `Route_Product_3`의 cascade 간격 | 30 × 2 시나리오 | 툴그룹마다 정해진 비에서 벗어나 xlsx 오기로 보이지만 원값을 둔다 |
| SuperHot 플래그 | 1 × 2 시나리오 | PySCFabSim은 이 열을 읽지 않는다 |

나중에 엔진에서 cascade 30개를 PySCFabSim 값(약 10배)으로 바꿔 돌려 보면 사이클타임 차이가 HVLM −0.1일 [−0.7, 0.6]으로 구간이 0을 포함한다.

### 라인이 어떻게 생겼나

![재진입 히트맵](/assets/img/posts/vfab-fab-simulation/reentry.png)
_가장 많이 쓰이는 툴그룹 25개가 583스텝 중 어디서 다시 나오는지. 전공정(FE)은 주로 앞, 후공정(BE)은 약 3분의 2 지점부터_

한 로트가 583스텝 동안 같은 툴그룹을 수십 번 다시 찾는다. 습식 식각은 52번이다. 노광 스텝은 35개다. 이 재진입 때문에 한 툴그룹의 대기열에 공정 초반 로트와 후반 로트가 섞이고, "어느 로트를 먼저 넣을지"가 팹 전체의 흐름을 바꾼다.

## 3. 병목은 한 값으로 말하지 않았다

정적 부하율은 장비 대수와 처리시간에서 산술로 나온다. 발견이 아니라 재구성이다. 그런데 산술만으로도 값이 하나로 정해지지 않는다. 두 개의 값이 시트에 없기 때문이다.

- **배치를 얼마나 채우나.** 확산로는 배치 최소·최대 크기만 있고, 실제로 몇 장을 넣고 시작할지는 디스패치가 정한다.
- **셋업을 얼마나 자주 치르나.** 이온주입은 셋업이 있고, 몇 로트마다 바뀔지는 역시 디스패치가 정한다.

그래서 이 두 값을 양 끝으로 놓고 부하율을 범위로 냈다.

![툴그룹 부하율과 가정의 범위](/assets/img/posts/vfab-fab-simulation/toolgroup-load.png)
_검은 점: 꽉 찬 배치·셋업 없음. 네모: 배치를 최소로만 채움. 세모: 셋업을 매번 치름(상한)_

| 가정 | 시나리오 | 먼저 차는 곳 | 부하율 |
|---|---|---|---|
| 꽉 찬 배치, 셋업 없음 | HVLM | 노광기 `Litho_BE_110` | 0.974, 모든 툴그룹 1 미만 |
| 배치를 최소로만 채움 | HVLM | 확산로 `Diffusion_FE_101` | 0.991 |
| 배치를 최소로만 채움 | LVHM | 확산로 `Diffusion_FE_100` | 1.006 |
| 셋업을 매번 치름 | HVLM | 이온주입 `Implant_128` | 1.140 |

가정 하나를 바꾸면 병목이 노광기에서 확산로로, 다시 이온주입으로 넘어간다. 정적 분석이 할 수 있는 말은 "이 셋 중 하나"까지다. 어느 쪽인지는 디스패치를 실제로 돌려 봐야 나온다. 그래서 답을 M4로 넘겼다.

## 4. 엔진: `simpy.Resource`를 쓰지 않은 이유

SimPy로 장비를 모델링하는 가장 흔한 방법은 툴그룹 하나를 `simpy.Resource(capacity=n)`로 두는 것이다. 이 팹에서는 안 된다. 엔진 docstring에 적어 둔 이유 그대로다.

- Resource의 요청 큐는 FIFO라 워크북의 순위 규칙(우선순위, 최소 셋업, 임계율)으로 로트를 고를 수 없다.
- LTL(lot-to-lens) 규칙은 뒤 스텝을 앞 스텝을 처리한 바로 그 장비에 묶는다. Resource는 특정 장비를 지정할 수 없다.
- cascade 장비는 로트가 끝나기 전에 장비를 놓아준다.
- 배치 장비는 로트 여러 개를 한 번에 싣는다.

그래서 툴그룹을 장비 `count`대의 목록으로 두고, 대기열에서 로트를 고르는 디스패처를 직접 짰다. PySCFabSim도 같은 구조다.

### 같은 순간에 도착한 로트는 함께 줄 세운다

로트가 대기열에 오거나 장비가 비면 디스패치를 요청한다. 그런데 바로 디스패치하면, 같은 시각에 도착한 로트 둘 중 먼저 처리된 이벤트의 로트가 순위와 무관하게 장비를 가져간다. 그래서 디스패치를 그 순간의 **모든 이벤트가 끝난 뒤** 한 번만 돌린다. SimPy는 같은 시각의 이벤트를 우선순위(URGENT 0, NORMAL 1) 순으로 처리하니, 그보다 낮은 우선순위 2짜리 이벤트를 만들었다.

```python
# SimPy processes events of one instant by priority (URGENT 0, NORMAL 1)
LAST_IN_INSTANT = simpy.events.EventPriority(2)


class _LastInInstant(simpy.events.Event):
    """A zero-delay event processed after every URGENT and NORMAL event of
    its instant, including ones scheduled after it (built like SimPy's own
    ``Timeout``, with a lower priority)."""

    def __init__(self, env: simpy.Environment) -> None:
        super().__init__(env)
        self._ok = True
        self._value = None
        env.schedule(self, LAST_IN_INSTANT, 0)
```

```python
def _request_dispatch(self, group: ToolGroup) -> None:
    """Dispatch ``group`` once, after every other event of this instant -
    also those created later in it, such as a zero-length move."""
    if group.name in self._pending:
        return
    self._pending.add(group.name)
    tick = _LastInInstant(self.env)
    tick.callbacks.append(functools.partial(self._on_tick, group))
```

`_pending`가 툴그룹당 한 번만 예약되게 막는다. 이 동작은 Task 31 리뷰에서 "같은 시각에 도착한 로트를 함께 줄 세운다"는 수정으로 들어갔다.

### 장비가 멈추면 위에 있던 작업이 밀린다

고장과 PM은 같은 함수 `_down`으로 처리한다. 장비 위에서 진행 중인 대기마다 `_Box`를 하나 두고, 수리가 오면 그 끝 시각을 수리 길이만큼 민다.

```python
def _down(self, tool: _Tool, kind: str, length: float) -> None:
    """Take ``tool`` down for ``length``: whatever is on it is postponed;
    an idle tool cannot be dispatched until then. Repairs add up."""
    self._downs.append((tool.name, tool.group, kind, self.env.now, length))
    tool.repair_until = max(tool.repair_until, self.env.now) + length
    tool.repair_kind = kind
    tool.down_s += self._book(length)
    # ... 셋업 회피 규칙의 대기 시간 집계 생략
    for box in tool.boxes:
        box.end += length
    if not tool.busy:
        tool.down_until = max(tool.down_until, self.env.now) + length
        self.env.process(self._back_up(tool))


def _on_tool(self, tool: _Tool, duration: float) -> Generator[simpy.Event, object, None]:
    """Wait ``duration`` on ``tool``; a repair meanwhile postpones the end."""
    box = _Box(self.env.now + duration)
    tool.boxes.append(box)
    while box.end - self.env.now > TOLERANCE_S:
        yield self.env.timeout(box.end - self.env.now)
    tool.boxes.remove(box)
```

SimPy의 `timeout`은 한 번 걸면 끝 시각을 바꿀 수 없다. 그래서 끝 시각을 `_Box`에 따로 들고, 깨어날 때마다 그새 밀렸는지 확인하고 남은 만큼 다시 잔다. SimPy에는 프로세스를 `interrupt()`로 깨우는 방법도 있다.

마지막 두 줄이 PySCFabSim과 갈리는 지점이다. PySCFabSim에서는 쉬는 장비에 고장이나 PM이 와도 아무 일이 없다. 바쁜 장비의 작업만 밀린다. 우리 엔진은 쉬는 장비도 수리가 끝날 때까지 디스패치하지 않는다. M3-B에서 가용률을 계산할 때 쓴 읽기("장비는 바쁘든 쉬든 수리 시간 내내 빠진다")를 엔진에도 그대로 썼다(차이는 5절).

### 공통 난수: 추첨은 쓰임새마다 따로

규칙 A와 B를 같은 시드로 돌려 비교하려면, 같은 로트가 같은 처리시간을, 같은 장비가 같은 고장을 받아야 한다. 난수 스트림 하나를 실행 전체가 나눠 쓰면 이게 깨진다. 디스패치가 바뀌면 이벤트 순서가 바뀌고, 순서가 바뀌면 같은 스트림에서 뽑히는 값이 다른 로트에게 간다.

그래서 추첨마다 **무엇을 위한 것인지**로 이름을 붙인 스트림을 쓴다.

```python
def _stream(self, *key: int) -> np.random.Generator:
    """The random stream named ``key`` for this run's seed."""
    return np.random.default_rng(np.random.SeedSequence(self.seed, spawn_key=key))
```

| 키 앞자리 | 스트림 | 이름 |
|---|---|---|
| 0 | 로트의 샘플링·처리시간·재작업·반송 | (로트형, 로트 번호) |
| 1 | 장비 고장 | (툴그룹, 장비, 고장 규칙) |
| 2 | PM 수리 시간 | (툴그룹, 장비, PM 번호) |
| 4 | 배치 처리시간 | (툴그룹, 장비) |
| 6 | Q4 품질 게이트 판정 | (로트형, 로트 번호) |

이 성질은 테스트로 고정했다. 로트가 지나가지 않는 다른 툴그룹에 PM을 켜고 끄면서, 로트들의 처리시간이 한 값도 안 바뀌는지 본다.

```python
def test_a_pm_elsewhere_does_not_move_a_lots_process_times() -> None:
    """Lots visit only T; a PM with random repair times runs on U. With one
    shared stream every PM repair would shift the lots' draws after it."""
    pm = {
        "toolgroup": "U", "kind": "time", "interval": 300.0, "interval_unit": "s",
        "first_at": 50.0, "ttr_dist": "exponential", "ttr_mean_s": 30.0, "ttr_offset_s": None,
    }  # fmt: skip
    config = fab([_random_step()], tools=[_tool("T"), _tool("U")], pm=[pm])
    with_pm = Simulation(config, seed=3).run(20000.0)
    without = Simulation(config, seed=3, pm_policy="none").run(20000.0)
    assert (with_pm.downs["toolgroup"] == "U").sum() > 10  # the PM did draw
    a, b = _durations(with_pm.events), _durations(without.events)
    assert len(a) > 50
    pd.testing.assert_series_equal(a, b)
```

이 테스트가 처음부터 충분했던 건 아니다. Task 33 1차 리뷰에서 "규칙이 바뀌어도 로트의 추첨은 같다"는 핵심 전제에 테스트가 없다는 지적을 받았다. 픽스처에 샘플링 1 미만, 재작업, 확률 반송이 없어서 그 추첨을 공유 스트림으로 되돌려도 다 통과했다. 2차 리뷰에서는 내가 만든 "공유 스트림" 변이 자체가 진짜보다 잡기 쉬웠다는 게 드러났다. 리뷰어가 넣은 진짜 공유 스트림은 대칭 픽스처에서 추첨 순서가 우연히 같아 4개가 살아남았다. 리뷰를 네 번 받고 나서야 통과했다.

## 5. PySCFabSim 교차검증: 12.5일 차이의 출처

같은 SMT2020 입력을 두 엔진으로 365일씩 돌리고 200~365일 창을 잰다. 두 엔진의 결과를 같은 함수(`window_summary`)로 집계한다. PySCFabSim 자체 통계는 초기 WIP 로트까지 세서 쓰지 않았다.

| HVLM, 시드 10개 평균 | 처리량 (로트/일) | 사이클타임 (일) | 납기 준수 |
|---|---|---|---|
| 투입 계획 | 57.2 | | |
| PySCFabSim FIFO | 55.0 | 54.5 | 0.01 |
| VFAB FIFO_setup | 56.4 | 41.9 | 0.55 |

처리량은 비슷한데 사이클타임이 12.5일 다르다.

### 하나씩 맞추기를 여덟 번 했다

처음에는 차이가 날 만한 것을 하나씩 맞췄다. 디스패치 순서, CQT 창 안 로트 우선, 배치 시작 조건, 배치 묶음, `min_runs`, cascade 값, 초기 WIP, 그리고 셋(배치 시작·`min_runs`·cascade)을 한꺼번에. 대부분 하루 안팎이었다. 가장 큰 것은 배치 묶음이었는데, PySCFabSim을 제품별로 묶으면 오히려 +4.1일 늘어 차이가 벌어졌다.

그다음에 한 게 고장과 PM을 **양쪽에서 다 끄는** 실험이다.

| 고장·PM 없음, HVLM, 시드 2개 | 처리량 | 사이클타임 | P90 |
|---|---|---|---|
| PySCFabSim FIFO | 57.2 | 24.4 | 31.3 |
| VFAB FIFO_setup | 57.2 | 25.1 | 32.2 |

처리량은 같고 사이클타임 차이가 0.7일로 줄었다. 기본 흐름(재진입, 배치, 셋업, 디스패치)은 두 엔진이 같고, 차이는 다운타임 처리에 있다는 뜻이다. 실험 4회로 끝났다. 큰 덩어리를 먼저 끄고 그 안에서 좁히는 이분법을 처음부터 했어야 했다. 이 순서 실수는 작업 일지에 그대로 적었다.

### 범인은 첫 PM의 시점

SMT2020의 PM 첫 발생(`FOA`)은 전부 상수다. 이걸 툴그룹의 장비들에 어떻게 나누느냐가 두 엔진에서 다르다.

```python
def _start_pm(self, pm: PMEvent, number: int) -> None:
    tools = self._tools.get(pm.toolgroup, [])
    for k, tool in enumerate(tools):
        # the specification's even spread, or every tool at once (PySCFabSim)
        first = pm.first_at * (k + 1) / len(tools) if self.pm_first == "spread" else pm.first_at
        rng = self._stream(2, self._group_index[pm.toolgroup], tool.index, number)
        if pm.kind == "counter":
            tool.counters.append(_Counter(pm, first, rng))
        # ... Q1 열화(CBM) 실험 분기 생략
        else:
            self.env.process(self._timed_pm(tool, pm, first, rng))
```

명세는 `FOA × k / 대수`로 나눈다. PySCFabSim은 장비 전부에 같은 첫 PM을 준다. 시간 기준 PM은 간격이 상수라 한 번 같이 멈추면 계속 같이 멈추고, 다음 PM이 수리 길이만큼씩만 어긋난다. 평균 가용률은 같아도 그룹 전체가 한꺼번에 서는 동안 대기열이 쌓이고, 풀리는 데 오래 걸린다.

한쪽만 바꿔 보면 결론이 섣부르다. 실제로 우리 엔진에서만 "한꺼번에"로 바꿨을 때 +19일이 나와서 차이(12.5일)의 1.5배였는데 "주 원인"이라고 썼다가 리뷰에서 지적받았다. 그래서 양쪽 엔진에서 반대로도 바꿨다.

![첫 PM 방식에 따른 사이클타임](/assets/img/posts/vfab-fab-simulation/crosscheck-light.png){: .light }
![첫 PM 방식에 따른 사이클타임](/assets/img/posts/vfab-fab-simulation/crosscheck-dark.png){: .dark }
_원래 설정은 시드 10개, 맞바꾼 쪽은 PySCFabSim 시드 2개, VFAB 시드 4개_

| | 사이클타임 (일) | 짝지은 차이 [95% 구간] |
|---|---|---|
| VFAB 나눔 → 한꺼번에 (시드 1~4) | 42.2 → 61.5 | +19.2 [18.2, 20.3] |
| PySCFabSim 한꺼번에 → 나눔 (시드 0~1) | 54.5 → 39.0 | −15.6 [−18.5, −12.7] |

PySCFabSim에서 첫 PM을 나누면 우리 쪽 수준으로 오고, 우리 엔진에서 한꺼번에 주면 PySCFabSim 쪽으로 간다. LVHM에서는 이 하나로 PySCFabSim의 납기 준수가 0.30에서 1.00이 됐다(우리 0.97). 엔진의 오류가 아니라 같은 데이터를 읽는 두 방법의 차이이고, 우리 엔진은 명세를 따랐다. SMT2020 결과를 서로 비교하려면 PM 첫 발생을 어떻게 읽었는지부터 맞춰야 한다.

### 남은 3~7일

맞바꾼 뒤에도 두 방향 모두 우리 엔진이 길다. 나눌 때 41.9 대 39.0, 한꺼번에일 때 61.5 대 54.5다. 4절의 "쉬는 장비도 수리 중이면 못 쓴다"와 같은 방향이고, 그룹이 한꺼번에 멈출 때 차이가 커지는 것도 맞는다. 다만 이 몫은 따로 재지 않았다. 추정으로 남겨 뒀다.

## 6. 디스패칭 규칙 비교: 공통 난수로 짝짓기

규칙 7개(워크북 순위, FIFO, EDD, 최소 여유, 그리고 셋이 각각 셋업을 먼저 보는 `_setup` 판)를 시드 10개, 두 시나리오로 140회 돌렸다. 공통 난수 덕분에 같은 시드 안에서 "규칙 − 기준"은 대부분 규칙의 효과다. 구간은 그 시드별 차이에 건다.

```python
table = runs.pivot(index=pair, columns=by, values=metric)
# ... 중복 실행·누락 시드 검사 생략
base = table[baseline]
n = len(table)
if n < 2:
    raise ValueError("a paired interval needs at least two seeds")
half = stats.t.ppf(0.5 + level / 2, n - 1)
rows = []
for policy in table.columns:
    if policy == baseline:
        continue
    diff = table[policy] - base
    sd = float(diff.std(ddof=1))
    mean = float(diff.mean())
    rows.append(
        {
            # ... 다른 열 생략
            "diff_lo": mean - half * sd / np.sqrt(n),
            "diff_hi": mean + half * sd / np.sqrt(n),
            "diff_sd": sd,
            "unpaired_sd": float(np.sqrt(table[policy].var(ddof=1) + base.var(ddof=1))),
            "seeds_above": int((diff > 0).sum()),
        }
    )
```

`unpaired_sd`는 짝짓지 않았다면 차이가 얼마나 퍼졌을지다. 둘의 비율이 공통 난수가 실제로 얼마를 벌었는지 보여 준다.

![디스패칭 규칙별 처리량 차이](/assets/img/posts/vfab-fab-simulation/dispatch-light.png){: .light }
![디스패칭 규칙별 처리량 차이](/assets/img/posts/vfab-fab-simulation/dispatch-dark.png){: .dark }
_기준은 SMT2020 워크북 순위. 막대는 시드 10개 짝지은 차이의 95% 구간_

| 질문 | 답 (HVLM 중심) |
|---|---|
| 셋업을 무시하면 | FIFO −7.35, EDD −7.65, 최소 여유 −8.20 로트/일. 사이클타임은 21~25일 길어지고, 창 안에서도 WIP가 늘어 정상상태가 아니다. 이 실험에서 가장 큰 효과 |
| 왜 그런가 | 시드 1 진단에서 셋업 무시 FIFO는 쉬는 시간이 없는 이온주입 두 그룹에서 셋업에 7%p를 더 쓰고 가공이 10% 준다 |
| 셋업을 먼저 보면 어느 순서가 낫나 | CQT를 빼면 FIFO. EDD·최소 여유는 처리량이 하루 0.8~1.2로트 낮고 납기 준수도 낮다. CQT 위반만은 적다 |
| 워크북 순위는 | 평균이 가장 높지만 FIFO_setup과는 구별되지 않는다(−0.17 [−0.64, 0.30]) |
| 공통 난수가 구간을 좁혔나 | 주역은 아니다. 차이의 SD가 짝짓지 않았을 때의 0.65~1.22배였다. 가장 많이 준 지표는 HVLM 처리량이고, 그 중앙값이 0.80 |

마지막 줄은 기대와 달랐다. 구간이 좁은 건 실행 하나가 165일 동안 9천 로트를 평균하기 때문이었다. 공통 난수의 값은 분산 감소보다 비교가 공정하다는 데 있다.

납기 준수율은 해석에 주의가 필요하다. SMT2020에서 제품마다 납기 여유가 FIFO 계열의 평균 사이클타임과 거의 같아서, 규칙이 같아도 절반쯤 늦는 구조다. 처음 쓴 결과 서술에서 이걸 "투입을 못 따라가서"라고 잘못 적었다가 리뷰에서 고쳤다.

## 7. 장비 정지 시나리오: 오경보 한 번이 1시간 정지일 때

프로젝트에서 데이터셋을 넘나드는 유일한 연결이 이 실험(Q4)이다. SECOM과 SMT2020 사이에는 공통 키가 없어서 센서값을 옮길 수 없다. 대신 M1 FDC 판정기의 **동작특성**(재현율·오경보율 곡선)만 가져와 SMT2020 건식 식각 툴그룹 뒤의 검사 게이트에 넣었다.

```python
def _judge(
    self, lot: _Lot, step: Step, tool: _Tool, gate: QualityGate
) -> Generator[simpy.Event, object, None]:
    # ... docstring, 게이트 스트림 확인 생략
    abnormal = bool(rng.random() < gate.p)
    alarm = bool(rng.random() < (gate.recall if abnormal else gate.false_alarm))
    lot.abnormal = abnormal or bool(lot.abnormal)
    if not alarm:
        return
    lot.alarms += 1
    self._log(lot, step, tool.name, "alarm")
    self._down(tool, "fdc", gate.hold_s)
    yield self.env.timeout(gate.hold_s)
    if abnormal:
        lot.scrapped_s = self.env.now
        self._log(lot, step, tool.name, "scrap")
```

경보가 나면 그 장비를 `hold_s`(1시간) 세운다. 고장·PM과 같은 `_down`을 쓰니 위의 작업이 밀리는 방식도 같다. 이상률은 4.98%로 가정했다.

M1의 비용식은 오경보를 상수 비용으로 센다. 라인에서는 다르다. 정지가 툴그룹의 남는 시간 안이면 공짜이고, 넘으면 양품을 잃는다. 이걸 해석식으로 쓰면 툴그룹 용량은 이렇다.

$$
\text{capacity}(a) = \frac{n \cdot 86400 \cdot A}{t_{lot} + a \cdot t_{hold}}
$$

$n$은 장비 대수, $A$는 가용률, $t_{lot}$는 로트당 장비 시간, $a$는 경보율이다. 이 용량이 팹의 산출 $X_0$ 아래로 내려가는 경보율이 꺾이는 점이다.

$$
a_{knee} = \frac{n \cdot 86400 \cdot A / X_0 - t_{lot}}{t_{hold}}
$$

```python
def knee(self) -> float:
    """The alarm rate where the gate starts to cost lots."""
    per_lot = self.tools * 86400.0 * self.availability / self.output_per_day
    return (per_lot - self.lot_s) / self.hold_s
```

같은 판정기를 병목 `DE_FE_54`(꺾임 0.24)와 여유 있는 `DE_BE_12`(꺾임 0.55)에 각각 두었다.

![경보율과 잃는 양품](/assets/img/posts/vfab-fab-simulation/gate-light.png){: .light }
![경보율과 잃는 양품](/assets/img/posts/vfab-fab-simulation/gate-dark.png){: .dark }
_점은 DES, ×는 각 실행의 가용률을 넣은 해석식. HVLM 시드 1_

해석식은 DES 8회(기준 2회 포함)로 확인했다. 꺾임 위의 네 점에서 DES와 0.02~0.30 로트/일 안이다. 꺾임 아래인 여유 위치 0.3 점은 해석식이 0, DES가 −0.72다. 점마다 시드 하나라 구간이 아니다. 각 실행의 가용률을 따로 넣은 이유도 있다. 경보가 늘면 웨이퍼 수 기준 PM이 줄어 가용률이 오른다.

| 비용비 | M1이 보고한 운용점의 경보율 | 병목 `DE_FE_54`에서 잃는 양품 | 여유 `DE_BE_12` |
|---|---|---|---|
| 5:1 | 0.05 | 0 (꺾임 아래) | 0 |
| 20:1 | 0.67 | 9.8 로트/일 | 1.8 로트/일 |
| 100:1 | 0.98 | 14.8 로트/일 | 5.9 로트/일 |

비용비 20:1에서 M1이 오프라인으로 고른 최적점은 병목 식각에서 하루 양품 9.8로트를 잃는 자리였다. 같은 판정기라도 어느 장비 뒤에 두느냐로 비용이 달라진다. 결론이 기대는 가정도 분명하다. 시간순 M1은 우연보다 조금 나을 뿐이고, 정지 시간을 0.5시간으로 줄이면 여유 위치는 전수 경보도 견딘다. 이상률과 양품 한 로트의 값도 가정이다.

같은 엔진으로 장비 정지를 다른 방향에서도 물었다.

| 실험 | 정지의 형태 | 결과 |
|---|---|---|
| Q2 증설 | 이온주입 세 그룹에 한 대씩 | 투입 계획 대비 부족분 0.8로트/일을 못 메웠다. 사이클타임 −1.6일. 밀어 넣으면 97% 넘게 쓰이는 그룹이 10개(노광·이온주입·건식 식각)라 병목을 한 곳으로 말할 수 없었다 |
| Q1 PM 정책 | 고정주기 PM 대 상태기반 정비 | 열화 모델은 합성(Weibull 수명, P-F 간격 5일). CBM이 이기려면 점검 1회 특이도 0.962 이상. 팹 처리량에서는 정책을 가르지 못했다(시드 2개) |

Q1의 열화가 합성인 이유는 SMT2020의 고장이 무기억(지수분포)이라서다. 무기억 고장에서는 정비 시점이 고장률을 바꾸지 못하므로, 그대로 두면 PM 정책 비교 자체가 의미가 없다.

## 8. 로트 하나의 길

![로트 하나의 여정](/assets/img/posts/vfab-fab-simulation/lot-journey.png)
_대시보드 시뮬레이션 탭. HVLM 워크북 규칙 시드 1에서 60일째 투입된 일반 로트 하나의 기록_

엔진의 이벤트 로그에서 로트 하나를 골라 그렸다. 투입부터 완성까지 46.2일, 583스텝 중 실제로 거친 것은 515스텝이다. 노광 35층, 습식 식각 `WE_FE_108`에는 52번 들렀다. 같은 줄로 몇 번이고 돌아오는 것이 재진입이다. 평균이 아니라 한 로트의 기록이다.

같은 결과를 Three.js 뷰어가 재생한다. 장비 1,043대를 상태색 블록으로 그리고 표식이 이 로트를 실제로 처리한 장비로 옮겨 간다. 실시간 렌더링이 아니라 재생으로 한 건 시뮬레이션과 렌더링 성능을 분리하고, 타임라인을 되감기 위해서다. 베이 배치는 연출이다. SMT2020에는 평면도가 없다.

## 9. 공구 마모: 데이터 25건 중 7건이 파생본이었다

KAMP CNC 데이터는 실험 25건이다. 그런데 19~25번은 각각 원본 하나와 가공 단계 순서가 행 단위로 같았다.

```python
def derived_from(runs: dict[int, pd.DataFrame]) -> dict[int, int]:
    """Each experiment that repeats an earlier one's machining phases row
    for row, mapped to the earliest such experiment."""
    parents: dict[int, int] = {}
    numbers = sorted(runs)
    for k, child in enumerate(numbers):
        phases = runs[child][PHASE].tolist()
        for parent in numbers[:k]:
            if parent not in parents and runs[parent][PHASE].tolist() == phases:
                parents[child] = parent
                break
    return parents
```

| 파생 실험 | 원본 | 라벨 | 원본 라벨 |
|---|---|---|---|
| 19, 22, 25 | 7 | worn | worn |
| 20 | 8 | unworn | worn |
| 21 | 2 | unworn | unworn |
| 23 | 5 | worn | unworn |
| 24 | 14 | unworn | worn |

7건 모두 길이와 가공 단계 순서가 원본 하나와 같고, 수치까지 같은 것은 24번뿐이다. 14번과 값이 모두 같은데 라벨이 반대다. 세 건(20·23·24)이 부모와 라벨이 다르다. 7건을 버리고 원본 18건(마모 10, 정상 8)만 썼다.

질문은 "축 센서가 가공 조건(이송·클램프)을 넘어 마모를 가르나"다. 실험 하나씩 빼는 LOO의 AUC는 이 크기에서 퇴화해서(라벨과 무관한 특징이 0.0) leave-pair-out AUC를 썼다. 유의성은 조건 층 안에서 라벨을 다시 붙이는 모든 경우 192가지를 열거하는 정확 순열 검정으로 봤다.

| 분석 | 조건만 AUC | 조건 + 센서 AUC | 개선 | 정확 p |
|---|---|---|---|---|
| 주 분석 (Layer 1 Up) | 0.538 | 0.575 | 0.037 | 0.18 |
| Layer 1 Up + Down | 0.538 | 0.575 | 0.037 | 0.20 |
| 전체 레이어 | 0.538 | 0.500 | −0.037 | 0.23 |

센서가 조건을 넘어선다는 증거는 없었다. 그리고 이 p도 확증이 아니다. 주 특징 세 개를 정하기 전에 설계 리뷰가 쌍 방향과 단일 특징 AUC를 먼저 봤다. 노출 후 지정이라는 것을 결과 옆에 적어 뒀다.

## 10. 한계를 결과보다 먼저 적었다

README의 "알려진 한계" 절은 첫날(9월 23일) 커밋에 이미 있었다. 그날 저장소에 있던 코드는 로더와 스모크 테스트뿐이고 분석 모듈은 하나도 없었다. 그때 9개였고, 지금은 18개다.

| 첫날 적은 것 (일부) | 나중에 붙은 것 (일부) |
|---|---|
| 데이터셋 간 조인 불가 | M3-B 가용률은 두 가정 위에 있다(처리량 = 투입 계획, 쉬는 장비도 수리 내내 빠진다) |
| SMT2020은 실측이 아니라 모델 파라미터 | SMT2020의 `_E` 변형과 LVHM 로트별 납기 시트는 읽지 않는다 |
| M3-B 부하율은 발견이 아니라 재구성 | M3-B 부하율은 정적 평균이라 줄 길이·CQT 위반·사이클타임을 말하지 못한다 |

먼저 적어 두면 결과를 쓸 때 문장의 범위가 거기에 묶인다. 그래도 넘친 적이 많았다. 작업 일지에는 "숫자는 다 맞았고 문장이 조건보다 넓었다"는 기록이 Q3 결과 리뷰에 있고, 같은 패턴이 Task 22·23·32에서 이미 반복됐다고 적혀 있다. 시드 하나로 규칙의 우열을 읽을 뻔했고, 네 번의 실행으로 "용량의 끝"이라고 썼다가 10시드에서 고쳤다.

프로젝트 전체 목표 8개 중 "모든 모듈이 베이스라인을 유의하게 이긴다"는 M1과 공구 마모에서 지키지 못했다. 그것도 결과로 적었다.

검증은 2026-10-07 기준 ruff·mypy 0건, pytest 1,524개 통과다. 노트북 10개를 다시 실행했고 원자료부터 계산하는 노트북과 계산 스크립트는 같은 `reports/` 파일을 냈다. DES 실험은 수 시간이 걸려 다시 돌리지 않았다.

## 11. 한계와 다음 단계

- **SMT2020은 테스트베드다.** 실측 로그가 아니고, 결론은 이 팹 모델 안의 것이다.
- **교차검증의 남은 3~7일을 재지 않았다.** 쉬는 장비의 다운타임 처리 차이로 추정만 했다. LVHM의 원래 차이(2.2일)가 HVLM(12.5일)보다 작은 이유도 밝히지 않았다. 시간 PM과 counter PM의 몫, 고장과 PM의 몫을 따로 떼지 않았다.
- **모델링하지 않은 것이 있다.** 초기 WIP(빈 팹에서 시작, 200일 워밍업), `SuperHotLotFIRST_and_Reservation`의 예약 부분, CQT 창 안 로트 우선, 반송 차량.
- **동적 재스케줄링은 없다.** 규칙을 바꿔 비교하는 정적 최적화까지만 했다. 강화학습 디스패처는 로드맵 기준 시뮬레이션 한 번이 20~30분이라 학습에 필요한 수백 번을 돌릴 수 없어 하지 않기로 했다.
- **Q4는 가정에 기댄다.** 이상률 4.98%, 정지 1시간, 양품 한 로트의 값이 모두 가정이고, 판정기인 시간순 M1이 우연보다 조금 나을 뿐이다.
- **다음은 비워 둔 칸이다.** 로드맵에는 웨이퍼 정렬 자동 보정(3D 장비와 카메라 시점), 시뮬레이터 위의 R2R, 이미지 기반 검사, 가상 계측이 순서대로 있다. 아직 설계 전이다.
