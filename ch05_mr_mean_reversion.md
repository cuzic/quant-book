# Chapter 5: MR — Mean Reversion (CRSI + Profit Target)

## 5.1 学術的根拠

### 短期リバーサルの anomaly

Jegadeesh (1990) と Lehmann (1990) が独立に発見した短期リバーサル (1-5 日) は、behavioral finance の overreaction 仮説と整合する: 投資家は短期的にニュースに過剰反応し、その後 mean revert する。

Connors RSI (Connors & Alvarez 2009) はこの anomaly を定量化した composite indicator:
- RSI(3): 3 日間の相対強度
- Streak length: 連続上昇/下落日数
- Percentile rank: 過去の return 分布における位置

### Profit Target による exit

伝統的な MR 戦略は「N 日後に exit」のような固定期間。Profit Target は「目標利益到達時に即 exit」で:
- 勝ちトレードの duration を短縮 → capital efficiency 向上
- 過度な mean reversion (行き過ぎ) を回避
- `sig_profit_target()` で numba 高速実装

## 5.2 実装詳細

```python
# portfolio.py: _build_mr()
from eval_final_v7 import build_mr_v7
mr = build_mr_v7(self.pre, self.orats)
```

`build_mr_v7` の内部:
1. 3746 銘柄の preprocessed universe (`preprocessed_universe.pkl`)
2. CRSI signal 計算 (RSI3 + streak + percentile)
3. ORATS earnings filter: 決算 2 日前〜発表日を除外 (event risk 回避)
4. Top 50 銘柄を long、profit target で dynamic exit
5. Vol target 10%

### Earnings Filter の重要性

MR 戦略は「下がりすぎた銘柄を買う」。決算前の下落は「event risk」であり mean reversion ではない。ORATS の earnings data (`impliedEarningsMove`) で filter することで、false signal を除去。

**注**: IV_PREM (Chapter 11) では earnings filter が alpha を**減少**させた (event risk premium こそが VRP alpha)。MR と IV_PREM は earnings に対して**逆の反応**を持つ — diversification の源泉。

## 5.3 パフォーマンス

### Standalone
```
Test (2019-2026) Sharpe: +0.94
Sub2 (2022-2026) Sharpe: ~+0.9
```

### Portfolio 内での役割
- **IV weight: ~19% (最大 component)**
  MR は低 vol (安定した small daily gains) → inverse-vol weighting で最大配分
- Overlay: default path (J3 × VIX × ORATS × Asymm 全適用)
- Core component group (core_extra = IVP<1 で 0.7x)

### Ablation (MR 除去時の portfolio 影響)
```
Drop MR: Sub2 Sharpe -0.053 (小幅悪化、weight 最大だが alpha は modest)
```

## 5.4 試行錯誤経緯

### 開発初期の DD brake バグ
MR component に DD brake (DD < -1.5% で scale 0.7) を適用していたが、**same-day look-ahead** で Sharpe が 30x に膨張していた。修正後の standalone Sharpe は ~0.9 (modest)。

### ML sizing の試み (失敗)
ML (Random Forest / XGBoost) で MR signal の strength を予測し、position size を adaptive にする試みは失敗 (eval_ml_sizing.py)。cross-sectional ranking + equal weight が最良。

### ORATS data の追加価値
Earnings filter 以外に、IV percentile で「vol regime」を判定し、低 vol 期に MR signal を boost する variant も試みたが、marginal improvement のため不採用。

## 5.5 他コンポーネントとの相関

| Component | Correlation with MR |
|---|---|
| GTAA | -0.02 (ほぼ独立) |
| JP_LS | +0.03 |
| IV_PREM | -0.05 (逆相関 = 分散に貢献) |
| VC | +0.08 |
| CTA | -0.01 |

MR は他 components とほぼ無相関 → **portfolio の基盤** としての役割。

## 5.6 Edge の持続可能性

短期リバーサルは 1990 年代から知られる anomaly だが:
- Execution cost が barrier (high turnover)
- 機関投資家の参入が限定的 (small-cap が主戦場)
- Behavioral origin (overreaction) は消えにくい

**リスク**: HFT の普及で short-term anomaly の alpha が erosion する可能性。本ポートフォリオでは MR の weight 19% は自動調整 (alpha 低下 → vol 上昇 → iv-weight 低下)。

---

*次章: GTAA — Global Tactical Asset Allocation*
