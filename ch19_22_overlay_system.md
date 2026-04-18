# Part III: Overlay System

# Chapter 19: J3 — JP-US Sentiment Divergence

## 19.1 設計

日本 (J-Quants 信用倍率) と米国 (ORATS VRP) のセンチメント divergence:
```
JP euphoric (jp_q > 0.80) AND US fearful (us_q > 0.70) → 0.8x (reduce)
JP pessimistic (jp_q < 0.30) AND US fearful (us_q > 0.70) → 1.2x (increase)
```

**根拠**: 日米で sentiment が diverge する時、一方の「過剰反応」を利用。

## 19.2 検証結果

Ablation: Drop J3 → Sub2 +0.003 (**neutral**)。J3 は portfolio にほぼ影響なし。

T1 shift test: J3 +1 shift → Δ -0.005 (robust)。Timing 問題なし。

**Honest assessment**: J3 は v12 で導入されたが、v32 以降の per-component overlay 最適化で**事実上 no-op**。但し R5 (USD) の J3_VIX overlay path で機能するため維持。

---

# Chapter 20: VIX Level Overlay

## 20.1 設計

```python
VIX < 17 → 1.2x (calm market, increase)
VIX > 25 → 0.8x (stress, reduce)
```

閾値 17/25 は VIX の歴史的分布 (25th/75th percentile 近傍) に基づく market convention。

## 20.2 「固定閾値 > Adaptive Percentile」の実証

A5 テストで 5 variant を比較:
```
Baseline (17/25 fixed):     Sub2 +6.291 (best!)
No VIX overlay:             Sub2 +6.203 (-0.089)
Percentile tanh 504d:       Sub2 +6.059 (-0.232)
Percentile tanh 252d:       Sub2 +6.053 (-0.239)
Z-score tanh 252d:          Sub2 +6.100 (-0.191)
```

**全ての adaptive 版が固定閾値より悪い** (全 variant p < 0.001 で有意に劣る)。

### なぜ固定閾値が勝つのか

VIX の 17/25 は「market convention」であり、多くの参加者がこの水準を意識する。Adaptive percentile は個別 portfolio の歴史に依存するため、market の collective behavior と sync しない。

**一般原則**: Overlay の閾値は **market convention に寄せる** のが robust (ChatGPT 推奨、実データで確認)。

---

# Chapter 21: ORATS EMA Overlay (PC × TS × IVP)

## 21.1 3 Signal の設計

| Signal | Data | EMA span | 判定 |
|---|---|---|---|
| PC | Put/Call volume ratio (inverse) | 14d | Low P/C (楽観) → 1.2x |
| TS | IV term structure M1/M4 (inverse) | 42d | Contango (calm) → 1.2x |
| IVP | IV percentile 1y (inverse) | 4d | Low IVP (calm) → 1.2x |

Combined: PC × TS × IVP (multiplicative chain)

### EMA Speed 最適化 (v20)

v19 SMA → v20 EMA + speed optimization:
- PC: 21d → **14d** (短期 sentiment → faster response)
- TS: 42d → **42d** (structural → unchanged)
- IVP: 21d → **4d** (daily regime indicator → much faster)

Validated: plateau at IVP=2-5, PC=5-14, TS=30-84 (no overfitting)。

## 21.2 「Binary > Continuous」の実証

**S1 テスト**: EMA binary (1.2/0.8) → continuous tanh を全面テスト。

```
v33 baseline (binary):    Sub2 +6.291
S1 EMA continuous:        Sub2 +5.680 (Δ -0.611, t=-6.58!)
```

**continuous 化は Sub2 で -0.611 — 有意に悪化。**

### なぜ binary が勝つのか

Binary 1.2/0.8 は **regime detection signal**: ratio > EMA = bullish regime、< EMA = bearish regime。これは continuous な proportional scaling ではなく、**categorical classification** (bull/bear) としての value がある。

Continuous tanh は「曖昧な中間値」を生む → regime の decisiveness が失われる → alpha 低下。

**本書の一般原則 (Principle 2)**: Binary regime detection は continuous 化より優れる。

## 21.3 Hybrid Shift (v18)

Overnight portion: shift=2 (conservative, 2 日前のデータ)
Intraday portion: shift=1 (fresher, 前日のデータ)

SPY の Open/Close から overnight/intraday fraction を推定:
```python
overnight_frac = |open[t] / close[t-1] - 1| / total_daily_move
intraday_frac = 1 - overnight_frac
```

T1 テスト: ORATS overlay +1 shift → Sub2 **-0.611** (最大の感度)。ORATS signal の freshness が Sharpe の ~10% を占める。

### ORATS Data Timing Validation

ORATS Delayed API: 15 分遅延。Market close 16:00 ET → data available ~16:30 ET → 翌日 open 9:30 ET = **17 時間の buffer**。Shift=1 は data availability 的に valid。

## 21.4 Portfolio への寄与

Ablation: Drop ORATS → Sub2 **-1.304** (**最大の alpha source**)。

T6 shuffle test: Component returns を shuffle しても Sub2 +2.638 (42%) が残存 → **ORATS overlay 単独で Sharpe ~2.6**。

ORATS overlay は「filter」ではなく「independent alpha source」。

---

# Chapter 22: Asymmetric Agreement (IC-Weighted 5 Z-score)

## 22.1 設計

5 つの ORATS z-score の合意度で sizing:
- z_pc, z_ts, z_ivp (Chapter 21 と同じ data の z-score 版)
- z_curvature: IV term structure の 2 次曲率 (-polyfit coeff)
- z_carry: IV30d - HV20d (variance risk premium)

### IC-Weighted Sum (v26)

Training period (2014-2018) で各 z-score の IC (Information Coefficient) を推定:
```
z_pc:   IC = +0.0146 → weight positive
z_ts:   IC = -0.0079 → weight negative (sign flip!)
z_ivp:  IC = +0.0161 → weight positive
z_curv: IC = -0.0121 → weight negative (sign flip!)
z_carry: IC = -0.0100 → weight negative (sign flip!)
```

**Sign flip**: TS, curvature, carry は negative IC → z-score を反転して合算。

### 5-Level Step Function

```python
if agree_score > 3:  scale = 1.3  (strong bullish → +30%)
elif agree_score > 1: scale = 1.1  (+10%)
elif agree_score < -3: scale = 0.5  (strong bearish → -50%)
elif agree_score < -1: scale = 0.9  (-10%)
else: scale = 1.0
```

**非対称設計**: 上 (+30%) は控えめ、下 (-50%) は大胆 = defensive bias。

## 22.2 S2 テスト: Smooth tanh 化の試みと失敗

S2 テスト: Step function → smooth tanh。

Eval script: Sub2 +0.045 (t=+4.16, positive)。
**しかし production 実装: Sub2 -0.224 (negative)**。

原因: eval script が `agree_raw` を独立計算 (ORATS ticker 全体) するのに対し、`_build_asymmetric_agreement()` は iv30d quality filter (>500 日) を適用 → **分布が異なる**。

**教訓**: Eval script と production code の pipeline 差異に注意。必ず production 関数を直接呼び出して検証すべき。

## 22.3 Ablation 結果

```
Drop Asymm: Sub2 -0.013 (neutral)
agree_score × 4.0 scaling: Sub2 -0.006 (neutral)
```

Asymm は portfolio にほぼ影響しない。5 z-score の complex logic が α source ではなく、他の overlay (ORATS binary, VIX) に redundant。

**但し**: Asymm は「保険」として機能 — strong bearish (-50%) で extreme loss を防止。Normal 時の α 寄与はゼロだが、tail event で発動。

---

*次章: DD Control — Depth + Speed Sensing*
