# Chapter 7: LV — Low Volatility Anomaly

## 7.1 学術的根拠

### "High Risk = High Return" の否定

Ang, Hodrick, Xing & Zhang (2006) の衝撃的発見: **高ボラティリティ株は低リターン、低ボラティリティ株は高リターン**。これは CAPM の基本的予測 (β が高いほど expected return が高い) と矛盾する。

### Betting Against Beta

Frazzini & Pedersen (2014) は leverage constraint 理論でこの anomaly を説明: 多くの投資家は leverage に制約があり (年金基金、投信)、高 β 株を「レバなしで market beat」するために over-bid → 高 β = overvalued, 低 β = undervalued。

Baker, Bradley & Wurgler (2011) は benchmark-relative incentive を追加: fund manager は benchmark 対比で評価されるため、低 β 株を underweight する動機がある → 構造的な mispricing。

### Anomaly の持続性

Low-vol anomaly は 1926-2023 の US データで robust (Blitz & van Vliet 2007)、かつ global に存在 (23 カ国で確認)。Implementation cost が低い (低 turnover) ため、取引コスト後も生存。

## 7.2 実装: Hybrid JP + non-JP

```python
# portfolio.py: _build_lv()
from eval_final_v8_real import build_hybrid_lv
lv = build_hybrid_lv(self.pre, self.spy_on_stocks, self.margin_df, top_n=150)
```

**Hybrid 設計**:
- **JP stocks**: J-Quants 信用倍率 (J14 factor) ベースの低 vol 選定
  - 信用倍率が高い (Long/Short 比率高い) = 機関投資家が楽観 = low-vol 株に多い
  - margin_df との cross: vol + 信用情報の複合 signal
- **Non-JP stocks**: Beta-SPY ベースの低 β 選定
  - SPY に対する rolling β を計算
  - 低 β 株 Top 150 を long

### なぜ JP と non-JP を分けるか

日本株市場の特異性:
1. 信用取引の情報が J-Quants で公開 (他国にない unique data)
2. JP の low-vol anomaly は「信用倍率」が additional signal
3. Non-JP では信用倍率がない → Beta-SPY が代替

## 7.3 パフォーマンス

```
Standalone Test Sharpe: +1.44
IV weight: ~5.6%
Ablation: Drop LV → Sub2 -0.460 (portfolio で 3 番目に大きい drag)
```

LV は standalone Sharpe が高く、portfolio への寄与も大きい。low-vol 株の安定的なリターンが portfolio 全体の vol を抑制し、Sharpe 分母を小さくする。

## 7.4 Negative month での行動

```
2023-05: LV = -1.75% (4 番目の drag)
2021-07: LV = -0.04% (neutral)
```

2023-05 の下落は、低 vol 株が momentum reversal で一時的に underperform したため。低 vol anomaly は長期的に robust だが、短期的 (月次) には momentum に劣ることがある。

## 7.5 Edge の持続可能性

Low-vol anomaly は **構造的** (leverage constraint は制度的) であり、裁定で消えにくい:
- ETF (USMV, SPLV) の AUM 増加にも関わらず anomaly は持続
- 「低 vol 株を買う」は退屈 → behavioral barrier
- Leverage constraint は金融規制で強化傾向

**リスク**: 金利環境の変化。低 vol 株は duration が長い (utility, REIT) → 金利上昇で relative underperformance。2022 年の rate shock でこれが顕在化。

---

*次章: IM — Idiosyncratic Momentum*
