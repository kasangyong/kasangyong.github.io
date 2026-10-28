---
title: "반도체 제조 데이터 분석 시스템 VFAB: 이상감지부터 팹 시뮬레이션까지 6개 모듈"
date: 2026-10-23 10:30:00 +0900
categories: [Simulation, DES]
tags: [simpy, discrete-event-simulation, smt2020, pyscfabsim, semiconductor, dispatching, python]
math: true
image:
  path: /assets/img/posts/vfab-fab-simulation/crosscheck-light.png
  alt: VFAB 엔진과 PySCFabSim의 HVLM 사이클타임 비교
---

반도체 팹 테스트베드 SMT2020을 읽어 들이는 파서와 재진입·배치·셋업·PM·고장을 돌리는 이산사건 시뮬레이터를 SimPy 위에 직접 짰다. 기준 시뮬레이터 PySCFabSim과는 입력을 칸 단위로, 출력을 실행 단위로 대조했고, 그다음에 디스패칭 규칙과 장비 정지가 어떤 효과를 내는지 쟀다.

| 항목 | 값 |
|---|---|
| 팹 모델 | SMT2020 HVLM·LVHM. 라우트 583스텝, 툴그룹 106, 장비 1,043대, PM 292건 |
| 엔진 | SimPy + 직접 짠 장비 단위 디스패처 (`sim/engine.py` 963줄) |
| 입력 대조 | PySCFabSim 사본과 85,582칸 대조, 다른 칸 62개 |
| 엔진 대조 | 고장·PM을 끄면 HVLM 처리량 57.2 대 57.2 로트/일, 사이클타임 24.4 대 25.1일 |
| 디스패치 비교 | HVLM, 워크북 순위 대비: 셋업을 무시하는 규칙 3종이 하루 7.4~8.2로트(13~15%) 덜 만든다 |
| 실행 비용 | 단독 실행은 HVLM 1년, 코어 하나로 약 9분(README). 로드맵은 시뮬레이션 한 번을 20~30분으로 잡았다 |
| 검증 | pytest 1,524개 통과, ruff·mypy 0건 (2026-10-07 기준) |

{% include embed/video.html src='/assets/img/posts/vfab-fab-simulation/explainer.mp4' poster='/assets/img/posts/vfab-fab-simulation/explainer-poster.jpg' title='첫 PM을 장비들에 한꺼번에 주느냐 나눠 주느냐가 두 엔진의 사이클타임 차이를 만든다. 장비 4대 레인은 예시' loop=true %}

## 1. 무엇을 시뮬레이션하고 무엇을 안 했나

VFAB는 혼자 만든 반도체 제조 데이터 분석 프로젝트다. 모듈은 FDC 이상감지(SECOM), 웨이퍼맵 분류(WM-811K), OEE 추정(KAMP), 팹 구조 분석(M3-B), DES 시뮬레이션(M4) 다섯 개이고, 여기에 선택 모듈인 공구 마모 판별(KAMP CNC)이 붙는다.

SMT2020은 실측 로그가 아니고 시뮬레이션 모델 파라미터다. 라우트, 툴그룹별 장비 대수, 처리시간 분포, 셋업 행렬, PM과 고장 규칙이 엑셀 시트에 들어 있다. 그러니 "실제 팹 데이터를 분석했다"보다는 "업계 표준 팹 테스트베드로 모델링했다"고 말하는 게 정확하다. 실물 설비와 동기화되지도 않으니 디지털 트윈이라는 이름도 붙이지 않는다.

물류는 모델링하지 않았다. SMT2020이 주는 반송 모델은 아래 한 줄이 전부다.

| FROM | TO | 분포 | 평균 | offset | 단위 |
|---|---|---|---|---|---|
| Fab | Fab | uniform | 7.5 | 2.5 | min |

차량 대수나 경로, 배차 규칙을 넣으려면 전부 지어내야 한다. 그렇게 나온 결과는 테스트베드의 숫자가 아니라 내 가정 위의 숫자다. 그래서 반송은 이 분포 그대로 스텝 사이 지연으로만 넣었다.

## 2. SMT2020을 믿을 수 있게 읽기

### 단위부터 초로

SMT2020은 열마다 시간 단위가 min, hr, day로 제각각이다. 파서는 단위 열을 읽어 전부 초로 바꾸고, 단위가 비어 있거나 시간 단위가 아니면 거부한다.

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

파싱 결과는 `fab_params_{hvlm,lvhm}.json`이라는 계약 파일에 담아 M4로 넘기고, 읽는 쪽은 `validate_fab_params`로 먼저 검사한다. 여기서 형식, 중복·0·음수 같은 크기, 툴그룹·재작업 대상 스텝 같은 참조, 배치·cascade·CQT 규칙의 모순을 걸러 낸다. 다만 분을 초로 잘못 적은 값은 계약 검사로 못 잡기 때문에, 그 부분은 단위 열을 읽는 파서가 맡도록 나눴다.

### 기준 엔진의 사본과 칸마다 대조

PySCFabSim(MIT)은 SMT2020을 자기 형식의 탭 구분 파일로 바꿔서 갖고 있다. 두 사본의 값이 같으면 파서와 기준 엔진이 같은 의미로 읽었다고 볼 수 있고, 다르면 어느 한쪽에 오기가 있다. 입력부터 맞춰 두지 않으면 뒤의 엔진 비교는 서로 다른 팹끼리 비교하는 꼴이 된다.

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

라우트는 스텝마다 툴그룹, 처리 단위, 처리시간 분포·평균·offset, cascade 간격, 배치 최소·최대, 셋업, LTL, 재작업, 샘플링, CQT까지 16개 열을 짝지어 비교했고, PM은 종류, 간격, 수리 시간, 첫 발생까지 봤다. 라우트·PM·툴그룹·셋업·고장·투입·반송 표를 합쳐 모두 85,582칸을 비교했고(HVLM·LVHM) 그중 다른 칸은 62개였다.

| 다른 값 | 칸 수 | 처리 |
|---|---|---|
| `Route_Product_3`의 cascade 간격 | 30 × 2 시나리오 | 툴그룹마다 정해진 비에서 벗어나 xlsx 오기로 보이지만 원값을 둔다 |
| SuperHot 플래그 | 1 × 2 시나리오 | PySCFabSim은 이 열을 읽지 않는다 |

나중에 엔진에서 cascade 30개를 PySCFabSim 값(약 10배)으로 바꿔 돌려 봤는데, 사이클타임 차이는 HVLM −0.1일 [−0.7, 0.6]으로 구간이 0을 포함했다.

### 라인이 어떻게 생겼나

![재진입 히트맵](/assets/img/posts/vfab-fab-simulation/reentry.png)
_가장 많이 쓰이는 툴그룹 25개가 583스텝 중 어디서 다시 나오는지. 전공정(FE)은 주로 앞, 후공정(BE)은 약 3분의 2 지점부터_

한 로트는 583스텝을 지나는 동안 같은 툴그룹을 수십 번 다시 찾는다. 습식 식각은 52번이고 노광 스텝은 35개다. 재진입이 이렇게 많으니 한 툴그룹의 대기열에 공정 초반 로트와 후반 로트가 섞이고, 어느 로트를 먼저 넣느냐에 따라 팹 전체의 흐름이 달라진다.

## 3. 병목은 한 값으로 말하지 않았다

정적 부하율은 장비 대수와 처리시간에서 산술로 나오는 값이니 발견이 아니라 재구성이다. 그런데 산술만으로도 값이 하나로 정해지지 않는다. 시트에 없는 값이 두 개 있어서다.

- 배치를 얼마나 채우나. 확산로는 배치 최소·최대 크기만 있고, 실제로 몇 장을 넣고 시작할지는 디스패치가 정한다.
- 셋업을 얼마나 자주 치르나. 이온주입에는 셋업이 있는데, 몇 로트마다 바꿀지는 이것도 디스패치가 정한다.

그래서 이 두 값을 각각 양 끝으로 잡아 부하율을 범위로 냈다.

![툴그룹 부하율과 가정의 범위](/assets/img/posts/vfab-fab-simulation/toolgroup-load.png)
_검은 점: 꽉 찬 배치·셋업 없음. 네모: 배치를 최소로만 채움. 세모: 셋업을 매번 치름(상한)_

| 가정 | 시나리오 | 먼저 차는 곳 | 부하율 |
|---|---|---|---|
| 꽉 찬 배치, 셋업 없음 | HVLM | 노광기 `Litho_BE_110` | 0.974, 모든 툴그룹 1 미만 |
| 배치를 최소로만 채움 | HVLM | 확산로 `Diffusion_FE_101` | 0.991 |
| 배치를 최소로만 채움 | LVHM | 확산로 `Diffusion_FE_100` | 1.006 |
| 셋업을 매번 치름 | HVLM | 이온주입 `Implant_128` | 1.140 |

가정 하나만 바꿔도 병목이 노광기에서 확산로로, 다시 이온주입으로 옮겨 간다. 정적 분석으로는 이 셋 중 하나라는 데까지만 말할 수 있고, 어느 쪽인지는 디스패치를 실제로 돌려 봐야 안다. 그 답은 M4로 넘겼다.

## 4. 엔진: `simpy.Resource`를 쓰지 않은 이유

SimPy로 장비를 모델링할 때 가장 흔히 쓰는 방법은 툴그룹 하나를 `simpy.Resource(capacity=n)`로 두는 것인데, 이 팹에서는 그게 안 된다. 엔진 docstring에 적어 둔 이유를 옮기면 이렇다.

- Resource의 요청 큐는 FIFO라 워크북의 순위 규칙(우선순위, 최소 셋업, 임계율)으로 로트를 고를 수 없다.
- LTL(lot-to-lens) 규칙은 뒤 스텝을 앞 스텝을 처리한 바로 그 장비에 묶는다. Resource는 특정 장비를 지정할 수 없다.
- cascade 장비는 로트가 끝나기 전에 장비를 놓아준다.
- 배치 장비는 로트 여러 개를 한 번에 싣는다.

그래서 툴그룹을 장비 `count`대의 목록으로 두고 대기열에서 로트를 고르는 디스패처를 직접 짰다. PySCFabSim도 구조는 같다.

### 같은 순간에 도착한 로트는 함께 줄 세운다

디스패치 요청은 로트가 대기열에 오거나 장비가 빌 때 나간다. 그런데 요청 즉시 디스패치하면 같은 시각에 도착한 로트 둘 중 이벤트가 먼저 처리된 쪽이 순위와 상관없이 장비를 가져간다. 그래서 디스패치는 그 순간의 모든 이벤트가 끝난 뒤에 한 번만 돌린다. SimPy는 같은 시각의 이벤트를 우선순위(URGENT 0, NORMAL 1) 순으로 처리하므로, 그보다 낮은 우선순위 2짜리 이벤트를 따로 만들었다.

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

`_pending`은 툴그룹마다 예약이 한 번만 걸리게 막는다. 이 동작은 Task 31 리뷰 때 "같은 시각에 도착한 로트를 함께 줄 세운다"는 수정으로 들어갔다.

### 장비가 멈추면 위에 있던 작업이 밀린다

고장과 PM은 둘 다 `_down` 함수 하나로 처리한다. 장비 위에서 진행 중인 대기마다 `_Box`를 하나씩 두고, 수리가 들어오면 그 끝 시각을 수리 길이만큼 뒤로 민다.

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

SimPy의 `timeout`은 한 번 걸고 나면 끝 시각을 바꿀 수 없다. 그래서 끝 시각은 `_Box`에 따로 들고 있다가, 깨어날 때마다 그사이 밀렸는지 보고 남은 만큼 다시 잔다. SimPy에는 프로세스를 `interrupt()`로 깨우는 방법도 있다.

PySCFabSim과 갈리는 건 마지막 두 줄이다. PySCFabSim에서는 쉬는 장비에 고장이나 PM이 와도 아무 일이 없고 바쁜 장비의 작업만 밀린다. 우리 엔진은 쉬는 장비도 수리가 끝날 때까지 디스패치하지 않는다. M3-B에서 가용률을 계산할 때 쓴 해석("장비는 바쁘든 쉬든 수리 시간 내내 빠진다")을 엔진에도 그대로 가져왔다(차이는 5절).

### 공통 난수: 추첨은 쓰임새마다 따로

규칙 A와 B를 같은 시드로 돌려 비교하려면 같은 로트는 같은 처리시간을, 같은 장비는 같은 고장을 받아야 한다. 실행 전체가 난수 스트림 하나를 나눠 쓰면 이 조건이 깨진다. 디스패치가 바뀌면 이벤트 순서가 바뀌고, 그러면 같은 스트림에서 뽑힌 값이 다른 로트에게 돌아간다.

그래서 추첨마다 어디에 쓰이는지를 기준으로 이름 붙인 스트림을 따로 쓴다.

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

이 성질은 테스트로 고정했다. 로트가 지나가지 않는 다른 툴그룹에서 PM을 켰다 껐다 하면서 로트들의 처리시간이 하나도 바뀌지 않는지 본다.

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

처음부터 이 테스트로 충분했던 건 아니다. Task 33 1차 리뷰에서는 "규칙이 바뀌어도 로트의 추첨은 같다"는 핵심 전제를 확인하는 테스트가 없다는 지적을 받았다. 픽스처에 샘플링 1 미만, 재작업, 확률 반송이 없다 보니 그 추첨을 공유 스트림으로 되돌려도 테스트가 다 통과했다. 2차 리뷰에서는 내가 만든 "공유 스트림" 변이가 진짜보다 잡기 쉬운 변이였다는 게 드러났다. 리뷰어가 진짜 공유 스트림을 넣어 보니 대칭 픽스처에서 추첨 순서가 우연히 같아 4개가 살아남았다. 결국 리뷰를 네 번 받고서야 통과했다.

## 5. PySCFabSim 교차검증: 12.5일 차이의 출처

같은 SMT2020 입력을 두 엔진에서 365일씩 돌리고 200~365일 창을 잰다. 집계는 두 엔진 모두 같은 함수(`window_summary`)로 한다. PySCFabSim 자체 통계는 초기 WIP 로트까지 세기 때문에 쓰지 않았다.

| HVLM, 시드 10개 평균 | 처리량 (로트/일) | 사이클타임 (일) | 납기 준수 |
|---|---|---|---|
| 투입 계획 | 57.2 | | |
| PySCFabSim FIFO | 55.0 | 54.5 | 0.01 |
| VFAB FIFO_setup | 56.4 | 41.9 | 0.55 |

처리량은 비슷한데 사이클타임이 12.5일 다르다.

### 하나씩 맞추기를 여덟 번 했다

처음에는 차이를 낼 만한 요소를 하나씩 맞춰 봤다. 디스패치 순서, CQT 창 안 로트 우선, 배치 시작 조건, 배치 묶음, `min_runs`, cascade 값, 초기 WIP를 차례로 맞췄고, 마지막에는 셋(배치 시작·`min_runs`·cascade)을 한꺼번에 맞췄다. 효과는 대부분 하루 안팎이었다. 가장 컸던 배치 묶음은 PySCFabSim을 제품별로 묶었더니 오히려 +4.1일 늘어나 차이가 더 벌어졌다.

그다음에 고장과 PM을 양쪽 엔진에서 모두 끄고 돌려 봤다.

| 고장·PM 없음, HVLM, 시드 2개 | 처리량 | 사이클타임 | P90 |
|---|---|---|---|
| PySCFabSim FIFO | 57.2 | 24.4 | 31.3 |
| VFAB FIFO_setup | 57.2 | 25.1 | 32.2 |

처리량이 같아졌고 사이클타임 차이는 0.7일로 줄었다. 기본 흐름(재진입, 배치, 셋업, 디스패치)은 두 엔진이 같고 차이는 다운타임 처리에서 온다는 얘기다. 실험은 4회로 끝났다. 큰 덩어리를 먼저 끄고 그 안에서 범위를 좁히는 이분법을 처음부터 썼어야 했는데, 이 순서 실수는 작업 일지에 그대로 적어 뒀다.

### 범인은 첫 PM의 시점

SMT2020의 PM 첫 발생(`FOA`)은 전부 상수인데, 이 값을 툴그룹 안의 장비들에 어떻게 나눠 주느냐가 두 엔진에서 다르다.

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

명세는 `FOA × k / 대수`로 나누고, PySCFabSim은 장비 전부에 같은 첫 PM을 준다. 시간 기준 PM은 간격이 상수라서 한 번 같이 멈춘 장비들은 계속 같이 멈추고, 다음 PM도 수리 길이만큼씩만 어긋난다. 평균 가용률이 같더라도 그룹 전체가 한꺼번에 서 있는 동안 대기열이 쌓이고, 이게 풀리는 데 오래 걸린다.

한쪽 엔진만 바꿔 보고 내린 결론은 섣부르다. 실제로 우리 엔진에서만 "한꺼번에"로 바꿨더니 +19일이 나왔고, 차이(12.5일)의 1.5배였다. 그걸 보고 "주 원인"이라고 썼다가 리뷰에서 지적받았다. 그래서 양쪽 엔진에서 반대 방향으로도 바꿔 봤다.

![첫 PM 방식에 따른 사이클타임](/assets/img/posts/vfab-fab-simulation/crosscheck-light.png){: .light }
![첫 PM 방식에 따른 사이클타임](/assets/img/posts/vfab-fab-simulation/crosscheck-dark.png){: .dark }
_원래 설정은 시드 10개, 맞바꾼 쪽은 PySCFabSim 시드 2개, VFAB 시드 4개_

| | 사이클타임 (일) | 짝지은 차이 [95% 구간] |
|---|---|---|
| VFAB 나눔 → 한꺼번에 (시드 1~4) | 42.2 → 61.5 | +19.2 [18.2, 20.3] |
| PySCFabSim 한꺼번에 → 나눔 (시드 0~1) | 54.5 → 39.0 | −15.6 [−18.5, −12.7] |

PySCFabSim에서 첫 PM을 나누면 사이클타임이 우리 쪽 수준으로 내려오고, 우리 엔진에서 한꺼번에 주면 PySCFabSim 쪽으로 올라간다. LVHM에서는 이것 하나만 바꿨는데 PySCFabSim의 납기 준수가 0.30에서 1.00이 됐다(우리 0.97). 어느 엔진이 틀린 게 아니고 같은 데이터를 두 가지로 읽은 차이이며, 우리 엔진은 명세를 따랐다. SMT2020 결과끼리 비교할 때는 PM 첫 발생을 어떻게 읽었는지부터 맞추는 게 먼저다.

### 남은 3~7일

맞바꾼 뒤에도 두 방향 모두 우리 엔진이 길다. 나눌 때는 41.9 대 39.0, 한꺼번에 줄 때는 61.5 대 54.5다. 4절에서 말한 "쉬는 장비도 수리 중이면 못 쓴다"와 방향이 같고, 그룹이 한꺼번에 멈출 때 차이가 커지는 것도 이 설명과 맞는다. 다만 이 몫을 따로 재지는 않아서 추정으로 남겨 뒀다.

## 6. 디스패칭 규칙 비교: 공통 난수로 짝짓기

규칙 7개(워크북 순위, FIFO, EDD, 최소 여유, 그리고 셋이 각각 셋업을 먼저 보는 `_setup` 판)를 시드 10개, 두 시나리오로 140회 돌렸다. 공통 난수를 쓰니 같은 시드 안에서 "규칙 − 기준"은 대부분 규칙의 효과로 볼 수 있다. 구간은 이 시드별 차이로 계산한다.

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

`unpaired_sd`는 짝짓지 않았을 때 차이가 얼마나 퍼졌을지를 나타낸다. 둘의 비율을 보면 공통 난수가 실제로 얼마나 벌어 줬는지 알 수 있다.

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

마지막 줄은 기대와 달랐다. 구간이 좁았던 건 실행 하나가 165일 동안 9천 로트를 평균하기 때문이었다. 공통 난수는 분산을 줄여 줬다기보다 비교를 공정하게 만들어 줬다는 데서 쓸모가 있었다.

납기 준수율은 조심해서 읽어야 한다. SMT2020은 제품마다 납기 여유가 FIFO 계열의 평균 사이클타임과 거의 같아서, 규칙이 같아도 절반쯤은 늦게 되어 있다. 처음 결과를 쓸 때는 이걸 "투입을 못 따라가서"라고 잘못 적었다가 리뷰에서 고쳤다.

## 7. 장비 정지 시나리오: 오경보 한 번이 1시간 정지일 때

이 실험(Q4)은 프로젝트에서 데이터셋을 넘나드는 유일한 연결이다. SECOM과 SMT2020 사이에는 공통 키가 없으니 센서값을 그대로 옮길 수는 없다. 그래서 M1 FDC 판정기의 동작특성(재현율·오경보율 곡선)만 가져와 SMT2020 건식 식각 툴그룹 뒤의 검사 게이트에 넣었다.

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

경보가 나면 그 장비를 `hold_s`(1시간) 동안 세운다. 고장·PM과 같은 `_down`을 쓰기 때문에 장비 위 작업이 밀리는 방식도 똑같다. 이상률은 4.98%로 가정했다.

M1의 비용식은 오경보를 상수 비용으로 세지만 라인에서는 사정이 다르다. 정지가 툴그룹의 남는 시간 안에서 끝나면 공짜이고, 그 시간을 넘기면 양품을 잃는다. 이를 해석식으로 쓰면 툴그룹 용량은 다음과 같다.

$$
\text{capacity}(a) = \frac{n \cdot 86400 \cdot A}{t_{lot} + a \cdot t_{hold}}
$$

$n$은 장비 대수, $A$는 가용률, $t_{lot}$는 로트당 장비 시간, $a$는 경보율이다. 꺾이는 점은 이 용량이 팹의 산출 $X_0$ 아래로 내려가는 경보율이다.

$$
a_{knee} = \frac{n \cdot 86400 \cdot A / X_0 - t_{lot}}{t_{hold}}
$$

```python
def knee(self) -> float:
    """The alarm rate where the gate starts to cost lots."""
    per_lot = self.tools * 86400.0 * self.availability / self.output_per_day
    return (per_lot - self.lot_s) / self.hold_s
```

같은 판정기를 병목인 `DE_FE_54`(꺾임 0.24)와 여유가 있는 `DE_BE_12`(꺾임 0.55)에 각각 두었다.

![경보율과 잃는 양품](/assets/img/posts/vfab-fab-simulation/gate-light.png){: .light }
![경보율과 잃는 양품](/assets/img/posts/vfab-fab-simulation/gate-dark.png){: .dark }
_점은 DES, ×는 각 실행의 가용률을 넣은 해석식. HVLM 시드 1_

해석식은 DES 8회(기준 2회 포함)로 확인했다. 꺾임 위의 네 점은 DES와 0.02~0.30 로트/일 안에서 맞았고, 꺾임 아래인 여유 위치 0.3 점은 해석식이 0, DES가 −0.72였다. 점마다 시드가 하나뿐이라 구간은 없다. 각 실행의 가용률을 따로 넣은 데도 이유가 있는데, 경보가 늘면 웨이퍼 수 기준 PM이 줄어서 가용률이 오르기 때문이다.

| 비용비 | M1이 보고한 운용점의 경보율 | 병목 `DE_FE_54`에서 잃는 양품 | 여유 `DE_BE_12` |
|---|---|---|---|
| 5:1 | 0.05 | 0 (꺾임 아래) | 0 |
| 20:1 | 0.67 | 9.8 로트/일 | 1.8 로트/일 |
| 100:1 | 0.98 | 14.8 로트/일 | 5.9 로트/일 |

비용비 20:1에서 M1이 오프라인으로 고른 최적점은 병목 식각에서 하루에 양품 9.8로트를 잃는 자리였다. 같은 판정기라도 어느 장비 뒤에 두느냐에 따라 비용이 달라진다. 이 결론이 기대는 가정도 적어 둔다. 시간순 M1은 우연보다 조금 나은 정도이고, 정지 시간을 0.5시간으로 줄이면 여유 위치는 전수 경보도 견딘다. 이상률과 양품 한 로트의 값 역시 가정이다.

같은 엔진으로 장비 정지를 다른 각도에서도 따져 봤다.

| 실험 | 정지의 형태 | 결과 |
|---|---|---|
| Q2 증설 | 이온주입 세 그룹에 한 대씩 | 투입 계획 대비 부족분 0.8로트/일을 못 메웠다. 사이클타임 −1.6일. 밀어 넣으면 97% 넘게 쓰이는 그룹이 10개(노광·이온주입·건식 식각)라 병목을 한 곳으로 말할 수 없었다 |
| Q1 PM 정책 | 고정주기 PM 대 상태기반 정비 | 열화 모델은 합성(Weibull 수명, P-F 간격 5일). CBM이 이기려면 점검 1회 특이도 0.962 이상. 팹 처리량에서는 정책을 가르지 못했다(시드 2개) |

Q1에서 열화를 합성한 건 SMT2020의 고장이 무기억(지수분포)이기 때문이다. 무기억 고장에서는 언제 정비하든 고장률이 바뀌지 않으니, 그대로 두면 PM 정책을 비교하는 것 자체가 의미가 없다.

## 8. 로트 하나의 길

![로트 하나의 여정](/assets/img/posts/vfab-fab-simulation/lot-journey.png)
_대시보드 시뮬레이션 탭. HVLM 워크북 규칙 시드 1에서 60일째 투입된 일반 로트 하나의 기록_

엔진의 이벤트 로그에서 로트 하나를 골라 그렸다. 투입부터 완성까지 46.2일이 걸렸고, 583스텝 중 실제로 거친 건 515스텝이다. 노광은 35층, 습식 식각 `WE_FE_108`에는 52번 들렀다. 이렇게 같은 줄로 몇 번이고 돌아오는 게 재진입이다. 평균이 아니라 로트 하나의 기록이다.

같은 결과를 Three.js 뷰어로도 재생한다. 장비 1,043대를 상태색 블록으로 그리고, 표식이 이 로트를 실제로 처리한 장비를 따라 옮겨 간다. 실시간 렌더링 대신 재생 방식을 고른 건 시뮬레이션과 렌더링 성능을 떼어 놓고 타임라인을 되감을 수 있게 하려는 것이었다. 베이 배치는 연출이다. SMT2020에는 평면도가 없다.

## 9. 공구 마모: 데이터 25건 중 7건이 파생본이었다

KAMP CNC 데이터는 실험 25건으로 이뤄져 있는데, 19~25번은 각각 원본 하나와 가공 단계 순서가 행 단위로 같았다.

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

7건 모두 길이와 가공 단계 순서가 원본 하나와 같았고, 수치까지 같은 건 24번뿐이었다. 14번과 값이 모두 같은데 라벨이 반대다. 세 건(20·23·24)은 부모와 라벨이 달랐다. 그래서 7건은 버리고 원본 18건(마모 10, 정상 8)만 썼다.

질문은 "축 센서가 가공 조건(이송·클램프)을 넘어 마모를 가르나"였다. 실험을 하나씩 빼는 LOO의 AUC는 이 정도 크기에서 퇴화하기 때문에(라벨과 무관한 특징이 0.0) leave-pair-out AUC를 썼다. 유의성은 조건 층 안에서 라벨을 다시 붙이는 경우 192가지를 모두 열거하는 정확 순열 검정으로 봤다.

| 분석 | 조건만 AUC | 조건 + 센서 AUC | 개선 | 정확 p |
|---|---|---|---|---|
| 주 분석 (Layer 1 Up) | 0.538 | 0.575 | 0.037 | 0.18 |
| Layer 1 Up + Down | 0.538 | 0.575 | 0.037 | 0.20 |
| 전체 레이어 | 0.538 | 0.500 | −0.037 | 0.23 |

센서가 조건을 넘어선다는 증거는 없었다. 이 p도 확증으로 볼 수는 없는데, 주 특징 세 개를 정하기 전에 설계 리뷰에서 쌍 방향과 단일 특징 AUC를 먼저 봤기 때문이다. 그래서 노출 후 지정이라는 점을 결과 옆에 적어 뒀다.

## 10. 한계를 결과보다 먼저 적었다

README의 "알려진 한계" 절은 첫날(9월 23일) 커밋부터 있었다. 그날 저장소에 있던 코드는 로더와 스모크 테스트가 전부였고 분석 모듈은 하나도 없었다. 한계 항목은 그때 9개였고 지금은 18개다.

| 첫날 적은 것 (일부) | 나중에 붙은 것 (일부) |
|---|---|
| 데이터셋 간 조인 불가 | M3-B 가용률은 두 가정 위에 있다(처리량 = 투입 계획, 쉬는 장비도 수리 내내 빠진다) |
| SMT2020은 실측이 아니라 모델 파라미터 | SMT2020의 `_E` 변형과 LVHM 로트별 납기 시트는 읽지 않는다 |
| M3-B 부하율은 발견이 아니라 재구성 | M3-B 부하율은 정적 평균이라 줄 길이·CQT 위반·사이클타임을 말하지 못한다 |

한계를 먼저 적어 두면 결과를 쓸 때 문장이 그 범위 안에 묶인다. 그래도 넘친 적이 많았다. 작업 일지의 Q3 결과 리뷰에는 "숫자는 다 맞았고 문장이 조건보다 넓었다"는 기록이 있고, 같은 패턴이 Task 22·23·32에서 이미 반복됐다고 적혀 있다. 시드 하나만 보고 규칙의 우열을 읽을 뻔한 적도 있고, 실행 네 번만 보고 "용량의 끝"이라고 썼다가 10시드에서 고친 적도 있다.

프로젝트 전체 목표 8개 중 "모든 모듈이 베이스라인을 유의하게 이긴다"는 M1과 공구 마모에서 지키지 못했고, 이것도 결과로 적었다.

2026-10-07 기준으로 ruff·mypy는 0건, pytest는 1,524개가 통과했다. 노트북 10개도 다시 실행했는데, 원자료부터 계산하는 노트북과 계산 스크립트가 같은 `reports/` 파일을 냈다. DES 실험은 수 시간씩 걸려서 다시 돌리지 않았다.

## 11. 한계와 다음 단계

- SMT2020은 테스트베드다. 실측 로그가 아니니 결론도 이 팹 모델 안의 이야기다.
- 교차검증에서 남은 3~7일은 재지 않았고, 쉬는 장비의 다운타임 처리 차이 때문이라고 추정만 했다. LVHM의 원래 차이(2.2일)가 HVLM(12.5일)보다 작은 이유도 밝히지 않았고, 시간 PM과 counter PM의 몫이나 고장과 PM의 몫도 따로 떼어 보지 않았다.
- 초기 WIP(빈 팹에서 시작, 200일 워밍업), `SuperHotLotFIRST_and_Reservation`의 예약 부분, CQT 창 안 로트 우선, 반송 차량은 모델링하지 않았다.
- 동적 재스케줄링은 없고, 규칙을 바꿔 비교하는 정적 최적화까지만 했다. 강화학습 디스패처는 로드맵 기준으로 시뮬레이션 한 번에 20~30분이 걸려 학습에 필요한 수백 번을 돌릴 수 없으니 하지 않기로 했다.
- Q4는 가정에 기댄다. 이상률 4.98%, 정지 1시간, 양품 한 로트의 값이 모두 가정이고, 판정기로 쓴 시간순 M1도 우연보다 조금 나은 수준이다.
- 다음 단계는 아직 비어 있다. 로드맵에는 웨이퍼 정렬 자동 보정(3D 장비와 카메라 시점), 시뮬레이터 위의 R2R, 이미지 기반 검사, 가상 계측이 순서대로 올라 있지만 아직 설계 전이다.
