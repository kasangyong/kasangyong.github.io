---
title: "XRP 선물 자동매매 봇: 14개 전략을 두 1년 구간에 돌려 하나를 고르기까지"
date: 2026-10-23 11:10:00 +0900
categories: [Trading, Backtesting]
tags: [python, pandas, ccxt, ta, binance, futures, backtesting, overfitting]
math: true
image:
  path: /assets/img/posts/binance-backtest/periods-light.png
  alt: 두 검증 구간의 기간과 겹치는 부분
---

바이낸스 선물에서 XRP/USDT를 15분봉 추세 전략으로 자동 매매하는 파이썬 봇과, 전략을 고르는 데 쓴 백테스트 스크립트 9개를 만들었다.

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

결과 파일이 저장소에 없어 아래 수익률 관련 서술은 모두 `work_log.md` 문장이다. 백테스트 출력은 콘솔로만 나오고 커밋되지 않았다.

## 1. 저장소 두 개의 관계

| 저장소 | 시기 | 내용 |
|---|---|---|
| `binance_inv` | 커밋 1개, 2026-01-30 | BTCUSDT, 테스트넷 전용. RSI+MACD, 볼린저 밴드, 이동평균 교차 3전략. `ccxt` 없이 REST를 직접 HMAC-SHA256으로 서명 |
| `binance` | 2026-03-02 ~ 2026-08-21 (로그 기준, 첫 커밋 03-19) | XRP/USDT, `ccxt`로 실계좌 연결. 이 글의 대상 |

`binance_inv`의 `api.py`는 `BASE_URL = 'https://testnet.binancefuture.com'`으로 고정돼 있고, 남은 로그도 API 키 검증과 캔들 수집, 봇 시작·중지뿐이다. 두 저장소는 파일 구성과 의존성이 다르고 공유하는 코드가 없다. 먼저 만든 테스트넷 연습판과, 새로 짠 실계좌판으로 보는 게 맞다.

`binance`의 전략은 네 번 바뀌었다.

| 날짜 (작업 로그·커밋 기준) | 판단 방식 | 근거 |
|---|---|---|
| 03-02 | 로컬 Ollama `gemma2:2b`에 지표를 넘겨 LONG/SHORT/HOLD와 레버리지(1~10배)를 JSON으로 받음 | 없음 |
| 03-04 | 규칙 기반 ADX+EMA로 교체, LLM 제거 | 최근 30일·60일, 후보 28개 |
| 03-30 | ADX > 25 + EMA | 1년, 8전략 (`backtest_1year.py`) |
| 08-04 | ADX > 30 + EMA + RSI, 트레일링 스탑 도입 | 2개 1년 구간, 후보 14개 (`backtest_strategy_lab3.py`) |

LLM 판단에서 규칙 전략으로 넘어간 이유는 로그에 "백테스트로 검증된 규칙 전략으로 전격 교체"라고만 적혀 있다.

## 2. 전략: 세 조건이 처음 겹치는 봉

라이브 신호는 `strategy_analyzer.py`의 `get_signal()`이 만든다. 지표는 `ta` 라이브러리로 계산한다.

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

역할은 셋으로 나뉜다.

- **ADX > 30**: 방향과 무관하게 추세가 강한지만 본다. 횡보장의 EMA 교차를 걸러내는 문턱이다.
- **EMA9 vs EMA20, EMA9 기울기**: 방향. 단기선이 장기선 위에 있으면서 아직 올라가는 중이어야 한다.
- **RSI 50**: 같은 방향을 한 번 더 확인한다. README는 이걸 "3중 확인"이라고 부른다.

마지막의 `not prev_long`이 "첫 신호만" 규칙이다. 조건이 10봉 연속 성립해도 진입은 첫 봉 한 번이다. 이미 한참 진행된 추세에 늦게 타는 걸 막는다.

봇은 포지션이 있으면 새 신호를 무시한다. 반대 신호가 나와도 청산하지 않고 손절이나 트레일링에 맡긴다.

```python
# 포지션이 있으면 SL/TP가 처리할 때까지 유지 (반대 신호로 청산 금지)
if self.position is not None:
    print(f"포지션 유지 중 ({self.position}). SL/TP 대기.")
    return
```

## 3. 백테스트 엔진

엔진은 따로 라이브러리로 분리하지 않고 스크립트마다 `backtest()`/`run()` 함수로 복사돼 있다. 최종 후보 비교에 쓴 `backtest_strategy_lab3.py` 버전이다.

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

체결 가정을 하나씩 정리하면 이렇다.

| 가정 | 코드 | 방향 |
|---|---|---|
| 진입가 | 신호가 난 봉의 종가 `row['close']` | 낙관 쪽. 실제로는 그 뒤에 주문이 나간다 |
| 청산 시작 | 진입 봉에서는 청산을 보지 않고 다음 봉부터 (`continue`) | 진입 봉 안의 움직임은 버린다 |
| 한 봉에서 SL과 TP가 다 닿으면 | SL을 먼저 본다 | 보수 쪽 |
| 청산가 | 정확히 SL/TP 가격 | 낙관 쪽. 갭이나 슬리피지가 없다 |
| 청산한 봉에서 재진입 | 하지 않는다 (`continue`) | 중립 |
| 포지션 크기 | 매번 현재 잔고 × `ratio` | 복리 |

### 수수료

`FEE = 0.0004`는 편도 0.04%이고, 증거금이 아니라 **명목 금액**(`amt * lev`)에 진입·청산 두 번 매긴다. 레버리지를 올리면 수수료도 같이 커진다는 뜻이다. 최종 설정(비중 30%, 15x)에서 한 번 왕복할 때 잔고 대비 비용은 다음과 같다.

$$
0.30 \times 15 \times 0.0004 \times 2 = 0.0036 = 0.36\%
$$

가격으로 환산하면 0.08% 움직임이다. 트레일링이 활성화된 뒤 가장 얕게 청산될 때 가격 이익이 약 +1%(+1.5%에서 0.5% 되돌림)이니, 그 경우 이익의 8% 정도가 수수료로 나간다.

모델링하지 않은 것도 분명하다. 슬리피지 항이 없고, 무기한 선물의 펀딩비가 없고, 강제청산은 `backtest_jackpot.py`에만 있다. lab3 그리드의 최대 레버리지는 20x라 손절(최대 3%)이 청산가보다 먼저 닿는 구조이긴 하다.

### 룩어헤드 방지: 상위 시간봉 필터

지표는 EMA, RSI, ADX처럼 과거 값만 쓰는 것들이라 전체 시계열에 한 번에 계산해도 미래가 섞이지 않는다. 미래가 섞일 수 있는 곳은 15분봉에 1시간봉 EMA50을 붙이는 후보 H(`1시간필터+ADX`)다.

```python
htf = df.set_index('timestamp')['close'].resample('1h').last().dropna().to_frame('close')
htf['ema50_htf'] = ta.trend.EMAIndicator(htf['close'], window=50).ema_indicator().shift(1)
merged = pd.merge_asof(df[['timestamp']], htf[['ema50_htf']].reset_index(), on='timestamp', direction='backward')
```

바이낸스 캔들의 타임스탬프는 봉이 **열린** 시각이고, pandas `resample('1h')`도 기본으로 구간 시작 시각을 라벨로 쓴다. 그래서 `10:00` 라벨의 1시간봉 종가는 `10:45` 15분봉의 종가, 즉 11:00에 확정되는 값이다. `shift(1)` 없이 `merge_asof(backward)`로 붙이면 `10:00` 15분봉(10:15에 판단)이 45분 뒤의 가격으로 만든 EMA를 보게 된다.

`shift(1)`을 하면 `10:00` 15분봉은 `09:00` 라벨, 즉 10:00에 확정된 종가까지의 EMA를 받는다. 판단 시각보다 앞선 값만 쓴다.

## 4. 트레일링 스탑과 봉 안의 경로

최종 트레일링 파라미터(+1.5%/0.5%)를 검증한 코드는 저장소에 없다(6절).

트레일링 비교 스크립트(`backtest_exit_test.py`)의 LONG 분기다. 이 스크립트의 파라미터는 +2% 활성화, 1% 추적이다.

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

한 봉 안에서 고점으로 추적선을 올리고, 같은 봉의 저가로 추적선 터치를 본다. **고가가 먼저, 저가가 나중**이라고 가정하는 셈이다. 이 스크립트는 5분봉이지만 봉 내부 경로 문제는 같다. 봉은 시가·고가·저가·종가 네 값뿐이라 봉 안에서 무엇이 먼저였는지 모른다.

추적폭이 좁을수록 이 가정이 결과를 크게 바꾼다. 봉 하나의 고저 폭이 추적폭보다 크기만 하면 "고점 찍고 정확히 추적선에서 익절"이 같은 봉에서 일어난다.

15분봉 그리드에 대해서는 별도 기록이 있다. 작업 로그에는 추적폭 0.3%가 그리드에서 수십만% 수익률로 나왔고, 이를 "15분봉 캔들 백테스트 특유의 복리 착시(봉 내부 경로를 알 수 없어 생기는 비현실적 결과)"로 판단해 버렸다고 적혀 있다. README도 0.3~0.7% 구간에서 결과가 민감하게 변한다고 적고 0.5%를 골랐다.

손절은 트레일링보다 먼저 검사한다. 한 봉에서 손절가와 추적선이 둘 다 닿으면 손실로 처리한다. 이쪽은 보수적이다.

5분봉으로 해상도를 올리는 시도도 있었다. `backtest_adaptive_15m.py` 첫 주석에 "5분봉 테스트에서 모든 모드가 손실 났음. 라이브 봇이 사용하는 15분봉으로 재검증."이라고 남아 있다. 5분봉에서는 전 모드가 손실이었다는 주석만 있고, 15분봉 결과는 남아 있지 않다.

라이브 봇의 트레일링은 같은 규칙을 30초마다 `fetch_ticker()`의 현재가로 돌린다. 봉 안 경로 문제는 없지만, 30초 사이의 고점은 놓친다.

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

lab3의 구조는 두 단계다.

1. **STAGE 1**: 14개 후보를 같은 조건(5x, 잔고 20%, SL 1.5%, TP 3.5%)으로 두 구간에 돌린다. 리스크 설정을 고정해 신호 자체의 품질만 비교한다.
2. **STAGE 2**: 두 구간 모두 수익이 양수인 후보만 SL 5개 × TP 4개 × 레버리지 3개 × 비중 3개 = 180조합 그리드에 넣는다.

STAGE 2의 정렬 기준이 핵심이다.

```python
avg_ret = sum(r['ret'] for r in combo_rets.values()) / 2
min_ret = min(r['ret'] for r in combo_rets.values())
# ...
results.sort(key=lambda x: x['min_ret'], reverse=True)
```

평균이 아니라 **두 구간 중 나쁜 쪽의 수익률**(`min_ret`)로 줄을 세운다. 한 구간에서 크게 벌고 다른 구간에서 잃는 조합은 평균이 높아도 위로 못 올라온다. 저장소 `CLAUDE.md`에도 "한 구간에서만 잘 되는 과최적화 후보를 걸러내기 위함"이라고 적혀 있다.

### 두 구간은 독립이 아니다

![두 검증 구간의 기간](/assets/img/posts/binance-backtest/periods-light.png){: .light }
![두 검증 구간의 기간](/assets/img/posts/binance-backtest/periods-dark.png){: .dark }
_구간 B의 정확한 시작일은 캐시를 처음 만든 시각으로 정해져 저장소에 남아 있지 않다. 그림은 월 단위다_

구간 A는 `2025-03-01T00:00:00Z`부터 1년으로 코드에 고정돼 있다. 구간 B는 `backtest_strategy_lab.py`가 처음 실행될 때 `datetime.now(timezone.utc)`에서 365일을 빼 받아 `xrp_15m_1year_fresh.csv`로 캐시한 데이터다. 라벨은 "2025-08~2026-08"이다.

두 구간은 2025-08부터 2026-03까지 약 7개월이 겹친다. 1년 중 절반 넘게 같은 캔들이다. 그래서 "두 구간에서 다 이겼다"는 독립된 두 번의 확인이 아니다. 거기에 그리드는 두 구간을 **모두 보고** 조합을 골랐으니 어느 쪽도 학습에 안 쓴 홀드아웃이 아니다. 최악 구간 기준은 한쪽 구간에만 맞춘 조합을 걸러주지만, 둘이 공유하는 7개월에 맞춘 조합은 못 거른다.

겹치지 않는 구간으로 나누거나, 앞 구간에서 고르고 뒤 구간에서 한 번만 확인하는 워크포워드 방식이 이 문제를 직접 다룬다. 저장소에는 그 구현이 없다.

## 6. 후보 비교 결과

14개 후보는 다음과 같다.

| 계열 | 후보 |
|---|---|
| ADX + EMA 계열 | A. ADX25+EMA+RSI, A2. ADX30+EMA+RSI, H. 1시간 EMA50 필터 + ADX |
| 이동평균 / 채널 | B. 트리플 EMA(5/13/34) 정렬, C. Keltner 돌파, G. OBV + Donchian |
| 추세 전환 | D. 일목균형표, E. PSAR + ADX, I. Aroon 교차, K. Vortex 교차 |
| 모멘텀 | F. RSI2 눌림목, J. CCI ±100 돌파, L. TRIX 교차, M. STC |

결과 표는 저장소에 없다. 작업 로그(2026-08-04)의 결론은 "신규 전략은 대부분 기존 ADX(>30)+EMA+RSI 전략보다 못했고, 기존 전략이 두 구간 모두에서 가장 견고했음"이다. 같은 신호에서 SL을 1.5%에서 2.0%로 바꾸는 것만으로 수익률이 크게 좋아졌다는 기록도 있다. 둘 다 수치 없이 문장으로만 남아 있다.

레버리지 결정(2026-08-21)도 로그 문장으로만 남아 있다. 로그상 최근 구간 기대수익은 10x가 더 높고 낙폭은 15x가 더 컸으나, "역대급 한방" 지향으로 15x를 채택했다. 그리드 스크립트가 커밋되지 않아 재현할 수 없다. `get_signal()` 독스트링에도 "최신구간 기대치는 10x보다 낮음"이 그대로 남아 있다.

저장소의 lab3 그리드는 고정 TP만 쓰고 레버리지 후보가 5/10/20x라 15x가 없다. 트레일링 비교 스크립트들은 구간 A 하나에 이전 전략(ADX > 25)과 +2%/1% 파라미터를 쓴다. 최종 설정인 "ADX>30 + 트레일링 +1.5%/0.5% + 15x"를 두 구간에 돌린 코드는 저장소에서 확인되지 않는다. 수수료는 모든 백테스트 스크립트가 `FEE = 0.0004`라 같은 값이었을 가능성이 높지만, 코드로 확인한 건 아니다.

## 7. 백테스트와 실거래 코드가 다른 지점

같은 전략을 두 번 구현했다. 백테스트는 pandas 벡터 연산, 라이브는 `get_signal()`의 스칼라 비교다. 둘을 나란히 놓으면 차이가 보인다.

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

바이낸스 klines API는 아직 닫히지 않은 현재 봉을 마지막 원소로 돌려준다. 라이브 봇은 `df.iloc[-1]`을 `curr`로 쓰니 진행 중인 봉의 현재가로 EMA, RSI, ADX를 계산한다. 5분마다 돌기 때문에 한 15분봉을 세 번, 각각 다른 "종가"로 판단한다. 백테스트는 마감된 종가로 한 번만 판단한다. 라이브가 봉 중간에 진입했다가 그 봉이 마감될 때는 조건이 풀려 있을 수 있다.

### 첫 신호 판정이 다르다

백테스트의 "첫 신호"는 전체 신호 시리즈를 한 칸 민다.

```python
def first_signal_only(sig):
    return (sig & ~sig.shift(1).fillna(False)).values
```

`sig`에는 ADX > 30과 RSI > 50이 들어 있다. 반면 라이브의 `prev_long`은 EMA 두 조건만 본다. EMA는 이미 정렬돼 있고 ADX가 이번 봉에 30을 넘은 경우, 백테스트는 직전 봉 신호가 거짓이라 진입하고 라이브는 직전 봉 EMA가 이미 참이라 진입하지 않는다. RSI가 50을 다시 넘는 경우도 같다. 라이브의 조건이 더 엄격하니 라이브 신호는 백테스트 신호의 부분집합이다.

얼마나 차이 나는지 구간 A(2025-03-01~2026-03-01) XRP/USDT 15분봉을 바이낸스 선물 API에서 다시 받아, 두 규칙의 진입 신호 봉 수(포지션 보유 중 무시되는 신호 포함)를 셌다. 저장소 밖에서 재계산했다. 지표 4개(EMA9, EMA20, RSI14, ADX14), 마감봉 기준, 진행 중 봉과 200봉 웜업은 미반영.

| | LONG | SHORT | 합계 |
|---|---|---|---|
| 백테스트 규칙 | 633 | 753 | 1,386 |
| 라이브 규칙 | 435 | 505 | 940 |

라이브 규칙이 내는 신호는 전부 백테스트 신호 안에 있었고, 백테스트 신호 1,386개 중 446개(32%)는 라이브 봇이라면 진입하지 않았을 봉이다.

![XRP/USDT 15분봉 진입 신호](/assets/img/posts/binance-backtest/signals-light.png){: .light }
![XRP/USDT 15분봉 진입 신호](/assets/img/posts/binance-backtest/signals-dark.png){: .dark }
_저장소 외 재계산, 마감봉 기준. 2026-02-27~02-28. 채운 삼각형은 두 규칙 모두, 빈 삼각형은 백테스트 규칙만 진입하는 봉. 이 이틀에 각각 7개, 3개_

빈 삼각형 세 개(02-27 12:00, 02-28 05:30, 02-28 19:30)는 모두 ADX가 직전 봉 29점대에서 이번 봉에 30을 넘은 봉이다. EMA 정렬은 그 전부터 이어지고 있었다. 백테스트가 진입할 수 있는 봉의 3분의 1가량을 라이브 봇은 진입 후보로 보지 않는다. 실제 거래 수 차이는 포지션 보유 중 무시되는 신호 때문에 이보다 작을 수 있고, 그 신호들이 수익 쪽이었는지 손실 쪽이었는지는 재보지 않았다.

이 그림에는 하나 더 보인다. 02-27 19:00부터 23:00까지 ADX가 40대에 머문 채 가격이 횡보하자 SHORT 신호가 네 번 났고, 자정에는 LONG이 났다. EMA9가 평평해지면 "EMA9 하락 중" 조건이 한 봉 꺼졌다 다시 켜지고, 그때마다 "첫 신호"가 새로 생긴다. 포지션 보유 중에는 무시되지만, 손절 직후라면 같은 횡보 안에서 다시 진입한다.

## 8. 실거래였나, 모의였나

| 근거 | 내용 |
|---|---|
| 코드 | `ccxt.binance`에 `defaultType: 'future'`, 테스트넷이나 샌드박스 설정 없음. API 키가 없을 때만 Dry-run |
| `CLAUDE.md` | "`trading_bot.py`와 `strategy_analyzer.py`는 실제 바이낸스 선물 계좌에 연결되어 있다" |
| `work_log.md` 2026-08-21 | "운영 중 발견한 버그" 항목 |
| `backtest_jackpot.py` | 실험 1의 제목이 "현재 라이브 봇 설정 정확 재현": 75x, 손절 없음, TP 2.5%, 비중 85% |

실계좌로 돌렸다는 근거는 있다. 실거래 성과는 없다. 매매 기록 `trade_history.log`가 `.gitignore`에 들어 있어서 진입·청산 내역도, 손익도 저장소에 남지 않았다. 작업 로그에는 8월 4일 레버리지를 75x에서 10x로 내렸다고 적혀 있는데, 75x 시절의 결과도 기록이 없다.

운영 버그 하나는 코드에 남아 있다. 사용자가 거래소 앱에서 포지션을 직접 닫으면, 봇은 그 사실을 모른 채 30초마다 청산 주문을 다시 내고, 거래소는 `ReduceOnly Order is rejected`(-2022)로 거절하고, 그때마다 텔레그램 알림이 갔다. `close_position()`에서 이 에러를 "거래소에 포지션이 없다"는 신호로 해석하게 고쳤다.

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

재시작 쪽은 `sync_position()`이 맡는다. 시작할 때 `fetch_positions()`의 `positionAmt` 부호로 방향을 복원하고, 재시작 전의 고점은 알 수 없으니 트레일링은 진입가부터 다시 무장한다.

```python
# 재시작 전 가격 흐름은 알 수 없으므로 트레일링은 진입가부터 보수적으로 재무장
self.peak_price = self.entry_price
self.trail_stop = 0
```

두 경로 모두 봇의 메모리 상태와 거래소 상태가 어긋나는 경우다. 하나는 시작 시점에 맞추고, 하나는 에러가 난 뒤에 맞춘다. 주기적으로 거래소 포지션과 대조하는 코드는 없다.

## 9. 한계

- **결과가 재현 가능한 형태로 남지 않았다.** 캔들 CSV와 백테스트 출력이 커밋되지 않았고, 최종 설정을 고른 그리드 스크립트도 없다. 수치는 작업 로그 문장뿐이다.
- **검증 구간이 7개월 겹치고 홀드아웃이 없다.** 최악 구간 기준은 한쪽에 맞춘 조합만 거른다.
- **최종 레버리지는 검증 기준과 다르게 골랐다.** 로그상 최근 구간 기대수익은 10x가 더 높았고, 낙폭은 15x가 더 컸다(구간 미기재). 근거 그리드는 재현할 수 없다.
- **트레일링 결과가 봉 내부 경로 가정에 기대고 있다.** README가 0.3~0.7% 추적폭에서 결과가 민감하고, 실제 슬리피지에서는 백테스트보다 보수적인 성과를 예상해야 한다고 적어 둔 부분이다.
- **비용 모델이 수수료뿐이다.** 슬리피지, 펀딩비가 없다.
- **백테스트와 라이브의 신호 규칙이 다르다.** 첫 신호 판정 범위와 진행 중인 봉 사용. 재계산 기준 약 32%의 백테스트 신호가 라이브 규칙에서는 나오지 않는다.
- **실거래 성과 기록이 없다.**

코드: [github.com/kasangyong/binance](https://github.com/kasangyong/binance)
