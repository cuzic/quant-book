# 付録: 全バックテスト結果データ

## A. Per-year Sharpe (v35)

```
Year    Sharpe  Sortino  Return%   MaxDD%
2019    +6.66    +9.3    +20.9%   -0.60%
2020    +8.11   +10.3    +16.9%   -0.75%
2021    +6.37    +8.1    +14.8%   -0.82%
2022    +5.78    +6.5    +13.3%   -1.00%
2023    +7.53    +9.9    +14.5%   -0.70%
2024    +6.55    +8.5    +12.1%   -0.69%
2025    +7.99   +10.6    +14.7%   -0.88%
2026    +7.73   +10.0    +12.5%   -0.41%
```

## B. Monthly Returns (v34, 88 months)

88 ヶ月中 86 positive (98%), 2 negative (2023-05: -0.16%, 2021-07: -0.04%)

## C. Ablation Table (17 components × overlay LOO)

| Drop | Test Δ | Sub2 Δ | Verdict |
|---|---|---|---|
| MR | -0.053 | -0.053 | keep |
| GTAA | -0.255 | -0.255 | keep (strong) |
| LV | -0.460 | -0.460 | keep (strongest) |
| VC | +0.218 | -0.074 | keep (Sub2 neutral) |
| MV | -0.105 | -0.105 | keep |
| IM | -0.383 | -0.383 | keep |
| R5 | -0.077 | -0.077 | keep |
| R2 | +0.030 | +0.030 | marginal |
| R1 | +0.061 | +0.061 | marginal |
| R6 | -0.037 | -0.037 | keep |
| CTA | -0.081 | -0.081 | keep |
| XLE_T | +0.018 | +0.018 | neutral |
| MN | +0.083 | +0.083 | marginal drag |
| T4b | -0.041 | -0.041 | keep |
| JP_LS | -0.408 | -0.408 | keep (2nd strongest) |
| AU_LS | -0.133 | -0.133 | keep |

## D. Weighting Scheme Comparison (10 schemes, Sub2)

| Scheme | Sub2 | vs baseline |
|---|---|---|
| Inverse-Volatility 504d | +2.99 | baseline |
| HRP | +2.84 | -0.15 |
| Inv-Correlation | +2.64 | -0.35 |
| 1/N Equal | +2.53 | -0.46 |
| Rank Top 12 | +2.32 | -0.67 |
| sqrt(Sharpe) | +2.20 | -0.79 |
| Rank Top 10 | +2.34 | -0.65 |
| Sharpe^1 | +2.12 | -0.87 |
| Rank Top 8 | +2.11 | -0.88 |
| Sharpe^2 | +1.84 | -1.15 |

## E. IV Premium Grid Search (selected results)

### Signal × TopN × Hold (72 combos, top both-positive)
| Config | Sub2 | 2023-05 | 2021-07 |
|---|---|---|---|
| vrp+skew top10 hold5 | +1.661 | +6.26% | +0.13% |
| vrp top5 hold5 | +1.138 | +7.15% | +0.63% |
| vrp top5 hold21 | +1.111 | +5.65% | +1.09% |

### Full grid (36 combos, signal × side × sizing × filter)
Champion: v+s long eq ORATS = +2.791

### 17 Combo tests
Champion: Long VRP-prop ORATS DD-speed = +3.522

## F. T1-T7 Look-Ahead Audit

| Test | Δ Sub2 | Verdict |
|---|---|---|
| T1-L1 ORATS +1 shift | -0.611 | freshness (not look-ahead) |
| T1-L2 J3 +1 shift | -0.005 | clean |
| T1-L3 VIX +1 shift | -0.049 | clean |
| T1-L4 Asymm +1 shift | -0.034 | clean |
| T1-L5 inv-vol shift | -0.002 | clean |
| T6 Shuffle | +2.638 (42%) | overlay = independent alpha |
| T6 Shift +5d | +4.174 (66%) | autocorrelation + overlay |
| T6 Reverse | -0.117 (0%) | no mechanical bug |
| T7 PnL averaging | **BUG FOUND** | revert (v34 hold overlap) |

## G. Statistical Validation

| Metric | Value | Threshold | Verdict |
|---|---|---|---|
| DSR z (N=30000) | +10.56 | >1.96 | PASS |
| PBO | 18.6% | <20% | PASS |
| Bootstrap 21d P(new>old) | 98% | >95% | PASS |
| Bootstrap 63d | 98% | >95% | PASS |
| Bootstrap 126d | 98% | >95% | PASS |

## H. Cost Stress Sensitivity

Break-even: 27.5 bp/side
Realistic (6bp): Live Sharpe ~4.1
Conservative (10bp): Live Sharpe ~3.2

## I. Negative Month Analysis

### 2023-05 (-0.158%)
- VIX: 16-20 (normal), SPY: +0.46%
- Drag: GTAA -0.159%, XLE_T -0.113%, CTA -0.106%, LV -0.104%
- Rescue: JP_LS +0.294%

### 2021-07 (-0.043%)
- VIX: 15-22.5, SPY: +2.44%
- Drag: MN -0.223%, T4b -0.137%, MR -0.128%, R1 -0.116%
- Rescue: R6 +0.160%

## J. Diversification Ratio by VIX Regime

| VIX | Days | Avg DR | Min DR | Avg Vol |
|---|---|---|---|---|
| <15 | 415 | 2.83 | 0.98 | 2.54% |
| 15-20 | 695 | 3.36 | 0.70 | 2.18% |
| 20-25 | 369 | 3.70 | 2.27 | 2.14% |
| 25-35 | 290 | 4.03 | 2.33 | 2.18% |
| >=35 | 59 | **5.06** | 3.57 | 2.43% |

**Crisis 時に DR が最大** → correlation collapse リスクは低い。
