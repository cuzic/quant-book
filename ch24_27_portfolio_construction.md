# Part IV: ポートフォリオ構築と検証

# Chapter 24: Inverse-Volatility Weighting の設計と検証

## 24.1 10 Weighting Scheme の比較実験

本セッションで 10 の weighting scheme を v33 components で直接比較。overlay なしの raw component sum で評価:

| Scheme | Sub2 Sharpe | 理論基盤 |
|---|---|---|
| **Inverse-Volatility (504d)** | **+2.99** 🥇 | Risk parity variant |
| HRP (López de Prado) | +2.84 | Hierarchical clustering |
| Inv-Correlation | +2.64 | Diversification maximization |
| 1/N Equal Weight | +2.53 | DeMiguel et al. (2009) |
| Rank Top 12 (6M) | +2.32 | Dynamic selection |
| Rank Top 10 (6M) | +2.34 | Dynamic selection |
| sqrt(Sharpe) | +2.20 | Performance chasing |
| Rank Top 8 (6M) | +2.11 | Concentrated selection |
| Sharpe^1 | +2.12 | Linear performance |
| Sharpe^2 | +1.84 | Quadratic performance |

**Inverse-volatility が全 scheme に勝利。** DeMiguel (2009) の「1/N が最適化に勝つ」は本 portfolio では不成立 (Chapter 2 参照)。

### Paired t-test の罠

| Scheme | ΔSharpe | t-stat | Ann Δret |
|---|---|---|---|
| 1/N | -0.46 | **+2.61** | **+2.35%** |
| Rank Top 10 | -0.65 | +2.67 | +4.48% |

**ΔSharpe はマイナスだが t-stat は正 = 「return は増えるが vol も増える」**。Sharpe (risk-adjusted) で判断すべき。

## 24.2 Rolling 504d Window の根拠

- 504 日 = 2 年 = 景気サイクル 1 周期の approximately half
- 短すぎる (252d): 推定 noisy → weight が不安定
- 長すぎる (1008d): regime change に遅い反応
- T2 テスト: inv-vol weight を shift(1) → Sub2 **Δ -0.002** (完全 robust)

## 24.3 AU_LS の warmup 問題

AU_LS は 2020-01 開始 (UTF-16 修正後)。2020 以前は return = 0 → vol = 0 → inv-vol weight = ∞ → fallback to equal weight。

504 日 window のため、AU_LS が正常に weighted されるのは **2022-01 以降**。これが Sub1 (2019-21) vs Sub2 (2022+) で AU_LS の寄与が異なる理由。

---

# Chapter 25: Per-Component Overlay Subset 最適化 (v32)

## 25.1 発見の経緯

ユーザーの仮説: 「US overlay を AU/UK にも適用したら?」

実験: 全 component × 8 overlay pattern の grid search (17 × 8 = 136 combos)。

結果: **component によって最適 overlay が大きく異なる**:

| Component Group | Best Overlay | Rationale |
|---|---|---|
| Core 6 (MR,GTAA,LV,VC,MV,IM) | **Full** (J3×VIX×ORATS×Asymm) | US equity strategies |
| R1 (TLT), R2 (TIP-TLT), MN | **NO_OVERLAY** | Rates/sector → option overlay は noise |
| AU_LS, XLE_T | **VIX_ONLY** | Foreign/commodity → ORATS は noise |
| R5 (UUP/USD) | **J3+VIX** | Currency → no ORATS |
| IV_PREM | **NO_OVERLAY** | Internal ORATS timing (redundancy 防止) |

### Insight

「US ORATS option overlay は US equity strategies でのみ effective。Non-equity (rates, commodity, currency, foreign) には noise として作用する。」

## 25.2 Walk-Forward Retention

各年の overlay choice を prior data のみで決定:
```
WF retention: 97% (overlay choices が時間安定)
Year-over-year change: 0-2 components per year
```

---

# Chapter 26: Statistical Validation (DSR, PBO, Bootstrap, T1-T7)

## 26.1 Deflated Sharpe Ratio (DSR)

| N_trials | SR_expected | v35 z-score | 判定 |
|---|---|---|---|
| 100 | 1.43 | +13.05 | 有意 |
| 680 | 1.78 | +12.09 | 有意 |
| 5,000 | 2.08 | +11.24 | 有意 |
| **30,000** | **2.33** | **+10.56** | **有意** |

**30,000 回の trial を仮定しても z > 10** — Sharpe は偶然の産物ではない。

## 26.2 PBO (Probability of Backtest Overfitting)

CSCV with 8 partitions, 100 combinations:
```
PBO = 18.6% (< 20% = Low overfit risk)
```

## 26.3 Bootstrap CI

v33 Sub2 (21/63/126 day blocks):
```
block= 21d: P(v33>v32) = 98%
block= 63d: P(v33>v32) = 98%
block=126d: P(v33>v32) = 98%
```

全 horizon で一貫 — 長期相関構造を考慮しても robust。

## 26.4 T1-T7 Look-Ahead Audit

| Test | 発見 | 重要度 |
|---|---|---|
| T1 (shift +1) | 4/5 layer clean, ORATS Δ-0.61 (freshness) | ★★★ |
| T2 (inv-vol shift) | Sub2 Δ-0.002 (clean) | ★★ |
| T3 (sqrt(S)/IC fixed) | Low risk (small weight) | ★ |
| T4 (hybrid shift) | T1 で covered | — |
| T5 (top30 timing) | T1+T7 で covered | — |
| T6 (random destruction) | **Overlay = Sharpe 42%** 発見 | ★★★★ |
| **T7 (hold overlap)** | **PnL averaging バグ発見** | ★★★★★ |

## 26.5 T6 の構造的発見

Component returns を shuffle しても Sub2 +2.64 (42%) が残存:
```
Portfolio Sharpe 6.87 = Overlay timing alpha (~2.9) + Component alpha (~3.9)
```

**Overlay は filter ではなく独立 alpha source。** ORATS subscription ($99/月) の ROI は極めて高い。

---

# Chapter 27: Cost Modeling と Live Sharpe 推定

## 27.1 Turnover 推定

実測 portfolio turnover: **10.62%/日、26.8x/年**

主な contributor:
```
T4b:   2.19% (wavelet, 50% assumed → 実測 10.8%)
R5:    1.84% (bsig 5d rebalance)
MN:    1.37% (daily sector rotation)
R6:    0.94%
MR:    0.93%
```

**注**: T4b の想定 50% は実測 10.8% と乖離 — cost stress test の精度に影響。

## 27.2 Cost Stress

| Cost (bp/side) | Annual Drag | Test Sharpe | Live 推定 |
|---|---|---|---|
| 3 (US-only) | 1.07% | +5.47 | +4.77 |
| **6 (mixed)** | **2.68%** | **+4.80** | **+4.10** |
| 10 (all CFD) | 5.35% | +3.91 | +3.21 |
| 15 (illiquid JP) | 8.03% | +2.79 | +2.09 |

**Break-even: 27.5 bp/side。** IB CFD Japan の typical cost (6-10bp) では十分な margin。

## 27.3 Live Sharpe 推定

```
Backtest Sharpe: +6.87 (v35)
Cost drag:       -0.8 to -1.5 (6-10bp assumption)
Overfit decay:   -0.4 (conservative)
Regime decay:    -0.3/yr

Live estimate:   4.0 - 5.0 (realistic range)
Central:         4.2 ± 1.0
```

---

*Part V 開始: Chapter 28 — safe_backtest.py*
