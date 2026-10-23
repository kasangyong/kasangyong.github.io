---
title: "공개 소스 12개를 매일 쌓는 Dagster 파이프라인: 백필이 오염이 되는 소스를 레지스트리로 막기"
date: 2026-10-23 10:10:00 +0900
categories: [Data, Pipeline]
tags: [python, dagster, github-actions, data-pipeline, backfill, idempotency, pytest]
image:
  path: /assets/img/posts/daily-dataset-pipeline/status-grid-light.png
  alt: 소스 12개 × 날짜별 파티션 최종 상태
---

환율, GitHub 신규 저장소, 해커뉴스, 위키백과 인기문서, 지진, 날씨, 팩트체크, arXiv, 임상시험, 암호화폐, 연합뉴스, GeekNews까지 공개 소스 12개를 매일 수집해 날짜 파티션으로 저장소에 커밋하는 파이프라인을 만들었다. Dagster 자산으로 정의하고 GitHub Actions cron으로 돌린다.

아래 수치는 2026-10-23 01:47 UTC 커밋(`3b0c863`) 기준이다.

| 항목 | 값 |
|---|---|
| 소스 | 12개 (매일 10, 매시간 요청 2) |
| 파티션 날짜 범위 | 2026-08-20 ~ 2026-10-23 |
| curated 파티션 파일 | 740개, 143,164행 |
| 저장 용량 | raw 85.3MB (gzip), curated 91.2MB (JSONL) |
| 매니페스트 | 1,669줄 (ok 1,440 / failed 229) |
| 최종 상태가 실패인 파티션 | 4개 (파일 없음 2, 파일은 있고 마지막 읽기 실패 2) |
| 코드 / 테스트 | 2,103줄 / 1,226줄, 133항목 통과 (네트워크 없음) |

![소스 12개 × 날짜별 파티션 최종 상태](/assets/img/posts/daily-dataset-pipeline/status-grid-light.png){: .light }
![소스 12개 × 날짜별 파티션 최종 상태](/assets/img/posts/daily-dataset-pipeline/status-grid-dark.png){: .dark }
_연한 칸은 0행으로 성공한 파티션이다. 임상시험의 주말이 대부분이다_

## 1. 소스마다 성질이 다르다

12개 소스는 같은 "하루치 데이터"처럼 보이지만 성질이 제각각이다. 어떤 소스는 끝난 하루를 언제든 다시 알려주고, 어떤 소스는 지금 이 순간만 알려준다. 어떤 피드는 11일치를 담고 어떤 피드는 90분치만 담는다. 이 차이를 무시하면 에러 없이 데이터가 틀린다.

| 소스 | 원천 | lag_days | 백필 | merge_key | 주기 |
|---|---|---|---|---|---|
| `fx_rates` | Frankfurter (ECB) | 1 | 가능 | | 일 |
| `github_repos` | GitHub Search API | 1 | 가능 | | 일 |
| `hn_stories` | HN Algolia | 1 | 가능 | | 일 |
| `wikipedia_top` | Wikimedia REST | 2 | 가능 | | 일 |
| `earthquakes` | USGS FDSN | 1 | 가능 | | 일 |
| `city_weather` | Open-Meteo archive | 1 | 가능 | | 일 |
| `fact_checks` | Snopes·FactCheck.org RSS + ClaimReview | 1 | 11일 이내 | `guid` | 일 |
| `arxiv_papers` | arXiv Atom API | 3 | 가능 | `arxiv_id` | 일, 최근 3일 재확인 |
| `clinical_trials` | ClinicalTrials.gov API v2 | 2 | 가능 | | 일 |
| `crypto_markets` | CoinGecko | 0 | 불가 | | 일 |
| `yna_news` | 연합뉴스 RSS | 0 | 불가 | `guid` | 시간 |
| `geeknews` | news.hada.io Atom | 0 | 불가 | `topic_id` | 시간 |

이 표의 오른쪽 네 열이 파이프라인 코드 대부분을 결정했다.

## 2. Dagster 자산과 파티션 레이아웃

### 왜 Dagster인가, 그런데 왜 스케줄러는 밖에 있나

수집 대상이 "매일 하루치가 쌓이는 데이터셋"이라 Dagster의 일별 파티션 자산과 개념이 그대로 맞는다. 백필이 프레임워크 기능이고, 자산별 재시도 정책을 선언으로 붙일 수 있다. 설계 문서에는 Prefect도 후보였지만 파티션과 백필을 직접 구현해야 해서 뺐다고 적혀 있다.

스케줄링은 Dagster 밖에 뒀다. Dagster 내장 스케줄러는 상주 데몬을 전제로 하는데 GitHub Actions 러너는 매번 새 VM이다. 그래서 cron이 Python 드라이버(`pipeline/cli.py`)를 부르고, 드라이버가 Dagster의 `materialize`를 호출한다. 부작용으로 실행 이력이 매번 사라지므로, 이력은 `data/_manifest.jsonl`에 따로 남겨 데이터와 함께 커밋한다.

### 파티션 = KST 달력 하루

```python
KST = ZoneInfo("Asia/Seoul")
START_DATE = "2026-08-01"

# Completed days only. A source that reports a finished day has nothing to
# say about today until today ends.
DAILY = DailyPartitionsDefinition(start_date=START_DATE, timezone="Asia/Seoul")

# Includes the day in progress, for sources that only ever report "now".
# Without end_offset Dagster rejects today's key as not yet a partition.
DAILY_OPEN = DailyPartitionsDefinition(
    start_date=START_DATE, timezone="Asia/Seoul", end_offset=1
)
```

정의가 둘인 이유는 `DailyPartitionsDefinition`이 끝나지 않은 날을 유효한 파티션으로 보지 않기 때문이다. `lag_days=0`인 스냅샷 소스(`crypto_markets`, `yna_news`, `geeknews`)는 정의상 오늘 파티션에 써야 하니 `end_offset=1`을 준 `DAILY_OPEN`을 쓰고, 나머지 9개는 `DAILY`를 쓴다.

시각으로 거르는 소스는 KST 자정 경계를 명시적으로 넘긴다. GitHub 검색에 맨 날짜(`created:2026-08-25`)를 주면 UTC 하루로 해석돼 파티션 라벨과 9시간 어긋난다.

```python
def day_bounds_iso(dt: str) -> tuple[str, str]:
    """Inclusive ISO-8601 bounds of a KST calendar day, with offset."""
    return f"{dt}T00:00:00+09:00", f"{dt}T23:59:59+09:00"
```

테스트가 이 쿼리 문자열을 고정한다. `"created:2026-08-25T00:00:00+09:00..2026-08-25T23:59:59+09:00"`

### 파티션 키가 곧 파일 경로다

```
data/
  raw/<source>/dt=YYYY-MM-DD.json.gz    API 응답 원본
  curated/<source>/dt=YYYY-MM-DD.jsonl  정규화·검증된 레코드
  _manifest.jsonl                        실행 이력 (append-only)
```

같은 날짜를 몇 번 돌려도 그 날짜 파일 하나만 덮어쓴다(병합 소스의 raw 제외). 중복이 쌓일 경로가 구조적으로 없다. 매니페스트만 누적되고, 재실행 이력 자체가 정보로 남는다.

raw를 따로 두는 이유는 과거를 다시 주지 않는 API 때문이다. HN 순위나 CoinGecko 시세는 정규화 버그가 나도 다시 받을 수 없으니, 원본이 유일한 복구 수단이다. 매일 새 파일이라 diff가 의미 없으므로 gzip으로 압축한다.

```python
def write_raw(source: str, dt: str, payload: Any, part: str | None = None) -> int:
    path = raw_path(source, dt, part)
    path.parent.mkdir(parents=True, exist_ok=True)
    blob = json.dumps(payload, ensure_ascii=False, separators=(",", ":")).encode("utf-8")
    with gzip.open(path, "wb", compresslevel=6) as fh:
        fh.write(blob)
    return path.stat().st_size
```

curated는 `sort_keys=True`로 쓴다. 키 순서가 고정돼야 같은 레코드가 같은 줄이 되고 diff가 의미를 가진다. 윈도에서 개발하고 리눅스 CI에서 쓰기 때문에 `.gitattributes`에 `* text=auto eol=lf`를 걸었다. 안 걸면 매일 커밋이 건드린 파일 전체를 줄바꿈만 바꿔 다시 쓴다.

### 모든 소스가 같은 수집 루틴을 탄다

`fetch → raw 저장 → normalize → pydantic 검증 → curated 저장 → 매니페스트 기록`. 이 순서는 `common/collect.py` 한 곳에 있고, 소스 파일은 `fetch`와 `normalize` 두 함수만 쓴다.

```python
payload = fetch(dt)
# Stamped here rather than after normalization: this is the instant the
# counts in the payload were true.
observed = datetime.now(timezone.utc)
collected_at = observed.strftime("%Y-%m-%dT%H:%M:%SZ")
# ... 생략 (raw 저장)

candidates = normalize(payload, dt)
if not candidates and not allow_empty:
    raise EmptyPayloadError(f"{source} returned no records for {dt}")

valid: list[dict[str, Any]] = []
invalid = 0
for candidate in candidates:
    try:
        stamped = {**candidate, "collected_at": collected_at}
        valid.append(model.model_validate(stamped).model_dump())
    except ValidationError as exc:
        invalid += 1
        # ... 생략 (앞의 3건만 로그)

if candidates and invalid / len(candidates) > MAX_INVALID_SHARE:
    raise SchemaDriftError(
        f"{source} {dt}: {invalid}/{len(candidates)} records failed validation"
    )
```

결정 세 가지가 들어 있다.

- **`collected_at`은 수집기가 찍는다.** `stars`나 `points`는 수집 순간에 한 번 잰 값이라 관측 시각 없이는 해석이 안 된다. 게시 6시간 뒤의 360점과 5일 뒤의 360점은 다른 사실이다. 정규화 함수마다 챙기게 두면 하나는 빠뜨린다.
- **0건은 기본이 실패다.** 0건이 정상인 소스만 `ALLOW_EMPTY = True`를 선언한다. 테스트가 "항상 데이터가 있는 `fx`는 이 상수를 갖지 않는다"까지 확인한다.
- **검증 실패가 절반을 넘으면 실패다** (`MAX_INVALID_SHARE = 0.5`). 상류 스키마가 바뀐 신호이고, 반쯤 빈 파티션을 조용히 쓰는 것보다 크게 실패하는 쪽이 낫다. 실패하면 curated를 새로 쓰지 않는다. raw는 남는다.

## 3. 레지스트리: 조용한 오염을 필드로 막는다

소스 카탈로그는 `pipeline/registry.py` 하나다. 이름과 자산 외에 필드 다섯 개를 더 갖는데, 전부 "무시하면 데이터가 조용히 틀리는" 성질이다.

```python
@dataclass(frozen=True)
class Source:
    name: str
    title: str  # Korean display name, used in the catalog
    asset: AssetsDefinition

    # Days between the partition date and the day the run happens. 1 means a
    # run collects yesterday, once that day is complete. 0 means the source has
    # no completed-day form -- it is a snapshot of the moment it is read.
    lag_days: int = 1

    # False when the source can only report its current state. Backfilling such
    # a source would file today's numbers under a past date.
    backfillable: bool = True

    # When set, a partition is merged rather than overwritten, keyed on this
    # field. Feeds that hold only a short window need many reads to cover a day.
    merge_key: str | None = None

    # Hourly rather than daily. Follows from a short feed window.
    hourly: bool = False

    # How many recent days to collect again on every scheduled run, whether or
    # not they already succeeded.
    # ... 생략 (나머지 주석)
    recheck_days: int = 0
```

### 백필이 실패가 아니라 오염이 되는 소스

CoinGecko의 `coins/markets`는 일별 종가가 아니라 현재 시세를 준다. 이 소스를 `--start 2026-08-01`로 백필하면 API는 정상 응답하고, 파이프라인은 성공하고, 오늘 시세가 8월 1일 라벨을 달고 저장된다. 실패보다 나쁘다. 실패는 눈에 띄지만 이건 안 띈다.

그래서 백필 계획 단계에서 거른다.

```python
for dt in date_range(start, end or start):
    if not source.backfillable and dt != partition_for(source.lag_days):
        # Writing a live snapshot under a past date would invent data.
        annotate("warning", f"{name} cannot be backfilled; skipping {dt}")
        continue
    plan.append((dt, name))
```

같은 규칙이 자동 복구(5절)에도 들어간다. 과거 구멍을 메우려고 스냅샷 소스를 다시 읽으면 복구가 곧 오염이다.

### 레지스트리 불변식은 테스트로 강제한다

필드를 잘못 조합하면 다시 조용한 오염이 생긴다. 그래서 조합 규칙을 테스트로 고정했다.

```python
def test_snapshot_sources_are_marked_unbackfillable():
    # A source with no completed-day form cannot be filled in after the fact.
    for source in SOURCES:
        if source.lag_days == 0:
            assert not source.backfillable, f"{source.name} would invent past data"


def test_hourly_sources_merge_rather_than_overwrite():
    # Reading a partition many times a day and overwriting would keep only the
    # final read.
    for name in HOURLY_SOURCES:
        assert BY_NAME[name].merge_key, f"{name} would discard earlier reads"


def test_re_read_sources_merge_so_a_thin_read_cannot_erase_a_full_one():
    # Overwriting would let a transient half-empty response replace a complete
    # partition. Merging on a natural key can only add.
    for source in SOURCES:
        if source.recheck_days:
            assert source.merge_key, f"{source.name} would overwrite on re-read"
```

README의 데이터셋 표에 모든 소스의 `title`과 id가 들어 있는지도 검사한다. 카탈로그는 사람이 제일 먼저 읽는 문서라 여기서 어긋나도 조용하다.

### 11일 창 밖은 소스가 거부한다

`fact_checks`는 백필이 "가능"과 "불가" 사이에 있다. RSS가 약 11일치를 담으니 그 안은 백필이 되고 그 밖은 피드가 답을 모른다. 이때 빈 파티션을 쓰면 "그날 검증된 주장이 없었다"로 읽히지만 실제로는 "알 수 없다"다. 둘은 다른 사실이라 소스가 직접 실패시킨다.

```python
oldest = min(
    _published_kst(item) for body in feeds.values() for item in _entries(body)
)
if date.fromisoformat(dt) < oldest:
    raise FeedWindowError(
        f"{dt} predates the feed window, which reaches back to {oldest.isoformat()}"
    )
```

반대로 임상시험의 주말 0건은 "없었다"가 정답이다. 등록부가 토·일에 게시하지 않는다(설계 당시 실측 토·일 0건, 평일 214~241건). 이걸 실패로 올리면 매주 두 번 빨간불이 뜨고, 결국 경보를 무시하게 된다. `ALLOW_EMPTY = True`로 선언하고 매니페스트의 `rows: 0`으로 남긴다. 운영 기록에서 0행 파티션은 23개이고 그중 20개가 임상시험이다. 첫 주말(08-22, 08-23)은 이 선언 전이라 `EmptyPayloadError`로 실패했다가 나중에 0행 성공으로 바뀌었다. 0행인 평일은 09-07과 10-12 두 번인데, 둘 다 미국 연방 공휴일(노동절, 콜럼버스 데이)과 겹친다.

## 4. lag_days는 실측으로 정했다

`lag_days`는 실행일과 파티션 날짜의 차이다. 처음에는 모든 소스가 D-1(KST 전날)을 채웠다. 소스마다 기준일이 다르면 날짜로 조인할 수 없기 때문이다. 셋이 실측으로 벗어났다.

| 소스 | lag | 근거 |
|---|---|---|
| `wikipedia_top` | 2 | UTC 하루 단위로 집계하고 몇 시간 뒤 발행한다. 21:00 UTC 실행 시점에는 그 UTC 날짜가 끝나지도 않았고, 끝난 직후 00:25 UTC에도 404였다 |
| `clinical_trials` | 2 | `StudyFirstPostDate`가 미 동부 날짜다. 06:00 KST는 미 동부로 전날 17시라 그날이 아직 진행 중이다 |
| `arxiv_papers` | 3 | 색인이 며칠에 걸쳐 채워진다. **잠정치다.** 안정 소요일을 재다가 arXiv 429에 막혀 끝까지 재지 못했다 |

```python
# Pageviews are aggregated per UTC day and published a few hours after
# that day closes. At 21:00 UTC the run's own UTC day has not even ended,
# so D-1 is never available -- measured as a 404 on the scheduled run.
Source("wikipedia_top", "위키백과 인기문서", wikipedia.wikipedia_top, lag_days=2),
```

매니페스트에 첫 404가 그대로 남아 있다. `dt=2026-08-26`, `run_at=2026-08-27T00:25:04Z`, 그 뒤 재시도 세 줄. 위키 쪽 `dt`는 KST가 아니라 UTC 날짜라는 점도 문서와 모듈 docstring에 적었다. KST 소스와 9시간 어긋나 추세 비교는 되지만 정확한 조인은 안 된다.

lag 2로도 매번 충분하지는 않았다. 10-13, 10-16, 10-17 파티션은 예정일에 또 404였다. 세 개 모두 다음 날 이후 자동 복구가 다시 읽어 최종 성공으로 끝났다. lag는 "보통 이때쯤 있다"는 추정이고, 그 추정이 틀린 날은 복구가 받친다.

`lag_days`가 소스마다 갈라진 뒤 생긴 버그도 있다. 5절의 종료 코드 판정이 그것이다.

## 5. 짧은 창 피드는 병합한다

연합뉴스 RSS는 최근 120건을 담는데, 설계 당시 실측으로 그 120건이 88분 범위였다. 하루 1,440분 중 88분이면 하루 한 번 읽어서는 6% 남짓만 남는다(README의 약 2%와 달리 88/1440로 계산). 그래서 매시간 읽고, 같은 날 파티션에 `guid` 기준으로 합친다.

```python
def _merge(
    source: str, dt: str, fresh: list[dict[str, Any]], key: str
) -> list[dict[str, Any]]:
    """Union a partition with what it already holds; the first sighting wins.

    A feed that only exposes the last couple of hours needs many reads to cover
    one day. Overwriting would keep just the final read; appending blindly
    would duplicate every item the feed still lists.

    An item already stored keeps its original record, so ``collected_at`` marks
    when the item was first seen rather than the last read that still listed it.
    """
    merged = {record[key]: record for record in storage.read_curated(source, dt)}
    for record in fresh:
        merged.setdefault(record[key], record)
    return sorted(merged.values(), key=lambda r: (_event_time(r), r[key]))
```

`setdefault`라서 먼저 본 레코드가 이긴다. 덮어쓰면 마지막 읽기만 남고, 그냥 이어 붙이면 피드에 아직 남아 있는 항목이 매번 중복된다. 병합 소스는 raw도 읽기마다 `raw/<source>/dt=<날짜>/<마이크로초 타임스탬프>.json.gz`로 따로 남긴다. 덮어쓰면 raw로 마지막 읽기 하나만 복원할 수 있어 raw를 두는 이유가 사라진다.

GeekNews도 매시간이다. 설계 당시 실측으로 피드 50건이 약 40.9시간을 덮으니 지금은 하루 한 번으로 충분하다. 그런데 창의 크기가 시간이 아니라 **건수**라 글이 몰리면 창이 짧아진다. 매시간 병합하면 게시량과 상관없어진다.

### 실제로는 매시간 읽히지 않았다

![yna_news 읽기 간격 분포](/assets/img/posts/daily-dataset-pipeline/feed-intervals-light.png){: .light }
![yna_news 읽기 간격 분포](/assets/img/posts/daily-dataset-pipeline/feed-intervals-dark.png){: .dark }
_cron 변경(08-27 03:31 UTC) 이후 매니페스트의 성공 기록 시각으로 잰 간격. 5분 안에 붙은 기록은 한 번의 읽기로 합쳤다_

설계는 매시간이었다. 초기에 GitHub이 시간당 요청을 49~174분 간격으로 실행했고, 2026-08-27에 연합뉴스 4시간치가 비었다. 그래서 cron을 `*/30 * * * *`로 바꿔 30분마다 요청하게 했다.

효과가 없었다. cron을 바꾼 커밋(08-27 03:31 UTC) 이후만 매니페스트로 재면 yna_news 성공 읽기 사이 간격 306개의 중앙값이 257분이고, 90분 안에 들어온 건 0개다. 최대는 875분이다. Actions 실행 기록으로 봐도 cron 변경 이후 첫 실행(08-27 15:03)부터 10-23 01:46 UTC까지의 schedule 실행은 370회, 하루 약 6.55회다. 요청은 하루 48회(30분)에 일별 1회를 더한 49회였다. 취소 기록이 없으니 concurrency 때문에 버려진 실행도 아니다. GitHub이 부하 시 스케줄 실행을 지연·누락한다는 점(워크플로 주석)을 보면, 요청 자체가 실행되지 않은 것으로 보인다.

`collected_at` 간격으로 어느 구간이 비었는지 사후 판별은 할 수 있다. 하지만 무료 러너에서 90분 이내 실행을 보장할 방법은 없다. 피드형 소스를 계속 쓰려면 상주 스케줄러가 필요하다. 이 부분은 해결하지 못한 채 한계로 남겼다.

## 6. 자동 복구와 종료 코드

### 구멍은 다시 보지 않으면 영구적이다

매일 실행은 그날 몫만 채운다. 그러면 일시적인 상류 장애로 한 번 실패한 파티션은 아무도 다시 시도하지 않는다. 실제로 2026-08-27 팩트체크(factcheck.org 일시 403)가 그렇게 굳었고, arXiv는 08-28 이후 계속 비어 있었다.

그래서 명시적 날짜 없이 실행하면 최근 7일을 훑어 구멍을 다시 시도한다.

```python
HEAL_DAYS = 7

def find_gaps(names: list[str], days: int = HEAL_DAYS) -> list[tuple[str, str]]:
    gaps: list[tuple[str, str]] = []
    for name in names:
        source = BY_NAME[name]
        if not source.backfillable:
            # A snapshot source cannot be repaired after the fact; retrying it
            # would file today's numbers under a past date.
            continue

        due = date.fromisoformat(partition_for(source.lag_days))
        for back in range(1, days + 1):  # the due day itself is the normal plan
            dt = (due - timedelta(days=back)).isoformat()
            if dt < START_DATE:
                continue
            healthy = (
                manifest.latest_status(dt).get(name) == "ok"
                and storage.curated_path(name, dt).exists()
            )
            if not healthy:
                gaps.append((dt, name))
    return gaps
```

"건강한 파티션"의 정의가 핵심이다.

| 매니페스트 최종 상태 | 파일 | 판정 |
|---|---|---|
| `ok` | 있음 (0행 포함) | 건강 |
| `ok` | 없음 | 구멍 |
| `failed` | 상관없음 | 구멍 |
| 기록 없음 | 없음 | 구멍 |

0행 성공을 구멍으로 보지 않는 이유는 임상시험 주말이다. 구멍으로 보면 매주 이틀을 7일 동안 무의미하게 다시 읽는다. 파일이 사라진 성공을 구멍으로 보는 테스트(`test_a_success_whose_file_vanished_is_a_gap`)도 따로 있다.

### 성공했지만 부족한 파티션

복구는 실패한 파티션만 본다. arXiv에서 이 전제가 깨졌다. 색인이 하루치를 한 번에 넣지 않고 며칠에 걸쳐 채우기 때문에, 다음 날 아침 읽은 파티션은 41~77건으로 "성공"하고 며칠 뒤 백필한 날은 159~347건이었다. `total == fetched`를 확인했으니 우리가 자른 게 아니라 그 시점 arXiv가 정말 그만큼만 갖고 있었다. 41건으로 성공한 파티션은 건강하다고 판정되니 영원히 41건으로 남는다.

대응은 두 겹이다. lag를 3일로 늘려 덜 채워진 상태로 읽을 확률을 줄이고, `recheck_days=3`으로 최근 3일을 성공 여부와 무관하게 다시 읽는다. 다시 읽기는 `arxiv_id` 병합이라 일시적으로 적게 와도 이미 있는 행을 지우지 못한다.

![arXiv 파티션별 첫 성공 행 수와 최종 행 수](/assets/img/posts/daily-dataset-pipeline/arxiv-recheck-light.png){: .light }
![arXiv 파티션별 첫 성공 행 수와 최종 행 수](/assets/img/posts/daily-dataset-pipeline/arxiv-recheck-dark.png){: .dark }
_진한 막대가 처음 성공했을 때, 연한 막대가 마지막으로 다시 읽은 뒤다_

60개 파티션 중 44개가 두 번 이상 성공적으로 읽혔고 그중 41개가 늘었다. 다만 08-28~08-30은 재확인이 아니라 수동 백필로 늘었다(커밋 `1e40c9a`, 59/43/133 → 308/232/169). 첫 성공이 100건 미만이던 파티션이 14개였는데 최종까지 100건 미만으로 남은 건 2개(09-19 45건, 09-26 62건)다. 09-29는 폭주 방지 상한 `MAX_RESULTS = 800`에 닿았다. 상한에 닿으면 자르되 경고 로그를 남긴다.

### 상류가 거부 중이면 더 두드리지 않는다

```python
# The day's own partitions go first, so a source that is refusing us can be
# spotted before the second look piles more requests onto it.
for dt, name in plan:
    # raise_on_error=False so one dead API cannot abort the rest of the
    # matrix; the failure is already recorded in the manifest.
    result = materialize([BY_NAME[name].asset], partition_key=dt, raise_on_error=False)
    outcomes.setdefault(dt, {})[name] = result.success
    # ... 생략 (요약 표 기록)
    if not result.success:
        refusing.add(name)
        # ... 생략 (경고 출력)

for dt, name in healing or []:
    if name in refusing:
        # Today's read already failed. Three more would deepen a rate limit,
        # not escape it; tomorrow's run will try again.
        annotate("notice", f"{name} {dt}: skipped, source is failing today")
        # ... 생략 (요약 표 기록)
        continue
    result = materialize([BY_NAME[name].asset], partition_key=dt, raise_on_error=False)
    # ... 생략 (결과는 요약 표와 로그에만 남고 outcomes에 들어가지 않는다)
```

오늘 몫을 먼저 돌리고, 오늘 실패한 소스는 그날 복구와 재확인에서 뺀다. 진단 중 실제로 arXiv에 429를 맞았고, 그 상태에서 재확인 3일치를 더 보내면 요청이 4배가 돼 차단만 깊어진다.

이 가드에는 대가가 있다. 9월 말 arXiv가 길게 막혔다.

| 실행일 (UTC) | arXiv 예정 파티션 | 결과 |
|---|---|---|
| 09-22 ~ 09-27 (6회) | 09-20 ~ 09-25 | 전부 406. 복구·재확인 건너뜀 |
| 09-29 | 09-26 | 성공 62건. 복구로 09-21 ~ 09-25 메움, 09-20은 429 |
| 09-30 ~ 10-03 00:08 (4회) | 09-27 ~ 09-30 | 전부 실패 (429·503·연결 오류). 복구·재확인 건너뜀 |
| 10-03 23:30 | 10-01 | 성공. 복구로 09-27 ~ 09-30 메움 |

09-20은 09-29 실행에서 429를 맞았고, 다음 실행은 오늘 몫이 실패해 복구를 건너뛰었고, 그다음 실행 때는 7일 창 밖이었다. 그렇게 **영구 구멍**이 됐다. 09-19(45건)와 09-26(62건)이 얇게 남은 것도 같은 이유다. 재확인 기회 세 번이 모두 "오늘 실패한 날"에 걸렸다. 창 7일과 재확인 3일은 장애가 그보다 짧다는 가정 위에 있고, 이번 장애는 그 가정보다 길었다.

### 종료 코드는 "아무것도 못 모았을 때"만 1이다

```python
# The alarm is for a run that came back empty-handed -- every source down,
# nothing collected, nobody told. It deliberately does not fire per day:
# sources target different days now, so a day can hold a single source, and
# that one source failing is not an outage.
attempts = [ok for results in outcomes.values() for ok in results.values()]
if attempts and not any(attempts):
    annotate("error", "this run collected nothing: every source failed")
    return 1
return 0
```

- 소스 하나가 실패해도 나머지가 모였으면 0이다. 워크플로의 커밋 단계는 `if: always()`라 모인 데이터는 커밋된다.
- 복구와 재확인의 실패는 종료 코드에 넣지 않는다. 이미 깨진 걸로 아는 과거를 다시 찔러보는 것이라, 그게 또 실패했다고 잡을 빨갛게 만들면 상류가 살아날 때까지 매일 빨간불이다.
- 이번 실행의 오늘 몫이 전부 실패했을 때만 1이다.

처음 구현은 **날짜별**이었다. "그 날짜의 모든 소스가 실패했으면 1". 모든 소스가 D-1을 채우던 시절에는 맞았다. lag가 갈라진 뒤에는 `lag_days=3`인 arXiv가 자기 날짜 칸에 혼자 앉게 됐고, arXiv 하나가 실패하면 "그날 100% 실패"가 됐다. 2026-09-08에 나머지 9개가 정상인데 빌드가 빨간불이 됐다. 이 회귀는 테스트로 고정했다.

```python
def test_a_lone_source_failing_on_its_own_day_does_not_fail_the_run(fake_materialize):
    # arxiv_papers is the only source with a three-day lag, so it sits alone on
    # its partition date. Grouping the alarm by day made that one failure look
    # like a total outage and turned the build red while the data was fine.
    fake_materialize({"arxiv_papers": False})
    names = ["arxiv_papers", "fx_rates", "hn_stories"]

    assert cli.run(cli.build_plan(names, None, None)) == 0
```

`--start`를 주면 복구·재확인을 돌리지 않는다. 종료 코드 규칙은 같다(`test_one_dead_day_in_a_backfill_fails_the_run`은 이름과 달리 실제로는 범위 전체가 실패하는 경우를 검사한다).

## 7. 매니페스트: 순서가 아니라 시각으로 읽는다

로컬 실행과 CI 실행이 둘 다 매니페스트에 줄을 추가하면 rebase에서 충돌이 난다. 양쪽 줄이 다 필요한 append-only 파일이라 git의 union 병합 드라이버를 걸었다.

```
# Append-only: a run here and a run in CI both add lines, and every line is
# wanted. Git's union driver keeps both sides instead of raising a conflict.
data/_manifest.jsonl merge=union
```

union 병합은 두 쪽 줄의 시간순을 보장하지 않는다. 그래서 "마지막 줄이 최신"이 깨진다. 파일 순서대로 읽으면 오래된 실패가 새 성공을 덮어 복구가 멀쩡한 파티션을 또 읽는다. 최신 판정을 `run_at` 정렬로 바꿨다.

```python
def latest_status(dt: str) -> dict[str, str]:
    result: dict[str, str] = {}
    for entry in sorted(read_all(), key=lambda e: e.get("run_at", "")):
        if entry.get("dt") == dt:
            result[entry["source"]] = entry["status"]
    return result
```

매니페스트 한 줄은 이렇게 생겼다.

```json
{"bytes_curated": 3302, "bytes_raw": 319, "dt": "2026-08-25", "duration_s": 0.5, "error": null, "invalid_rows": 0, "rows": 29, "run_at": "2026-08-26T02:43:50Z", "source": "fx_rates", "status": "ok"}
```

재시도도 한 줄씩 남는다. Dagster `RetryPolicy(max_retries=3, delay=5, backoff=EXPONENTIAL)`라 한 번 실패한 파티션은 최대 4줄이다. 4절의 위키 404 사례에서 네 줄의 `run_at`은 00:25:04, 00:25:09, 00:25:24, 00:26:00이다. 재시도 간격이 5초, 15초, 36초로 늘어난다.

## 8. 두 달 동안 무엇이 실패했나

![소스별 실패 원인](/assets/img/posts/daily-dataset-pipeline/failures-light.png){: .light }
![소스별 실패 원인](/assets/img/posts/daily-dataset-pipeline/failures-dark.png){: .dark }
_재시도를 포함한 줄 수라 한 번의 실패가 최대 4줄로 잡힌다_

failed 229줄 중 151줄이 arXiv다. 429 요청 제한, 406, 503, 연결 오류, 그리고 색인이 아직 안 찬 날의 빈 응답이 섞여 있다. 나머지 소스의 실패는 대부분 같은 실행의 재시도나 다음 실행·다음 읽기에서 풀렸다(임상시험 첫 주말은 코드 수정으로). 최종까지 실패로 남은 파티션은 네 개다.

| 파티션 | 상태 | 이유 |
|---|---|---|
| `arxiv_papers` 09-20 | 파일 없음 | 6절의 9월 말 장애. 7일 창 밖으로 밀려남 |
| `crypto_markets` 09-30 | 파일 없음 | CoinGecko 403. 스냅샷 소스라 복구 대상이 아님 |
| `yna_news` 09-29, 10-14 | 파일 있음, 마지막 읽기 실패 | 피드 XML 파싱 오류(`not well-formed`). 그 전 읽기들이 병합한 354행, 331행은 남아 있음 |

마지막 줄은 매니페스트 해석의 함정을 보여준다. 병합 소스의 "최종 상태"는 파티션의 상태가 아니라 마지막 **읽기**의 상태다. 스냅샷 소스는 복구 대상이 아니라 동작에는 영향이 없지만, 매니페스트만 보고 "그날 연합뉴스가 없다"고 읽으면 틀린다.

## 9. 팩트체크: 판정은 RSS가 아니라 ClaimReview에 있다

"거짓 기사만 모은 피드"는 없다. 있는 건 **검증 기록**이다. 누군가 한 주장과 검증 기관의 판정. 그런데 Snopes와 FactCheck.org RSS에는 제목·링크·주제 태그뿐이다. Snopes 카테고리를 열어 보면 `['Entertainment', 'Music', 'AI-generated Content']` 같은 주제어지 등급이 아니다.

판정은 기사 페이지의 schema.org **ClaimReview** 마크업에 있다. 기계가 읽으라고 붙이는 구조화 데이터다. 그날 올라온 기사 페이지를 열어 그 블록만 꺼낸다.

```python
LD_JSON = re.compile(
    r'<script[^>]*type=["\']application/ld\+json["\'][^>]*>(.*?)</script>', re.S
)

def _extract_claim_review(page: str) -> dict[str, Any] | None:
    """Pull the ClaimReview node out of a page's structured data."""
    for block in LD_JSON.findall(page):
        try:
            data = json.loads(block)
        except json.JSONDecodeError:
            continue
        nodes = data if isinstance(data, list) else [data]
        if isinstance(data, dict):
            nodes = nodes + list(data.get("@graph", []))
        for node in nodes:
            if isinstance(node, dict) and "ClaimReview" in str(node.get("@type", "")):
                return node
    return None
```

페이지 하나에 `ld+json` 블록이 여럿 있을 수 있고, 한 블록이 배열이거나 `@graph` 안에 노드를 묶고 있을 수도 있다. 셋 다 펼친 뒤 `@type`에 `ClaimReview`가 들어간 첫 노드를 쓴다. `@type`을 `str()`로 감싸 부분 문자열로 비교하므로 `@type`이 배열이어도 걸린다. 깨진 JSON 블록은 건너뛴다. 테스트 두 개가 "`WebPage` 블록 옆의 ClaimReview만 집는다"와 "깨진 JSON이면 `None`"을 고정한다.

정규화에서 쓰는 필드는 둘이다.

```python
"claim_reviewed": _plain_text(review.get("claimReviewed")),
# alternateName is the human-readable verdict ("Fake",
# "False"); ratingValue is a scale that differs per checker.
"rating": rating.get("alternateName"),
```

`ratingValue`는 검증 기관마다 척도가 달라서 쓰지 않고 사람이 읽는 판정 문구 `alternateName`을 쓴다. FactCheck.org는 이 마크업을 붙이지 않는다. 그쪽 `rating`은 비운다. 없는 판정을 기본값으로 채우면 "없다"는 사실이 사라진다. 테스트 이름이 그대로 `test_fact_checks_leaves_the_rating_empty_rather_than_guessing`이다.

운영 데이터로 보면 이렇다.

| 발행처 | 레코드 | `rating` 있음 |
|---|---|---|
| Snopes | 351 | 222 |
| FactCheck.org | 47 | 0 |

Snopes의 `rating` 빈칸 129건은 URL로 갈린다. `/fact-check/` 경로 220건은 전부 판정이 있고, 빈칸은 `/news/`, `/collections/`, `/articles/`, `/tracker/` 같은 검증 글이 아닌 페이지다. 판정 분포 상위는 True 63, False 43, Mixture 25, Fake 22다.

기사 하나를 못 읽어도 그날 전체를 잃지 않게, 페이지 요청 실패는 경고로 남기고 그 기사만 `None`으로 둔다. 페이지 요청 사이에는 1초 간격을 둔다. 운영 데이터에서 파티션당 레코드는 중앙값 7건, 최대 11건이라 이 간격이 부담이 되지 않는다.

## 10. 위키백과: 대문을 빼지 않으면 순위가 무의미하다

한국어 위키 조회수 상위 목록에는 `위키백과:대문`, `특수:검색` 같은 탐색용 문서가 섞여 있다. 설계 당시 하루치를 열어 보니 상위 987건 중 8건이 그런 문서였고, 대문 하나가 실제 1위 문서의 5배를 넘었다.

수집된 raw 62일치로 다시 세 봤다.

| 지표 | 값 |
|---|---|
| 한국어 상위 목록 길이 | 982 ~ 1,000건 |
| 그중 탐색용 문서 | 5 ~ 14건 |
| 대문 조회수 ÷ 실제 1위 문서 조회수 | 3.13 ~ 6.38배, 중앙값 5.32배 |

62일 모두 대문이 목록에 있었다. 빼지 않으면 매일 1위가 대문이다.

```python
# Navigation rather than subject matter -- the main page alone outdraws every
# real article, which would make any ranking useless.
SKIP_EXACT = {"Main_Page", "위키백과:대문", "-"}
SKIP_PREFIXES = ("특수:", "Special:", "위키백과:", "Wikipedia:", "Portal:", "포털:")


def _is_navigation(title: str) -> bool:
    return title in SKIP_EXACT or title.startswith(SKIP_PREFIXES)
```

```python
articles = [a for a in items[0].get("articles", []) if not _is_navigation(a["article"])]
# ... 생략
# Ranks are renumbered after dropping navigation so they stay contiguous.
for rank, article in enumerate(articles[:TOP_N], start=1):
```

API가 준 `rank`를 그대로 쓰면 빠진 자리 때문에 순위에 구멍이 생기고 "상위 10개"가 실제로는 8개가 된다. 거른 뒤 1부터 다시 매긴다. 테스트가 프로젝트별 순위가 `1..N`으로 연속인지 확인한다.

상위 500건에서 자르는 근거도 설계 문서에 있다. 500위 조회수가 173회, 마지막 979위가 114회로 꼬리가 얇다.

## 11. 테스트: CI에서는 네트워크를 쓰지 않는다

CI 게이트에는 네트워크 없이 도는 테스트만 넣었다. 외부 API 가용성이 CI 신뢰도를 흔들면 안 된다. 워크플로는 수집 전에 테스트를 돌린다.

```yaml
- name: Run tests
  # Guards against committing data produced by broken normalizers.
  run: uv run pytest -q
```

정규화기가 깨진 상태로 데이터를 커밋하는 걸 막는 장치다. 네트워크와 실제 데이터를 테스트에서 떼어 내는 방법은 세 가지다.

**저장 경로 격리.** 모든 테스트가 자동으로 임시 디렉터리에 쓴다. 테스트가 실수로 `data/`나 매니페스트를 건드릴 수 없다.

```python
@pytest.fixture(autouse=True)
def isolated_data_root(tmp_path, monkeypatch):
    """Point every storage write at a temp dir so tests never touch data/."""
    monkeypatch.setenv("DATASETS_DATA_ROOT", str(tmp_path / "data"))
    return tmp_path / "data"
```

**실제 응답 픽스처.** `tests/fixtures/`에 소스별로 실제 캡처한 API 응답을 넣고, `normalize`에 넣어 스키마를 통과하는지 본다. 날짜로 거르는 피드(`yna_news`, `geeknews`, `fact_checks`)는 캡처한 날짜를 `*.meta.json`에 따로 적어 그 날짜로 돌린다. 오늘 날짜로 돌리면 픽스처의 항목이 전부 걸러져 테스트가 아무것도 검사하지 않는다.

**Dagster 실행 대체.** 드라이버 테스트는 `materialize`를 소스별 성공/실패를 미리 정한 가짜로 바꾼다. 복구, 재확인, 요청 제한 가드, 종료 코드를 실제 API 없이 조합별로 검사할 수 있다.

```python
@pytest.fixture
def fake_materialize(monkeypatch):
    """Replace Dagster execution with a scripted outcome per source."""
    calls = []

    def factory(outcomes):
        def _materialize(assets, partition_key, raise_on_error):
            name = assets[0].key.to_user_string()
            calls.append((name, partition_key))
            return FakeResult(outcomes.get(name, True))

        monkeypatch.setattr(cli, "materialize", _materialize)
        return calls

    return factory
```

| 스위트 | 항목 | 잡는 것 |
|---|---|---|
| `test_cli.py` | 40 | 소스별 예정 파티션, 스냅샷 백필 거부, 복구 대상 판정, 재확인 범위, 요청 제한 가드, 종료 코드 |
| `test_normalize.py` | 48 | 픽스처 → 스키마 통과, 정규화 순수성, KST 경계 쿼리, GeoJSON 좌표 순서, 위키 대문 제외, ClaimReview 추출 |
| `test_collect.py` | 18 | 재실행 멱등성, 빈 응답 실패, 스키마 드리프트, 실패 시 curated 미생성, 병합 |
| `test_registry.py` | 11 | 레지스트리 필드 조합 불변식, README 카탈로그 동기화 |
| `test_storage.py` | 9 | 경로 규약, gzip 압축, 키 정렬 |
| `test_manifest.py` | 7 | append-only, `run_at` 기준 최신 판정 |

로컬(Windows, Python 3.14)에서 133항목이 19.9초에 통과한다.

GeoJSON 좌표 순서 테스트는 짧지만 이유가 분명하다. GeoJSON은 `[경도, 위도, 깊이]` 순서라 바꿔 읽으면 한반도 지진이 태평양에 찍히고, 아무 에러도 나지 않는다. 위도가 -90~90, 경도가 -180~180 안인지만 봐도 상당 부분 걸린다.

실제 API를 때리는 테스트는 `@pytest.mark.network`로 분리하도록 `pyproject.toml`에 마커를 선언해 뒀다. 다만 지금 이 마커를 단 테스트는 0개다(`pytest -m network`가 133개 전부 deselect). 외부 API 계약을 확인하는 테스트는 아직 없고, 그 역할은 매일 실제 수집과 매니페스트가 대신하고 있다.

## 12. 한계와 다음 단계

- **피드 수집이 스케줄러에 막혀 있다.** 30분마다 요청해도 실제 읽기 간격 중앙값이 257분이고 90분 이내 간격은 306개 중 0개다. 연합뉴스처럼 창이 90분인 피드는 하루 대부분을 놓친다. 근본 해결은 상주 스케줄러(로컬 상시 실행이나 클라우드 VM)다.
- **arXiv lag 3일은 잠정치다.** 안정 소요일을 재다 429에 막혔다. 재확인으로 대부분 수렴했지만 장애가 길면 09-19, 09-26처럼 얇게 남는다.
- **복구 창보다 긴 장애는 영구 구멍이 된다.** 09-20 arXiv가 그렇다. 창 7일과 재확인 3일은 장애가 그보다 짧다고 가정한 값이다.
- **병합 소스의 매니페스트 상태는 읽기 단위다.** 파티션에 데이터가 있어도 마지막 읽기가 실패했으면 `failed`로 남는다.
- **텍스트 소스가 용량 대부분이다.** 설계 문서 추정으로 arXiv 연 165MB, 임상시험 연 153MB다. 커지면 이 둘만 curated를 gzip하거나 카테고리를 줄인다. 리텐션 정책은 아직 없고 무기한 보관이다.
- **네트워크 테스트가 비어 있다.** 마커만 있고 테스트는 없다. CI는 마커 필터 없이 pytest를 돌리므로 지금 네트워크 테스트를 추가하면 CI에서도 돈다.
- **curated는 JSONL이다.** 최종 활용처가 정해지면 Parquet 변환을 붙일지 정한다.

데이터 수집에서 "성공"의 정의가 소스마다 달랐다. 0건이 정답인 날이 있고, 성공했지만 부족한 날이 있고, 성공하면 오히려 오염인 날이 있다. 이 차이를 레지스트리 필드로 적고 필드 조합을 테스트로 묶는 게 이 파이프라인 코드의 대부분이었다.

코드: [github.com/kasangyong/dataset](https://github.com/kasangyong/dataset)
