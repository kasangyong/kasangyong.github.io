---
title: "XRP 선물 자동매매 봇: 전략 14개를 백테스트로 걸러 하나를 고르다"
date: 2026-10-23 11:10:00 +0900
categories: [Trading, Backtesting]
tags: [python, pandas, ccxt, ta, binance, futures, backtesting, overfitting]
math: true
image:
  path: /assets/img/posts/binance-backtest/thumb-poster.jpg
  alt: 전략 후보 14개 중 하나만 남고 그 전략으로 봇이 돌아가는 XRP 자동매매 봇 소개 애니메이션
---

바이낸스 선물에서 XRP/USDT를 15분봉 추세 전략으로 자동 매매하는 파이썬 봇을 만들었고, 전략을 고르는 데 쓴 백테스트 스크립트도 9개 짰다.

| 항목 | 값 |
|---|---|
| 대상 | XRP/USDT 선물, 15분봉 |
| 진입 | ADX(14) > 30, EMA9/EMA20 방향, EMA9 기울기, RSI(14) 50 기준. 조건이 처음 성립한 봉만 |
| 청산 | 손절 -2%, +1.5%부터 고점 대비 0.5% 되돌림 트레일링, 안전 상한 +15% |
| 레버리지 / 비중 | 15x / 잔고의 30% |
| 후보 비교 | 14개 전략 × 2개 1년 구간 (2025-03~2026-03, 2025-08~2026-08) |
| 비용 가정 | 수수료 편도 0.04%, 슬리피지·펀딩비 미반영 |
| 코드 | 봇 708줄 + 전략 117줄, 백테스트 9개 2,243줄 |
| 결과 파일 | 저장소에 없음 (`*.csv`, 매매 로그 모두 `.gitignore`) |

결과 파일이 저장소에 없어서 아래에 나오는 수익률 이야기는 전부 `work_log.md`에 적힌 문장에 기댄다. 백테스트 출력은 콘솔에만 찍혔고 커밋하지 않았다.

{% include embed/video.html src='/assets/img/posts/binance-backtest/intro.mp4' poster='/assets/img/posts/binance-backtest/intro-poster.jpg' title='XRP 선물 자동매매 봇 소개: 전략 14개 중 하나를 골라 봇으로 돌린다' %}

## 1. 저장소 두 개의 관계

| 저장소 | 시기 | 내용 |
|---|---|---|
| `binance_inv` | 커밋 1개, 2026-01-30 | BTCUSDT, 테스트넷 전용. RSI+MACD, 볼린저 밴드, 이동평균 교차 3전략. `ccxt` 없이 REST를 직접 HMAC-SHA256으로 서명 |
| `binance` | 2026-03-02 ~ 2026-08-21 (로그 기준, 첫 커밋 03-19) | XRP/USDT, `ccxt`로 실계좌 연결. 이 글의 대상 |

`binance_inv`의 `api.py`는 `BASE_URL = 'https://testnet.binancefuture.com'`으로 고정돼 있고 남은 로그도 API 키 검증, 캔들 수집, 봇 시작·중지 정도다. 두 저장소는 파일 구성도 의존성도 다르고 공유하는 코드가 없으니, 먼저 만든 테스트넷 연습판과 새로 짠 실계좌판으로 보면 된다.

`binance`의 전략은 네 번 바뀌었다.

| 날짜 (작업 로그·커밋 기준) | 판단 방식 | 근거 |
|---|---|---|
| 03-02 | 로컬 Ollama `gemma2:2b`에 지표를 넘겨 LONG/SHORT/HOLD와 레버리지(1~10배)를 JSON으로 받음 | 없음 |
| 03-04 | 규칙 기반 ADX+EMA로 교체, LLM 제거 | 최근 30일·60일, 후보 28개 |
| 03-30 | ADX > 25 + EMA | 1년, 8전략 (`backtest_1year.py`) |
| 08-04 | ADX > 30 + EMA + RSI, 트레일링 스탑 도입 | 2개 1년 구간, 후보 14개 (`backtest_strategy_lab3.py`) |

LLM 판단을 버리고 규칙 전략으로 넘어간 이유는 로그에 "백테스트로 검증된 규칙 전략으로 전격 교체" 한 줄만 남아 있다.

## 2. 전략: 세 조건이 처음 겹치는 봉

라이브 신호는 `strategy_analyzer.py`의 `get_signal()`이 만들고, 지표 계산은 `ta` 라이브러리에 맡겼다.

```python
if adx > 30:
    # LONG: EMA9가 EMA20 위 + EMA9 상승 중 + 이전 봉에서는 조건 미충족 (첫 신호) + RSI > 50
    long_cond  = (ema9_now > ema20_now) and (ema9_now > ema9_prev) and (rsi > 50)
    short_cond = (ema9_now < ema20_now) and (ema9_now < ema9_prev) and (rsi < 50)

    prev2_ema9  = df.iloc[-3]['ema9']
    prev2_ema20 = df.iloc[-3]['ema20']
    prev_long  = (ema9_prev > prev['ema20']) and (ema9_prev > prev2_ema9)
    prev_short = (ema9_prev < prev['ema20']) and (ema9_prev < prev2_ema9)

    if long_cond and not prev_long:
        decision = "LONG"
    elif short_cond and not prev_short:
        decision = "SHORT"
```

(`reason` 문자열 대입 줄은 생략했다.)

세 조건은 맡은 일이 다르다.

- **ADX > 30**: 방향은 따지지 않고 추세가 강한지만 본다. 횡보장에서 나오는 EMA 교차를 여기서 걸러낸다.
- **EMA9 vs EMA20, EMA9 기울기**: 방향을 정한다. 단기선이 장기선 위에 있으면서 아직 오르는 중이어야 한다.
- **RSI 50**: 같은 방향인지 한 번 더 확인한다. README에서는 이걸 "3중 확인"이라고 부른다.

마지막의 `not prev_long`이 "첫 신호만" 규칙이다. 조건이 10봉 연속으로 성립해도 진입은 첫 봉에서 한 번만 하니, 한참 진행된 추세에 뒤늦게 올라타는 일이 없다.

포지션이 있는 동안 봇은 새 신호를 무시한다. 반대 신호가 나와도 청산하지 않고 손절이나 트레일링이 처리하게 둔다.

```python
# 포지션이 있으면 SL/TP가 처리할 때까지 유지 (반대 신호로 청산 금지)
if self.position is not None:
    print(f"포지션 유지 중 ({self.position}). SL/TP 대기.")
    return
```

## 3. 백테스트 엔진

{% include embed/video.html src='/assets/img/posts/binance-backtest/explainer.mp4' poster='/assets/img/posts/binance-backtest/explainer-poster.jpg' title='신호가 난 봉의 종가에 들어가 다음 봉부터 SL을 먼저, TP를 나중에 보고 청산한 뒤 잔고를 갱신하는 백테스트 엔진의 한 바퀴' loop=true %}

엔진은 라이브러리로 따로 빼지 않고 스크립트마다 `backtest()`/`run()` 함수로 복사해 두었다. 아래는 최종 후보 비교에 쓴 `backtest_strategy_lab3.py` 버전이다.

```python
def backtest(ls, ss, df, lev, ratio, sl, tp):
    bal, pos, ep = START, None, 0.0
    trades, max_bal, max_dd = [], START, 0

    for i in range(1, len(df)):
        row = df.iloc[i]
        high, low = row['high'], row['low']

        if pos:
            exit_p, reason = None, None
            if pos == 'LONG':
                if sl is not None and low <= ep * (1 - sl):
                    exit_p, reason = ep * (1 - sl), 'SL'
                elif high >= ep * (1 + tp):
                    exit_p, reason = ep * (1 + tp), 'TP'
            # ... SHORT 대칭 분기 생략

            if exit_p is not None:
                pct = (exit_p - ep) / ep if pos == 'LONG' else (ep - exit_p) / ep
                amt = bal * ratio
                pnl = amt * lev * pct - amt * lev * FEE * 2
                bal += pnl
                # ... 낙폭 기록 생략
                pos = None
                if bal < 1.0:
                    bal = 0
                    break
            continue

        if ls[i] and bal > 1.0:
            pos = 'LONG'; ep = row['close']
        elif ss[i] and bal > 1.0:
            pos = 'SHORT'; ep = row['close']
```

체결을 어떻게 가정했는지 표로 정리했다.

| 가정 | 코드 | 방향 |
|---|---|---|
| 진입가 | 신호가 난 봉의 종가 `row['close']` | 낙관 쪽. 실제로는 그 뒤에 주문이 나간다 |
| 청산 시작 | 진입 봉에서는 청산을 보지 않고 다음 봉부터 (`continue`) | 진입 봉 안의 움직임은 버린다 |
| 한 봉에서 SL과 TP가 다 닿으면 | SL을 먼저 본다 | 보수 쪽 |
| 청산가 | 정확히 SL/TP 가격 | 낙관 쪽. 갭이나 슬리피지가 없다 |
| 청산한 봉에서 재진입 | 하지 않는다 (`continue`) | 중립 |
| 포지션 크기 | 매번 현재 잔고 × `ratio` | 복리 |

### 수수료

`FEE = 0.0004`는 편도 0.04%다. 증거금이 아닌 명목 금액(`amt * lev`)에 진입과 청산 때 한 번씩 매기기 때문에 레버리지를 올리면 수수료도 같이 커진다. 최종 설정(비중 30%, 15x)이면 한 번 왕복할 때 잔고 대비 비용이 이 정도다.

$$
0.30 \times 15 \times 0.0004 \times 2 = 0.0036 = 0.36\%
$$

가격 움직임으로 치면 0.08%다. 트레일링이 켜진 뒤 가장 얕게 청산되면 가격 이익이 약 +1%(+1.5%에서 0.5% 되돌림)이니, 이때는 이익의 8% 정도가 수수료로 나간다.

모델링에서 빠진 것도 있다. 슬리피지 항이 없고 무기한 선물의 펀딩비도 없으며, 강제청산은 `backtest_jackpot.py`에만 들어 있다. 다만 lab3 그리드는 레버리지가 최대 20x라서 손절(최대 3%)이 청산가보다 먼저 닿기는 한다.

### 룩어헤드 방지: 상위 시간봉 필터

EMA, RSI, ADX는 과거 값만 쓰는 지표라 전체 시계열에 한 번에 계산해도 미래가 섞이지 않는다. 미래가 섞일 여지가 있는 곳은 15분봉에 1시간봉 EMA50을 붙이는 후보 H(`1시간필터+ADX`)다.

```python
htf = df.set_index('timestamp')['close'].resample('1h').last().dropna().to_frame('close')
htf['ema50_htf'] = ta.trend.EMAIndicator(htf['close'], window=50).ema_indicator().shift(1)
merged = pd.merge_asof(df[['timestamp']], htf[['ema50_htf']].reset_index(), on='timestamp', direction='backward')
```

바이낸스 캔들의 타임스탬프는 봉이 열린 시각이고 pandas `resample('1h')`도 기본값으로 구간 시작 시각을 라벨로 쓴다. 그래서 `10:00` 라벨이 붙은 1시간봉의 종가는 `10:45` 15분봉의 종가, 곧 11:00에 확정되는 값이다. `shift(1)` 없이 `merge_asof(backward)`로 붙이면 `10:00` 15분봉(10:15에 판단)이 45분 뒤 가격으로 만든 EMA를 보게 된다.

`shift(1)`을 걸면 `10:00` 15분봉은 `09:00` 라벨, 즉 10:00에 확정된 종가까지 반영한 EMA를 받으니 판단 시각보다 앞선 값만 쓰게 된다.

## 4. 트레일링 스탑과 봉 안의 경로

최종 트레일링 파라미터(+1.5%/0.5%)를 검증한 코드는 저장소에 없다(6절).

아래는 트레일링 비교 스크립트(`backtest_exit_test.py`)의 LONG 분기로, 파라미터는 +2% 활성화에 1% 추적이다.

```python
if pos == 'LONG':
    if high > best_p:
        best_p = high
    # +2% 달성 후 고점 -1% 추적
    if best_p >= ep * 1.02:
        new_t = best_p * (1 - 0.01)
        if new_t > trail_sl:
            trail_sl = new_t
    is_trail = (trail_sl > 0) and (low <= trail_sl)
```

한 봉 안에서 고점으로 추적선을 올린 다음 같은 봉의 저가로 추적선 터치를 확인하니, 고가가 먼저 오고 저가가 나중에 왔다고 가정하는 셈이다. 이 스크립트는 5분봉을 쓰지만 봉 내부 경로 문제는 똑같다. 봉에는 시가·고가·저가·종가 네 값만 있어서 봉 안에서 무엇이 먼저였는지 알 수 없다.

추적폭이 좁을수록 이 가정에 결과가 크게 흔들린다. 봉 하나의 고저 폭이 추적폭보다 크기만 하면 같은 봉 안에서 "고점 찍고 정확히 추적선에서 익절"이 일어나기 때문이다.

15분봉 그리드는 따로 기록이 남아 있다. 작업 로그를 보면 추적폭 0.3%가 그리드에서 수십만% 수익률로 나왔고, 이를 "15분봉 캔들 백테스트 특유의 복리 착시(봉 내부 경로를 알 수 없어 생기는 비현실적 결과)"로 판단해 버렸다고 적혀 있다. README도 0.3~0.7% 구간에서 결과가 민감하게 변한다고 적은 뒤 0.5%를 골랐다.

손절은 트레일링보다 먼저 검사하기 때문에 한 봉에서 손절가와 추적선이 둘 다 닿으면 손실로 친다. 이 부분은 보수적인 쪽이다.

5분봉으로 해상도를 올려 본 적도 있다. `backtest_adaptive_15m.py` 첫 주석에 "5분봉 테스트에서 모든 모드가 손실 났음. 라이브 봇이 사용하는 15분봉으로 재검증."이라고 남아 있는데, 5분봉에서 전 모드가 손실이었다는 이 주석뿐이고 15분봉 결과는 남아 있지 않다.

라이브 봇의 트레일링은 같은 규칙을 30초마다 `fetch_ticker()`의 현재가로 돌린다. 봉 안 경로 문제는 없는 대신 30초 사이에 찍힌 고점은 놓친다.

```python
if current_price > self.peak_price:
    self.peak_price = current_price
if self.peak_price >= self.entry_price * (1 + TRAIL_START_PCT):
    candidate_trail = self.peak_price * (1 - TRAIL_DIST_PCT)
    if candidate_trail > self.trail_stop:
        self.trail_stop = candidate_trail
trail_hit = self.trail_stop > 0 and current_price <= self.trail_stop
```

## 5. 과적합 대응: 두 구간과 최악 구간 기준

lab3는 두 단계로 돌아간다.

1. **STAGE 1**: 14개 후보를 같은 조건(5x, 잔고 20%, SL 1.5%, TP 3.5%)으로 두 구간에 돌린다. 리스크 설정을 묶어 두고 신호 품질만 비교하려는 단계다.
2. **STAGE 2**: 두 구간 모두 수익이 양수인 후보만 SL 5개 × TP 4개 × 레버리지 3개 × 비중 3개 = 180조합 그리드에 넣는다.

STAGE 2에서 눈여겨볼 건 정렬 기준이다.

```python
avg_ret = sum(r['ret'] for r in combo_rets.values()) / 2
min_ret = min(r['ret'] for r in combo_rets.values())
# ...
results.sort(key=lambda x: x['min_ret'], reverse=True)
```

줄은 평균 대신 두 구간 중 나쁜 쪽의 수익률(`min_ret`)로 세운다. 그러면 한 구간에서 크게 벌고 다른 구간에서 잃는 조합은 평균이 높아도 위로 올라오지 못한다. 저장소 `CLAUDE.md`에도 "한 구간에서만 잘 되는 과최적화 후보를 걸러내기 위함"이라고 적어 두었다.

### 두 구간은 독립이 아니다

![두 검증 구간의 기간](/assets/img/posts/binance-backtest/periods-light.png){: .light }
![두 검증 구간의 기간](/assets/img/posts/binance-backtest/periods-dark.png){: .dark }
_구간 B의 정확한 시작일은 캐시를 처음 만든 시각으로 정해져 저장소에 남아 있지 않다. 그림은 월 단위다_

구간 A는 `2025-03-01T00:00:00Z`부터 1년으로 코드에 고정돼 있다. 구간 B는 `backtest_strategy_lab.py`를 처음 실행했을 때 `datetime.now(timezone.utc)`에서 365일을 빼서 받아 온 데이터를 `xrp_15m_1year_fresh.csv`로 캐시해 둔 것이고, 라벨은 "2025-08~2026-08"이다.

두 구간은 2025-08부터 2026-03까지 약 7개월이 겹치니, 1년 중 절반 넘게 같은 캔들을 쓴 셈이다. 그래서 "두 구간에서 다 이겼다"고 해도 독립된 확인을 두 번 한 건 아니다. 게다가 그리드는 두 구간을 모두 보고 조합을 골랐으므로 어느 쪽도 학습에 안 쓴 홀드아웃이 되지 못한다. 최악 구간 기준으로 한쪽 구간에만 맞춘 조합은 걸러지지만, 둘이 공유하는 7개월에 맞춘 조합은 그대로 남는다.

이 문제를 직접 다루려면 겹치지 않는 구간으로 나누거나, 앞 구간에서 고르고 뒤 구간에서 한 번만 확인하는 워크포워드 방식을 써야 하는데 저장소에는 그런 구현이 없다.

## 6. 후보 비교 결과

14개 후보를 계열별로 묶으면 이렇다.

| 계열 | 후보 |
|---|---|
| ADX + EMA 계열 | A. ADX25+EMA+RSI, A2. ADX30+EMA+RSI, H. 1시간 EMA50 필터 + ADX |
| 이동평균 / 채널 | B. 트리플 EMA(5/13/34) 정렬, C. Keltner 돌파, G. OBV + Donchian |
| 추세 전환 | D. 일목균형표, E. PSAR + ADX, I. Aroon 교차, K. Vortex 교차 |
| 모멘텀 | F. RSI2 눌림목, J. CCI ±100 돌파, L. TRIX 교차, M. STC |

결과 표는 저장소에 없고 작업 로그(2026-08-04)에 "신규 전략은 대부분 기존 ADX(>30)+EMA+RSI 전략보다 못했고, 기존 전략이 두 구간 모두에서 가장 견고했음"이라는 결론만 적혀 있다. 같은 신호에서 SL만 1.5%에서 2.0%로 바꿨더니 수익률이 크게 좋아졌다는 기록도 있는데, 둘 다 수치 없이 문장으로만 남았다.

레버리지 결정(2026-08-21)도 로그에 문장으로만 있다. 로그를 보면 최근 구간 기대수익은 10x가 더 높고 낙폭은 15x가 더 컸는데도 "역대급 한방"을 노리고 15x를 택했다. 그리드 스크립트를 커밋하지 않아서 재현은 안 된다. `get_signal()` 독스트링에도 "최신구간 기대치는 10x보다 낮음"이 그대로 남아 있다.

저장소에 있는 lab3 그리드는 고정 TP만 쓰고 레버리지 후보도 5/10/20x라 15x가 빠져 있다. 트레일링 비교 스크립트들은 구간 A 하나만 쓰고, 전략도 이전 것(ADX > 25)에 +2%/1% 파라미터다. 최종 설정인 "ADX>30 + 트레일링 +1.5%/0.5% + 15x"를 두 구간에 돌린 코드는 저장소에서 찾지 못했다. 수수료는 모든 백테스트 스크립트가 `FEE = 0.0004`라 같은 값이었을 가능성이 높지만 코드로 확인하지는 못했다.

## 7. 백테스트와 실거래 코드가 다른 지점

같은 전략을 두 번 구현했다. 백테스트는 pandas 벡터 연산으로, 라이브는 `get_signal()`의 스칼라 비교로 짰는데 둘을 나란히 놓으면 차이가 보인다.

| 항목 | 백테스트 | 라이브 봇 |
|---|---|---|
| 판단 봉 | 마감된 15분봉 | `fetch_ohlcv(limit=200)`의 마지막 봉 |
| 판단 주기 | 봉마다 1회 | 5분마다 |
| 지표 계산 범위 | 1년 전체 시계열 | 최근 200봉. ADX의 Wilder 평활은 초기값 영향이 남아 값이 다를 수 있다 |
| "첫 신호" 판정 | 직전 봉의 전체 조건(ADX, RSI 포함) | 직전 봉의 EMA 조건만 |
| 진입가 | 신호 봉 종가 | 주문 직전 `ticker['last']`로 시장가 주문 |
| 청산 감시 | 봉의 고가/저가 | 30초마다 현재가 |
| 청산 체결 | 정확히 목표가 | `reduceOnly` 시장가 |

### 진행 중인 봉

바이낸스 klines API는 아직 닫히지 않은 현재 봉을 마지막 원소로 돌려준다. 라이브 봇은 `df.iloc[-1]`을 `curr`로 쓰기 때문에 진행 중인 봉의 현재가로 EMA, RSI, ADX를 계산하게 되고, 5분마다 도니 15분봉 하나를 세 번, 매번 다른 "종가"로 판단한다. 백테스트는 마감된 종가로 한 번만 본다. 그래서 라이브가 봉 중간에 진입했는데 막상 그 봉이 마감될 때는 조건이 풀려 있을 수 있다.

### 첫 신호 판정이 다르다

백테스트에서는 전체 신호 시리즈를 한 칸 밀어 "첫 신호"를 골라낸다.

```python
def first_signal_only(sig):
    return (sig & ~sig.shift(1).fillna(False)).values
```

`sig`에는 ADX > 30과 RSI > 50까지 들어 있는데, 라이브의 `prev_long`은 EMA 두 조건만 본다. EMA가 이미 정렬된 상태에서 ADX가 이번 봉에 30을 넘었다고 하자. 백테스트는 직전 봉 신호가 거짓이니 진입하고, 라이브는 직전 봉 EMA가 이미 참이니 진입하지 않는다. RSI가 50을 다시 넘을 때도 마찬가지다. 라이브 쪽 조건이 더 엄격하므로 라이브 신호는 백테스트 신호의 부분집합이 된다.

차이가 얼마나 되는지 보려고 구간 A(2025-03-01~2026-03-01)의 XRP/USDT 15분봉을 바이낸스 선물 API에서 다시 받아 두 규칙의 진입 신호 봉 수(포지션 보유 중 무시되는 신호 포함)를 셌다. 계산은 저장소 밖에서 했다. 지표 4개(EMA9, EMA20, RSI14, ADX14)를 마감봉 기준으로 썼고, 진행 중 봉과 200봉 웜업은 반영하지 않았다.

| | LONG | SHORT | 합계 |
|---|---|---|---|
| 백테스트 규칙 | 633 | 753 | 1,386 |
| 라이브 규칙 | 435 | 505 | 940 |

라이브 규칙이 내는 신호는 전부 백테스트 신호 안에 있었고, 백테스트 신호 1,386개 중 446개(32%)는 라이브 봇이라면 진입하지 않았을 봉이었다.

![XRP/USDT 15분봉 진입 신호](/assets/img/posts/binance-backtest/signals-light.png){: .light }
![XRP/USDT 15분봉 진입 신호](/assets/img/posts/binance-backtest/signals-dark.png){: .dark }
_저장소 외 재계산, 마감봉 기준. 2026-02-27~02-28. 채운 삼각형은 두 규칙 모두, 빈 삼각형은 백테스트 규칙만 진입하는 봉. 이 이틀에 각각 7개, 3개_

빈 삼각형 세 개(02-27 12:00, 02-28 05:30, 02-28 19:30)는 모두 ADX가 직전 봉 29점대에서 이번 봉에 30을 넘긴 경우이고, EMA 정렬은 그 전부터 이어지던 중이었다. 백테스트가 진입할 수 있는 봉 가운데 3분의 1가량을 라이브 봇은 진입 후보로 보지 않는 셈이다. 포지션 보유 중 무시되는 신호가 있으니 실제 거래 수 차이는 이보다 작을 수 있다. 그 신호들이 수익 쪽이었는지 손실 쪽이었는지는 재보지 않았다.

그림에서 보이는 게 하나 더 있다. 02-27 19:00부터 23:00까지 ADX가 40대에 머문 채 가격이 횡보하는 동안 SHORT 신호가 네 번 났고, 자정에는 LONG이 났다. EMA9가 평평해지면 "EMA9 하락 중" 조건이 한 봉 꺼졌다가 다시 켜지는데, 그럴 때마다 "첫 신호"가 새로 생긴다. 포지션을 들고 있을 때는 무시되지만 손절 직후라면 같은 횡보 안에서 또 진입한다.

## 8. 실거래였나, 모의였나

| 근거 | 내용 |
|---|---|
| 코드 | `ccxt.binance`에 `defaultType: 'future'`, 테스트넷이나 샌드박스 설정 없음. API 키가 없을 때만 Dry-run |
| `CLAUDE.md` | "`trading_bot.py`와 `strategy_analyzer.py`는 실제 바이낸스 선물 계좌에 연결되어 있다" |
| `work_log.md` 2026-08-21 | "운영 중 발견한 버그" 항목 |
| `backtest_jackpot.py` | 실험 1의 제목이 "현재 라이브 봇 설정 정확 재현": 75x, 손절 없음, TP 2.5%, 비중 85% |

실계좌로 돌렸다는 근거는 있지만 실거래 성과는 없다. 매매 기록 `trade_history.log`를 `.gitignore`에 넣어 둬서 진입·청산 내역도 손익도 저장소에 남지 않았다. 작업 로그에는 8월 4일에 레버리지를 75x에서 10x로 내렸다고 적혀 있는데, 75x로 돌리던 때의 결과도 기록이 없다.

운영 버그 하나는 코드에 흔적이 남아 있다. 사용자가 거래소 앱에서 포지션을 직접 닫으면 봇은 그걸 모른 채 30초마다 청산 주문을 다시 냈다. 거래소는 이를 `ReduceOnly Order is rejected`(-2022)로 거절했고 그때마다 텔레그램 알림이 갔다. 그래서 `close_position()`이 이 에러를 "거래소에 포지션이 없다"는 신호로 받아들이도록 고쳤다.

```python
except Exception as e:
    err_str = str(e)
    if "ReduceOnly Order is rejected" in err_str or "-2022" in err_str:
        # 거래소에 실제 포지션이 없다는 뜻 (수동 청산 등으로 이미 닫힘) → 내부 상태만 정리
        # ... 알림 전송 생략
        self.position = None
        self.entry_price = 0
        self.order_qty = 0
        self.peak_price = 0
        self.trail_stop = 0
        return
```

재시작은 `sync_position()`이 처리한다. 시작할 때 `fetch_positions()`의 `positionAmt` 부호를 보고 방향을 복원하고, 재시작 전 고점은 알 길이 없으니 트레일링은 진입가부터 다시 무장한다.

```python
# 재시작 전 가격 흐름은 알 수 없으므로 트레일링은 진입가부터 보수적으로 재무장
self.peak_price = self.entry_price
self.trail_stop = 0
```

두 경로 모두 봇이 메모리에 들고 있는 상태와 거래소 상태가 어긋난 경우를 다룬다. 한쪽은 시작할 때, 다른 쪽은 에러가 난 뒤에 맞출 뿐이고 거래소 포지션과 주기적으로 대조하는 코드는 없다.

## 9. 한계

- 결과를 재현할 수 있는 형태로 남기지 않았다. 캔들 CSV와 백테스트 출력을 커밋하지 않았고 최종 설정을 고른 그리드 스크립트도 없어서, 수치라고는 작업 로그 문장뿐이다.
- 검증 구간이 7개월 겹치고 홀드아웃도 없다. 최악 구간 기준으로는 한쪽에 맞춘 조합만 걸러진다.
- 최종 레버리지를 검증 기준과 다르게 골랐다. 로그상 최근 구간 기대수익은 10x가 더 높았고 낙폭은 15x가 더 컸다(구간 미기재). 근거가 된 그리드는 재현할 수 없다.
- 트레일링 결과가 봉 내부 경로 가정에 기댄다. README에도 0.3~0.7% 추적폭에서 결과가 민감하고, 실제 슬리피지에서는 백테스트보다 보수적인 성과를 예상해야 한다고 적어 두었다.
- 비용 모델에는 수수료만 있고 슬리피지와 펀딩비는 없다.
- 백테스트와 라이브의 신호 규칙이 다르다. 첫 신호를 판정하는 범위가 다르고 라이브는 진행 중인 봉을 쓴다. 재계산 기준으로 백테스트 신호의 약 32%가 라이브 규칙에서는 나오지 않는다.
- 실거래 성과 기록이 없다.

코드: [github.com/kasangyong/binance](https://github.com/kasangyong/binance)
