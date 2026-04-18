# Chapter 6: GTAA — Global Tactical Asset Allocation

## 6.1 学術的根拠

### Faber (2007) の原型

Mebane Faber の "A Quantitative Approach to Tactical Asset Allocation" は、5 資産クラス (US stocks, foreign stocks, bonds, REITs, commodities) に対して 10 ヶ月移動平均を用いた timing strategy を提案した。SMA(10M) 超なら保有、未満なら現金。1973-2006 で Sharpe 0.56 → 0.73 に改善、MaxDD を -45% → -13% に大幅縮小。

### Cross-sectional Momentum の ETF 適用

Jegadeesh & Titman (1993) の個別株 momentum (12-1 month) を ETF universe に適用すると、asset class 間の relative strength を捕捉できる。Asness, Moskowitz & Pedersen (2013) "Value and Momentum Everywhere" は、momentum が equity, bond, currency, commodity の全 asset class で機能することを実証。

本戦略は Faber の timing と Jegadeesh-Titman の cross-sectional ranking を融合:
- **87 ETF** の broad universe (Faber の 5 asset class を大幅拡張)
- **189 日 lookback** (学術的 12-1 month ≈ 252-21 に近い)
- **Top 15** を equal-weight 保有

## 6.2 実装詳細

```python
# portfolio.py: _build_gtaa()
from eval_final_v3 import build_gtaa_v2
gtaa = build_gtaa_v2(self.etf_closes, self.etf_dates)
```

`build_gtaa_v2` の内部:
1. 87 ETF の Close price を取得 (SPY, QQQ, IWM, EFA, EEM, TLT, GLD, DBC, VNQ, etc.)
2. 189 日間の累積 return で cross-sectional ranking
3. Top 15 ETF を等加重で long
4. PortVol (portfolio volatility targeting) で 10% に normalize

### Universe 選定

87 ETF は以下のカテゴリから広く選定:
- **US Equity**: SPY, QQQ, IWM, MDY, IJH + sector (XLK, XLF, XLE, ...)
- **International**: EFA, EEM, VWO, IEFA, VEA
- **Fixed Income**: TLT, IEF, SHY, TIP, LQD, HYG, AGG
- **Real Assets**: GLD, SLV, DBC, VNQ, VNQI
- **Thematic**: XBI, ARKK, etc.

広い universe は diversification を高めるが、correlated ETF 間で ranking が不安定になるリスク。

## 6.3 パフォーマンス

### Standalone
```
Test Sharpe: +1.10
2019: +23.5%/yr (strong trend year)
2020: +16.4%/yr (post-COVID recovery rally)
2022: +12.7%/yr (commodity momentum saved)
```

### Portfolio 内での役割
- IV weight: ~4.5%
- Overlay: default path (Core component)
- Ablation: Drop GTAA → Sub2 -0.255

### Negative month 貢献
2023-05 で GTAA は **-3.20%** (月内最大の drag, portfolio の -0.16% の主因)。短期 momentum reversal に脆弱。

## 6.4 試行錯誤

### Lookback 期間の探索
- 63 日 (3 ヶ月): 過度に短期、turnover 高い
- 126 日 (6 ヶ月): moderate
- **189 日 (9 ヶ月)**: 最適 (v20 で確定)
- 252 日 (12 ヶ月): やや遅い反応

189 日は「12-1 month (skip 最新 1 ヶ月)」に相当し、学術的 momentum の最適期間と整合。

### Top N の探索
- Top 5: 集中しすぎ、volatility 高い
- **Top 15**: balanced (87 ETF の ~17%)
- Top 30: 希釈

### Sharpe × IV weighting (失敗)
component 間で Sharpe × inverse-volatility weighting を試みたが、diversification を破壊 (-0.36 Sharpe)。1/N が理論通り勝った。

## 6.5 他コンポーネントとの相関

```
GTAA vs MR:     -0.02 (独立)
GTAA vs CTA:    +0.15 (both momentum-based, moderate)
GTAA vs JP_LS:  +0.01 (独立)
GTAA vs IV_PREM: -0.07 (微逆相関)
```

CTA との moderate correlation (+0.15) は、両方が「trend-following」の変種であることを反映。しかし GTAA は cross-sectional (相対 ranking)、CTA は time-series (個別 asset の trend) → signal construction は異なる。

## 6.6 実務家へのノート

### 執行コスト
Top 15 ETF は全て highly liquid (SPY, QQQ, TLT 等) → slippage < 1bp/side。GTAA は全 components 中で**最も執行容易**。

### 実運用における月次 rebalance
GTAA の 189 日 momentum は月次でしか変化しない。日次 rebalance は不要だが、vol-targeting の調整のため daily が維持されている。

### GTAA 単独運用
GTAA standalone (Sharpe ~1.1) は「そこそこの戦略」であり、実運用可能だが exciting ではない。本ポートフォリオでの価値は**他 components との低相関による分散効果** (portfolio level で Sharpe +0.26 寄与)。

---

*次章: LV — Low Volatility Anomaly*
