---
title: "렌더링 통념 27개를 CDP 트레이스로 다시 쟀다: 뒤집힌 것, 조건이 붙은 것, 내가 틀린 것"
date: 2026-10-23 10:20:00 +0900
categories: [Web, Performance]
tags: [chrome, cdp, tracing, rendering, compositor, content-visibility, lcp]
image:
  path: /assets/img/posts/browser-rendering-myths/a1-stage-skip.png
  alt: left와 transform의 단계별 트레이스 이벤트 횟수
---

Chrome DevTools Protocol(CDP)로 트레이스를 떠서 렌더링 성능에 관한 통념 27개를 실측과 나란히 놓았다. 측정 전에 예측을 파일로 등록하고 나중에 채점했다.

| 항목 | 값 |
|---|---|
| 실험 | 33개 (Track A 기초 측정, Track C 재검증과 실제 사이트) |
| 사전 등록 주장 | 102개: 성립 51 · 반증 36 · 부분·보류 15 |
| 통념 27개 판정 | 맞다 3 · 틀렸다 16 · 조건부 8 |
| 실측 사이트 | 18곳 (Wikipedia, WHATWG, Python 문서, Wikivoyage 등) |
| 자기 정정 / 폐기한 측정법 | 정정 표 25행 / 6개 |
| 환경 | Chrome 152.0.7977.83 (headless=new, GPU 가속), Windows 11 |
| 의존성 | Node 22 내장 `WebSocket`만 사용, 0개 |

## 1. 프레임 시간 대신 이벤트를 세는 측정 장치

### 처음 고른 지표가 틀렸다

첫 실험(A1)은 rAF 프레임 시간을 성공 기준으로 잡았다. "`transform`의 p95가 `left`의 절반 이하"를 걸었는데, 박스 600개에서 58.2ms 대 41.7ms로 1.40배에 그쳐 실패했다. 그런데 같은 600개 조건에서 Layout 이벤트는 91회 대 15회였다. 박스를 5개로 줄이고(두 모드 모두 482프레임) 트레이스로 세 보니 Layout은 32배, Paint는 94배 차이가 났다.

rAF 콜백 간격에는 애니메이션 비용 말고도 측정 루프 자체, 프레임 스케줄링, headless의 프레임 상한이 같이 섞인다. 프레임 시간은 결과일 뿐이고 원인은 어느 단계가 몇 번 돌았는지에 있다. 그래서 이후 실험은 모두 트레이스 이벤트 카운트를 1차 지표로 쓴다.

### 최소 CDP 클라이언트

Puppeteer 없이 Node 22 내장 `WebSocket`으로 CDP에 붙는다. 요청 id로 응답을 짝짓고 id가 없는 메시지는 이벤트로 돌린다.

```javascript
class CDP {
  constructor(ws) {
    this.ws = ws;
    this.id = 0;
    this.pending = new Map();
    this.handlers = new Map();
    ws.addEventListener('message', ev => {
      const msg = JSON.parse(ev.data);
      if (msg.id !== undefined) {
        const p = this.pending.get(msg.id);
        if (!p) return;
        this.pending.delete(msg.id);
        msg.error ? p.reject(new Error(JSON.stringify(msg.error))) : p.resolve(msg.result);
      } else {
        const h = this.handlers.get(msg.method);
        if (h) h(msg.params);
      }
    });
  }
  send(method, params = {}) {
    const id = ++this.id;
    this.ws.send(JSON.stringify({ id, method, params }));
    return new Promise((resolve, reject) => this.pending.set(id, { resolve, reject }));
  }
  // ... on(), evaluate() 생략
}
```

트레이스는 `Tracing.start`로 `devtools.timeline` 계열 카테고리를 켜고 `Tracing.dataCollected`로 들어오는 이벤트를 모은 뒤 이름별로 횟수와 `dur` 합계를 낸다.

```javascript
function summarize(events, elapsedMs) {
  const counts = {};
  const durations = {};
  for (const e of events) {
    if (e.ph === 'M') continue;            // 메타데이터 제외
    counts[e.name] = (counts[e.name] || 0) + 1;
    if (typeof e.dur === 'number') {
      durations[e.name] = (durations[e.name] || 0) + e.dur / 1000; // µs → ms
    }
  }
  // ... STAGES 별 정리 생략
}
```

세는 대상은 `UpdateLayoutTree`(Style), `Layout`, `PrePaint`, `Paint`, `Commit`, `RasterTask` 같은 파이프라인 단계 이벤트다. 측정을 시작하기 전에 600ms 동안 rAF가 10프레임 이상 도는지 확인하고, 못 미치면 측정을 중단한다. 처음에는 브라우저 패널에서 rAF를 쟀는데, 패널이 가려져 `document.hidden`이 되면 rAF가 완전히 멈췄다. 그걸 모르면 빈 배열을 정상 측정으로 착각하게 된다. 그래서 그 방법은 버리고 이 검사를 넣었다.

### 그 밖의 계측기

| 무엇을 재나 | 도구 |
|---|---|
| 레이어 수, 승격 사유 | `LayerTree.layerTreeDidChange` |
| GPU·타일 메모리 | `disabled-by-default-memory-infra` + `Tracing.requestMemoryDump` (`gpu/shared_images`, `cc/tile_memory`) |
| 메인 블로킹 중 화면 | `Page.startScreencast` (스크린샷 API는 못 쓴다, 2장) |
| 실제 사이트에 CSS 주입 | `Fetch.requestPaused`로 HTML 응답을 가로채 `<style>`을 심는다 |
| LCP·리소스 타이밍 | 주입한 `PerformanceObserver` + `performance.getEntriesByType('resource')` |
| 규칙의 근거 | Chromium 소스 (`cc/layers/tile_size_calculator.cc` 등) |

### 절차의 규칙

- 측정 전에 `PREDICTION.md`에 주장과 판정 기준을 적는다. A4에서는 "`cv-auto`의 Layout이 N=100→8000에서 5배 미만 증가"를 미리 박아 두었고, 실측 21.4배를 반증으로 기록했다.
- 설정마다 Chrome을 새 프로필로 새로 띄운다. 한 브라우저에서 GPU 메모리를 연속으로 재면 리소스 풀이 이월돼 값이 소수점까지 같게 나왔다.
- 조건 순서를 섞는다. 고정 순서로 돌렸을 때 세션 드리프트가 조건 차이로 보인 적이 있다(5장).

## 2. "transform은 메인 스레드를 안 쓴다": 틀렸다

![A1 단계별 이벤트 횟수](/assets/img/posts/browser-rendering-myths/a1-stage-skip.png)
_박스 5개, 모드당 4초. 세 모드가 정확히 같은 482프레임을 냈다_

박스를 5개로 줄이자 `left`, `transform`, `transform + will-change` 세 모드가 모두 482프레임을 냈다. 프레임 수가 같으니 이벤트 횟수를 그대로 비교할 수 있다.

| 단계 | `left` | `transform` | 배율 |
|---|---:|---:|---:|
| `UpdateLayoutTree` (Style) | 482회 | 482회 | 1× |
| `Layout` | 482회 | 15회 | 32× |
| `PrePaint` | 482회 | 482회 | 1× |
| `Paint` | 2,832회 | 30회 | 94× |
| `RasterTask` | 979회 | 15회 | 65× |
| `InvalidateLayout` | 497회 | 0회 | ∞ |

"Layout·Paint를 건너뛴다"는 맞고, `InvalidateLayout` 497 대 0이 가장 깨끗한 증거다. 다만 Style과 PrePaint는 양쪽 모두 매 프레임 돈다. RenderingNG 문서는 "animations can skip layout, pre-paint and paint"라고 쓰지만 PrePaint는 건너뛰지 않았다. 매 프레임 호출되고, 할 일이 없어 즉시 반환했을 뿐이다(0.13ms/프레임). Style은 애니메이션 값을 갱신하느라 매 프레임 실제로 일하는데, 박스 5개일 때는 0.19ms/프레임으로 작다.

### 메인 스레드를 막아 보면

박스 1개를 두고 메인 스레드를 1,500ms 동안 busy loop로 점유한 다음, 그 구간에 컴포지터 스레드가 무엇을 했는지 트레이스에서 잘라냈다. 블록 구간은 메인 스레드에서 가장 긴 태스크로 잡는다.

```javascript
// 메인 스레드의 가장 긴 task = busy loop 구간
let blockEv = null;
for (const e of events) {
  if (e.ph === 'M' || typeof e.dur !== 'number') continue;
  if (names.get(`${e.pid}:${e.tid}`) !== 'CrRendererMain') continue;
  if (e.dur > (BLOCK_MS * 900) && (!blockEv || e.dur > blockEv.dur)) blockEv = e;
}
```

`BLOCK_MS * 900`은 ms를 µs로 바꾸고 90%를 곱한 값이다. 이 구간 안의 이벤트를 스레드 이름별로 센다.

박스 1개, 메인 1,500ms 점유 구간:

| 모드 | Compositor 이벤트 | `MainFrame.Draw` | `Swap` (화면 제출) |
|---|---:|---:|---:|
| `left` | 4,512 | 0 | 0 |
| `transform` | 17,948 | 536 | 360 |

![메인 스레드 블로킹 중 left와 transform](/assets/img/posts/browser-rendering-myths/main-thread-block.gif)
_램프가 빨간 동안 메인 스레드가 JS에 점유돼 있다. `Page.startScreencast`로 녹화했다_

### 측정 도구가 메인 스레드에 걸렸다

이 장면을 처음엔 `Page.captureScreenshot`으로 찍으려 했는데, 결과가 "세 모드 전부 움직였다"로 나왔다. 확인해 보니 2,000ms busy loop 중 `captureScreenshot` 응답이 1,936ms 뒤에 왔다. 스크린샷도 메인 스레드에 걸려 있어서 4장 모두 블록이 끝난 뒤에 찍혔다. 그대로 보고했다면 `left`도 컴포지터에서 돈다고 썼을 것이다.

같은 함정을 A5에서 한 번 더 만났다. "화면 갱신 시점"을 rAF 콜백으로 쟀는데, 그 콜백도 메인 스레드에서 돌기 때문에 `busy(200)`이 막고 있는 동안 잰 값은 픽셀 시점이 아니라 메인이 한가해진 시점이었다. 그 뒤로 "계측기가 메인 스레드에 걸리는지 먼저 확인한다"를 규칙에 넣었다.

### 요소가 많아지면

박스 1,000개에서 프레임당 Style은 `left` 11.58ms, `transform` 14.95ms였다. 애니메이션이 없는 대조군은 1,000개여도 0.004ms에 그쳤으니 Style 비용은 DOM 크기가 아니라 애니메이션 개수를 따라간다. `transform`의 이득은 250~500개에서 최대 2.1배였다가 1,000개에서 1.5배로 줄었다.

Commit도 레이어 수에 따라 커진다. 처음엔 "1,000개에서 commit이 7배 폭발"이라고 썼다가 나중에 이 배율을 철회했다. 화면이 120Hz라 프레임 간격이 8.3ms의 배수로 양자화되는데, `총합 ÷ 프레임 수`로 낸 값은 레이어 수가 아니라 이 양자화 계단을 재고 있었다. 표본을 고정한 하네스(3장)로 다시 재 보니 commit은 500~3,000 레이어에서 레이어당 14.4µs로 선형이었다. `transform`이 없애 주는 건 Layout·Paint까지다. Style·PrePaint·Commit은 메인 스레드에 남는다.

## 3. "레이어가 많으면 어느 지점에서 급격히 나빠진다": 절벽은 측정 잡음, 임계점은 내 2초였다

### 표본 수가 측정 대상의 함수였다

C2에서는 2,000 레이어 근처에서 commit이 5배 뛰는 것처럼 보였다. 그런데 그 측정은 자유 실행 애니메이션의 프레임을 표본으로 써서, 느릴수록 프레임이 적게 잡혔다. 2,000 레이어에서는 3초 동안 표본이 49개뿐이었고 같은 설정을 반복했을 때 편차가 5.0배였다. 찾으려던 신호와 같은 크기였다.

그래서 표본 수를 내가 정하는 하네스를 새로 만들었다. 모든 박스가 같은 커스텀 프로퍼티 하나를 `transform`에 쓴다.

```css
.box{position:absolute;left:0;
     width:var(--bw,38px);height:var(--bh,15px);
     transform:translateX(var(--tx,0px));
     /* 정적 transform 만으로는 합성 레이어가 안 생긴다.
        이걸 빼면 박스 500개가 레이어 1개에 그려져서, 재는 게 commit 이 아니라 repaint 가 된다.
        (처음에 빼먹고 측정했다가 layerCount 가 4로 나와서 잡았다.) */
     will-change:transform}
```

```javascript
/**
 * --tx 를 k번 토글한다. 토글 1회 = 레이어 N개 전부 새 transform = commit 1회.
 * 프레임이 빠르든 느리든 표본 수는 정확히 k다.
 */
async function pump(k, dist) {
  const d = dist || 240;
  const warm = 10;                       // 첫 몇 번은 캐시가 차오르는 구간이라 버린다
  const times = [];
  await frame();
  for (let i = 0; i < k + warm; i++) {
    const t0 = performance.now();
    root.style.setProperty('--tx', (i % 2 ? d : 0) + 'px');
    await frame();
    if (i >= warm) times.push(performance.now() - t0);
  }
  // ... 분위수 계산 생략
}
```

`frame()`은 rAF를 두 번 기다린다. 이렇게 바꾸자 같은 설정 반복 간 편차가 5.0배에서 1.1배로 줄었다.

![C3 레이어 스윕](/assets/img/posts/browser-rendering-myths/c3-layer-cliff.png)
_설정마다 표본 480개. 2,000(55.5ms)과 1,750(55.7ms)은 95% 구간이 포개진다_

절벽이 있어야 할 1,750→2,000 구간의 기울기(시간 배율 ÷ 레이어 배율)는 0.87이었다. 원래 가설 "512MB 타일 예산과 관련"은 측정 전에 산수로 죽일 수 있었다. 38×15 박스는 64×64 타일 16KB라 2,000개를 모아도 31.3MB, 예산의 6.1%다. 예산을 채우려면 32,768개가 있어야 한다.

### Raster 110배를 반복별로 풀고 횟수로 셌다

C3에서 하나가 더 남았다. Raster가 1,000→1,500 레이어에서 110배 뛰었는데, 반복별로 풀어 보니 평균이 만든 값이었다.

| 레이어 | rep0 | rep1 | rep2 |
|---:|---:|---:|---:|
| 1,000 | 1.00 | 0.07 | 0.05 |
| 1,500 | 61.69 | 60.37 | 0.03 |
| 2,000 | 39.52 | 67.61 | 50.40 |

값은 0.05 아니면 60이고 그 사이가 없다. 상한 후보 둘은 소스를 보고 먼저 지웠다. 타일 바이트 예산은 가장 심한 조건에서도 512MB의 7%였고 리소스 개수 상한 `kDefaultNumResourcesLimit`는 1,000만 개였다.

답은 `RasterTask`의 횟수에 있었다. 켜진 실행을 예로 들면 1,000 레이어에서 1,003회, 1,100 레이어에서 1,103회로 레이어당 1.00회였고, 켜진 실행 전체 평균도 레이어당 1.01회다.

| 가설 | 예상 횟수 | 판정 |
|---|---|---|
| 매 프레임 재래스터 | 레이어 × 토글 ≈ 15,000 | 기각 |
| 축출 후 재생성 | 1회보다 많고 불규칙 | 기각 |
| 초기 래스터가 안 끝났다 | 레이어당 정확히 1회 | 채택 |

하네스는 `build(N)` 뒤 2초를 기다렸다가 측정을 시작한다. 타일을 처음 만드는 일이 2초 안에 안 끝나면 그 꼬리가 측정 창으로 넘어온다. 그게 맞다면 레이어 수는 그대로 두고 기다리는 시간만 늘려도 래스터가 사라져야 하고, 축출이라면 사라질 이유가 없다.

![settle만 바꾼 판별 실험](/assets/img/posts/browser-rendering-myths/c4-settle.png)
_`results-settle.json`. 레이어 2,000개, settle마다 5회. "켜짐"은 토글당 Raster가 5ms를 넘은 실행 수_

settle 1초에서 1,407회이던 것이 4초에서는 1회로 떨어진다. 2,000 타일을 처음 만드는 데 1,395ms가 걸렸으니 2초는 아슬아슬했다. 레이어 수별 켜짐 비율도 계단이 아니라 0%(800)에서 83%(1,750)까지 완만하게 올라가는데, 이것도 같은 쪽을 가리킨다. 상한을 넘는 문제라면 계단이어야 하고, 경주라면 매번 이기기도 하고 지기도 한다.

![C4 레이어 수별 켜짐 비율](/assets/img/posts/browser-rendering-myths/c4-raster-threshold.png)
_레이어 수별 6회 반복 중 Raster가 관측된 비율_

### 판별 도구 자체가 조건을 바꿨다

`probe-settle`의 첫 버전은 가설과 반대 결과를 냈다. settle 1초인데도 0/4로 안 켜졌는데, 초기 래스터 총량까지 재려고 settle 동안 트레이싱을 켜 둔 게 원인이었다. 그때 남긴 주석이 지금도 코드에 있다.

```javascript
// settle 동안은 트레이싱을 켜지 않는다.
// 처음엔 초기 래스터 총량도 같이 재려고 여기서 한 번 더 트레이싱했는데,
// 두 트레이스 사이의 전송 지연(20만 이벤트면 수 초)이 settle 에 몰래 더해져서
// settle 1초짜리가 사실상 수 초가 됐다. 그래서 래스터가 안 새는 것처럼 보였다.
// 지금은 measure.mjs 와 똑같은 조건으로 두고 settle 만 바꾼다.
await ev(`__C3.build(${layers}, 38, 15)`);
await sleep(settle);
```

## 4. "LCP가 이미지면 content-visibility는 소용없다": 콜드 캐시에서 12~28% 빨라진다. 이유는 다운로드가 아니었다

### 예측과 결과

사전 등록 예측은 "이미지 LCP는 다운로드에 묶여 있으니 레이아웃을 줄이는 `content-visibility`로는 안 빨라진다"였다. 대상으로는 모바일 폭(390×844)에서 LCP가 `<img>`인 위키 문서 3곳을 골랐다. 실제 HTML 응답을 `Fetch.requestPaused`로 가로채 `.mw-parser-output section > *`에 `content-visibility: auto`를 심었다.

| 사이트 | Layout | LCP |
|---|---|---|
| Grand Canyon | 478 → 125ms (−74%) | 2,224 → 1,597ms (−28%) |
| Milky Way | 618 → 217ms (−65%) | 2,349 → 1,935ms (−18%) |
| Mount Everest | 644 → 312ms (−52%) | 2,346 → 2,064ms (−12%) |

콜드 캐시에서 6회 중앙값이고, warm에서는 0~38%였다. 예측은 틀렸다. 그런데 LCP 이미지의 리소스 타이밍을 직접 보니 다운로드가 빨라진 건 아니었다.

![LCP 이미지 요청 시작과 전송](/assets/img/posts/browser-rendering-myths/c8-request-start.png)
_C8 `RESULTS.md` §4의 중앙값으로 그렸다. 바이트는 38,064 · 19,938 · 28,194로 두 모드가 같다_

바이트는 같은데 요청 시작이 415~579ms 앞당겨졌다. `content-visibility`는 네트워크를 건드리지 않는데도 요청은 일찍 나갔다.

### 정정 21번, "화면 밖 이미지 요청을 줄여서 빨라진 것"

다음으로 세운 가설은 cv가 화면 밖 이미지 요청을 줄여서 대역폭 경쟁이 덜해졌다는 것이었다. `probe-why.mjs`는 로드가 끝난 뒤 이걸 브라우저에 직접 묻는다.

```javascript
const REPORT = `(() => {
  const L = window.__lcp;
  const res = performance.getEntriesByType('resource');
  const imgRes = res.filter(r => r.initiatorType === 'img');
  // ... LCP 요소 찾기 생략
  return {
    lcp: L,
    lcpImg: el ? {
      loading: el.getAttribute('loading'),
      fetchpriority: el.getAttribute('fetchpriority'),
      // ... 생략
      // 레이아웃 후 좌표. 초기 뷰포트 안인지.
      top: r ? +r.top.toFixed(0) : null, h: r ? +r.height.toFixed(0) : null,
    } : null,
    // 문서 전체 img 중 lazy 비율
    imgTotal: document.images.length,
    imgLazy: [...document.images].filter(i => i.getAttribute('loading') === 'lazy').length,
    // 실제로 요청이 나간 이미지 수. cv 가 화면 밖 요청을 줄이는지
    imgRequested: imgRes.length,
    imgBytes: imgRes.reduce((a, b) => a + (b.encodedBodySize || 0), 0),
  };
})()`;
```

| 관측 | plain | cv-auto |
|---|---|---|
| 요청된 이미지 / 전체 (lazy) | 10/69(63) · 6/46(43) · 18/233(226) | 완전히 동일 |
| 요청된 이미지 바이트 | 110KB · 81KB · 87KB | 완전히 동일 |
| LCP 이미지의 `loading` | `lazy` (세 곳 전부) | `lazy` |
| LCP 이미지 top | 369 · 340 · 465px (뷰포트 844px 안) | 동일 |

요청 수도 바이트도 같으니 경쟁 완화 가설은 틀렸고, 정정 표 21번이 이 건이다. 남은 설명은 `loading="lazy"`였다. lazy 이미지는 뷰포트 안에 있는지 알아야 요청이 나가는데, 그건 레이아웃을 해 봐야 안다. 레이아웃이 요청의 선행 조건이 되니 cv가 레이아웃을 줄이면 요청도 앞당겨진다.

### LCP로 판정하려다 지표를 바꾼 반증 테스트

설명을 추측으로 두지 않으려고 HTML을 가로챌 때 `lazy`를 전부 `eager`로 바꿨다. CSS로는 못 바꾸니 문자열을 치환했다.

```javascript
// 반증 조건: lazy 를 전부 eager 로. CSS 로는 못 바꾸므로 HTML 문자열에서 직접.
if (lazy === 'eager') html = html.split('loading="lazy"').join('loading="eager"');
```

2×2(plain·cv-auto × lazy·eager) × 3곳 × 5회 = 60회, 순서를 섞고 콜드 캐시로 쟀다. 치환 뒤 남은 lazy는 0/69, 0/46, 0/233이었다. 가설이 맞다면 eager에서는 cv의 LCP 개선이 사라져야 한다.

LCP로 보면 2대1로 갈렸다. Grand Canyon과 Everest는 개선이 사라졌지만 Milky Way는 lazy 팔에서부터 개선이 없었다(−5%). 이 실행의 LCP 흩어짐이 1.3~1.6배로 효과 크기와 비슷했으니, 가설에서 몇 단계 떨어진 최종 지표인 LCP로는 판정을 낼 수 없었다.

그래서 지표를 가설이 직접 예측하는 양인 요청 시작 시각으로 바꿨다.

| 이미지 | 점 | Layout 절감 | 요청 앞당김 |
|---|---:|---|---|
| `lazy` | 6 (두 실행) | 57~402ms | +59 ~ +579ms, 전부 양수 |
| `eager` | 3 | 65~134ms | −106 · +47 · −98ms (평균 −52ms) |

Layout 절감 자체는 lazy(59~66%)든 eager(61~72%)든 같았는데, 요청이 앞당겨지는 건 lazy일 때뿐이었다. 이때 일이 방법론 21조 "최종 지표가 아니라 가설이 예측하는 양을 잰다"로 남았다.

lazy 6점에서 Layout 절감과 요청 앞당김의 상관은 r = 0.95였지만, 6점이 본측정과 반증 실행 두 덩어리로 갈려 그 간격이 만든 값이라 증거로 쓰지 않았다. 증거는 같은 실행 안에서 lazy/eager로 부호가 뒤집힌 대조다.

### 실제 eager 사이트와 비싼 eager 사이트

eager 조건은 내가 HTML을 고쳐 만든 것이라, LCP 이미지가 실제로 eager인 사이트를 18곳 뒤져 Wikivoyage에서 찾았다. 거기서는 Layout 절감 예산이 30~70ms뿐이라 시간 차이로는 검정력이 없었다(셀당 약 96회 필요, 실제 5회). 기전이 예측하는 건 시간 차이가 아니라 순서여서 "요청 전에 끝난 Layout 수"를 셌다. eager 6/6은 0개, 강제 lazy 6/6은 4~8개, 실제 lazy인 Wikipedia 2/2는 6~8개로 14회가 전부 갈렸다.

레이아웃이 비싼데 eager인 문서도 찾았다. Turku는 Layout을 424ms 아꼈는데(Grand Canyon 354ms보다 많다) 요청은 38ms 늦어졌다. 앞당김이 크고 일관되게 나온 건 "LCP 이미지가 lazy이고 레이아웃이 수백 ms"인 칸 하나뿐이었다.

## 5. 내가 잘못 잰 것들

정정 표는 25행이고, 가설이 틀린 경우와 측정이 틀린 경우가 섞여 있다. 아래에는 그중 측정이 틀린 경우만 추렸다.

| 무엇을 잘못 쟀나 | 어떻게 드러났나 | 바로잡은 뒤 |
|---|---|---|
| 타일은 2의 거듭제곱 크기라고 결론 | 고른 크기 세 개(38×15, 100×100, 200×200)가 하필 64배수 규칙과 답이 같았다. Chromium 소스 `kTileRoundUp = 64` | 두 규칙이 갈리는 130×130으로 판별. 실측 140.3KB/레이어, 64배수 예측 144KB의 97.4% |
| 프레임당 commit 비용을 `총합 ÷ 프레임 수`로 계산 | 120Hz 양자화(8.3 → 16.3 → 39.9ms) | 토글 수 고정 하네스로 레이어당 14.4µs, 선형 |
| 켜짐/꺼짐 분류를 시간 기준(5ms)으로 | 100×100 박스가 "절대 안 켜진다"를 쫓다가 `rasterN`을 보니 253·403·603회 | 레이어당 1회로 똑같이 새고 있었다. 횟수로 답을 얻어 놓고 분류는 시간으로 하고 있었다 |
| GPU 래스터의 타일당 µs 비교 | 같은 설정 반복이 26배 흩어짐. 조건 차이는 2.1배 | 이전 브라우저 종료를 기다리지 않은 절차(0.5초) 탓. 완전 종료 + 3초 대기로 p50 흩어짐 6.1배 → 1.6배 |
| 점프형 "스크롤"(21,786px씩 40단계) | 연속형(300px×60)과 결론이 정반대 | 정확한 `contain-intrinsic-size`가 점프형에선 1.9배 나쁘고 연속형에선 1.78배 좋았다 |
| cv 요소당 비용이 2.8배 흔들린다 | 분자는 FCP까지의 Layout, 분모는 끝까지 파싱된 요소 수. r(FCP, 요소당 비용) 0.76·0.84 | 기준선을 `DOMContentLoaded`로 맞추자 23~58µs, 사이트 안 2배 이내. 앞서 반증이라 쓴 두 주장이 성립으로 뒤집혔다 |

26배 흩어짐을 찾은 뒤 영향 범위를 정리했다. 종료 대기 0.5초는 A2부터 모든 시간 측정에 섞여 있었다. 횟수·개수로 낸 결론은 영향이 없지만, 시간으로 낸 결론 중 차이가 2~6배 미만인 건 다시 봐야 한다. Track A의 주요 결론(Layout 32배, Paint 94배, Style 82%)은 배율이 커서 그대로 살아남는다.

정정이 쌓이면서 방법론이 23조가 됐다.

- 표본 수는 내가 정한다. 측정 대상의 함수가 되면 안 된다.
- 평균 전에 반복별 값을 본다. 시간 전에 횟수를 센다.
- 내가 정한 상수(2초 settle, 5ms 기준)를 변수로 바꿔 본다.
- 조건 차이를 말하기 전에 같은 조건 반복의 흩어짐부터 잰다.
- 비율을 만들 때 분자와 분모의 기준선을 맞춘다.

## 6. 통념 27개 전체

| # | 통념 | 판정 | 실측 요약 | 실험 |
|---:|---|:---:|---|---|
| 1 | `transform`은 레이아웃·페인트를 건너뛴다 | 맞다 | Layout 32배, Paint 94배 감소 | A1 |
| 2 | `transform`은 메인 스레드를 안 쓴다 | 틀림 | Style·PrePaint·Commit은 매 프레임 돈다 | A1, A1b |
| 3 | `will-change`를 쓰면 레이어로 승격된다 | 조건부 | CSS transform 애니메이션은 이미 승격돼 있어 차이 없음 | A1b |
| 4 | 레이어는 공짜가 아니다 | 맞다 | 38×15 박스 하나가 16KB, 실제 필요량의 7.2배 | A2 |
| 5 | cv는 화면 밖을 공짜로 만든다 | 조건부 | 리레이아웃은 여전히 O(N), 초기 로드는 항목당 비용이 측정 한계 아래 | A4, C1 |
| 6 | cv는 layout과 paint를 건너뛴다 | 조건부 | 무효화 종류가 정한다. 폭 변경이면 Layout 이득, 항목 스타일만이면 손해 | C2 |
| 7 | cv의 이득은 Style 절감이다 | 틀림 | `:root` 커스텀 프로퍼티라는 최악 조건에서만. 흔한 경우 Style이 0.41 → 9.29ms로 증가 | C5 |
| 8 | cv는 켜 두면 손해 볼 일이 없다 | 틀림 | 세로 밀기에서 레이아웃 패스 8 → 17회, 색만 바꿔도 0 → 9회 | C2 |
| 9 | `contain-intrinsic-size`의 `auto`가 성능을 좌우한다 | 틀림 | 리레이아웃에서 차이 3~8%, 반복 편차보다 작다 | C2 |
| 10 | `contain-intrinsic-size`는 성능용이다 | 틀림 | 정확성용. CPU 비용은 같고 없으면 스크롤바가 망가진다 | A4 |
| 11 | `contain-intrinsic-size`는 값만 잘 주면 된다 | 조건부 | 실제 페이지에선 줄 값이 없다. 오차 32% · 47% · 89% | C6 |
| 12 | cv는 아무 페이지에나 붙일 수 있다 | 틀림 | 반복 단위가 `<tr>`이면 `contain: size`가 안 걸린다 | C6 |
| 13 | cv는 조건에 따라 손해다 | 틀림 | 실제 3곳에서 손해 없음(5.9~11.3배 이득, 최악도 무승부) | C6 |
| 14 | cv는 초기 렌더를 빠르게 한다 | 조건부 | 레이아웃이 병목일 때만. LCP는 3곳 중 1곳(24초 → 2.2초) | C7 |
| 15 | LCP가 이미지면 cv는 소용없다 | 틀림 | 콜드 12~28%, warm 최대 38% 빨라진다. 요청 시작만 415~579ms 앞당겨진다 | C8 |
| 16 | 화면 밖 이미지를 안 받아서 빨라진 것 | 틀림 | 요청 수·바이트 동일. 원인은 LCP 이미지의 `lazy` | C8 |
| 17 | `eager`로 바꾸면 LCP가 그만큼 빨라진다 | 조건부 | 요청은 300~400ms 앞당겨지나 LCP는 2~23%. 전송 123 → 303ms | C8b |
| 18 | cv를 붙이면 이미지 LCP가 빨라진다 | 조건부 | lazy + 수백 ms 레이아웃 둘 다 필요 | C8c |
| 19 | 레이아웃만 충분히 비싸면 빨라진다 | 틀림 | Turku는 424ms를 아꼈는데 요청이 38ms 밀렸다 | C8d |
| 20 | LCP는 이미지가 준비된 시각과 비슷하다 | 틀림 | 준비 뒤 426ms(78%)를 더 기다린다. 레이아웃 몫은 22% | C8e |
| 21 | 늦는 건 큰 이미지 디코드 때문이다 | 틀림 | `Decode Image`가 `PaintImage`보다 먼저 온 적 0/12 | C8f |
| 22 | cv는 첫 페인트를 늦출 리 없다 | 틀림 | warm에서 160~248ms 늦춘다. FCP 전 Layout 1ms → 200ms | C8j~C8l |
| 23 | 늦추는 폭은 cv 요소 수가 정한다 | 틀림 | r = 0.11. FCP는 파싱 완료를 따라간다(r = 0.97~0.99) | C8o |
| 24 | 화면 아래쪽에만 조금 걸면 된다 | 맞다 | 요소당 23~58µs, 865개 +25ms → 373개 +9ms | C8n |
| 25 | `auto`는 성능에 거의 영향이 없다 | 조건부 | 첫 페인트 전엔 선행 비용 빈도 75% 대 45%(p = 0.040, 약한 증거) | C8l |
| 26 | 레이어를 많이 만들면 어느 지점에서 급격히 나빠진다 | 틀림 | 3,000개까지 불연속 없음. commit 레이어당 14.4µs | C3 |
| 27 | INP 200ms를 통과하면 반응이 빠르다 | 틀림 | 양보만 해도 통과한다. `yield`는 INP 16ms 미만인데 화면 갱신은 210.4ms | A5 |

7·14번처럼 통념 자체가 틀렸다기보다 조건이 빠진 경우도 있다. 이런 항목은 반박하기보다 빠진 조건을 되찾아 붙이는 쪽이 정확했다.

## 7. 한계와 다음 단계

- 엔진은 Chrome 152 하나뿐이다. Safari·Firefox 대조는 없고, Skia Graphite 경로도 못 봤다(이 환경은 Ganesh).
- 세션 사이 드리프트는 못 고쳤다. 엄격 격리로 줄어드는 건 세션 안의 흩어짐뿐이라, 같은 설정이 세션에 따라 8~51µs 대 39~963µs로 갈렸다.
- 스크롤은 JS `scrollTop` 대입으로 대체했고, 실제 휠·터치 스크롤은 재지 않았다.
- paint chunk는 직접 세지 못했다. `internals.layerTreeAsText()`는 테스트 빌드 전용이라 소스를 빌드해야 쓸 수 있다.
- 필드 데이터가 없다. 실험실에서는 실행 안의 짝지어진 지표가 유의했지만 절대 LCP는 그렇지 않았고, 실사용자 p75 분포는 못 봤다.
- 이미지 LCP 표본이 위키 계열에 몰렸다. 위키 밖 후보는 텍스트 LCP였거나 봇 차단에 걸렸다.
- cv의 선행 비용을 피하는 방법이 없다. 화면 아래쪽에만 거는 CSS가 없고, `auto`를 빼면 빈도는 75%에서 45%로 줄지만 대신 스크롤바 정확성을 잃는다.

코드: [github.com/kasangyong/browser-rendering](https://github.com/kasangyong/browser-rendering)
