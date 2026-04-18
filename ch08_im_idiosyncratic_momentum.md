# Chapter 8: IM — Idiosyncratic Momentum

## 8.1 学術的根拠

Blitz, Huij & Martens (2011) "Residual Momentum": 株式の total return から market β を除去した residual return の momentum は、total return momentum よりも strong かつ robust。

**なぜ residual momentum が優れるか**:
- Total momentum は market factor に汚染される (bull market で全株上昇 → ranking が noisy)
- Residual momentum は「市場要因を除いた後の、その銘柄固有の強さ」を捕捉
- Fama-French 3-factor residual でさらに改善するが、本戦略では SPY β 除去のみ (simplicity)

## 8.2 実装

```python
# portfolio.py: _build_im()
from eval_final_v5 import build_idio_momentum
im = build_idio_momentum(self.pre, self.spy_on_stocks)
```

手順:
1. 各銘柄の return から SPY return × β を減算 → residual return
2. Residual return の 252 日 momentum を計算
3. Top 100 銘柄を long
4. Vol target 10%

## 8.3 パフォーマンス

```
Standalone Test Sharpe: +1.58
IV weight: ~4.4%
Ablation: Drop IM → Sub2 -0.383 (significant contribution)
```

IM は MR に次ぐ high-alpha component。SPY β 除去により market-neutral 寄りの alpha を提供。

## 8.4 MR との対比

| | MR (Mean Reversion) | IM (Idio Momentum) |
|---|---|---|
| Horizon | 1-5 days | 12 months |
| Direction | Contrarian | Trend-following |
| β exposure | Market-neutral (L/S) | Long-only (residual) |
| Correlation | — | -0.05 with MR |

MR と IM は **逆方向** の signal (reversal vs momentum) → **分散に大きく寄与**。

---

# Chapter 9: MV — Minimum Variance Portfolio

## 9.1 学術的根拠

Clarke, de Silva & Thorley (2006): Minimum Variance portfolio は market-cap weighted portfolio を risk-adjusted basis で outperform。Jagannathan & Ma (2003) は「間違った制約を課すことが helped」という paradox を示した — short-sale constraint が estimation error を natural に shrink。

## 9.2 実装

```python
# portfolio.py: _build_mv()
from eval_final_v3 import build_min_variance
mv = build_min_variance(self.etf_closes, self.etf_dates)
```

87 ETF の rolling 共分散行列から minimum variance portfolio を構築。Ledoit-Wolf shrinkage は不使用 (共分散の direct 推定で十分な universe size)。

## 9.3 パフォーマンス

```
Standalone Test Sharpe: +0.89
IV weight: ~14.4% (2 番目に大きい component)
Ablation: Drop MV → Sub2 -0.105
```

MV は standalone alpha は modest だが、**低 vol** のため iv-weighting で大きな配分 (14.4%) を受ける。Portfolio の「安定基盤」として機能。

---

*次章: VC — VIX Contango と crisis alpha の真実*
