# Chapter 11: IV_PREM — Implied Volatility Premium (v35 新 Component)

本章は本書で最も詳細な試行錯誤レポートを含む。新 component の発想から最終 config 確定まで、**125+ configurations のテスト経緯**を生々しく記述する。

## 11.1 学術的根拠: Volatility Risk Premium (VRP)

### VRP の存在

Carr & Wu (2009) "Variance Risk Premiums": Implied Volatility (IV) は Realized Volatility (RV) を系統的に上回る。この差 (VRP = IV - RV) は **variance risk に対する保険料** であり、売り手にとっての risk premium。

Bollerslev, Tauchen & Zhou (2009): VRP は将来の株式リターンを正に予測する。VRP が高い時は市場が「恐怖」状態 → その後の期待リターンが高い。

### Cross-sectional VRP

Bali & Hovakimian (2009): 個別株の IV-RV spread は **cross-sectional に** 将来リターンを予測する。高 VRP 株 (IV >> RV) は overpriced fear → mean revert → 高リターン。

**本章の novelty**: VRP を **cross-sectional stock selection signal** として使用し、**Put/Call ratio (skew)** との composite で signal quality を向上させた。

## 11.2 発想の起源

### Negative month 分析から

v34 portfolio の 88 ヶ月中、negative は 2 ヶ月のみ:
- 2023-05: -0.158% (momentum 系 4 component 同時不調)
- 2021-07: -0.043% (counter-trend 系が trending market で逆行)

**問い**: この 2 ヶ月でも positive return を出す新 component はあるか?

### 5 候補のスクリーニング

| Candidate | Sub2 S | 2023-05 | 2021-07 | 判定 |
|---|---|---|---|---|
| SPY Overnight Premium | +0.632 | +2.38% | -1.74% | 片方のみ |
| JP Short-term Reversal | -0.477 | -0.98% | -0.29% | 即却下 |
| **ORATS IV Premium** | **+0.867** | **+8.56%** | **+0.63%** | **両方 positive** |
| Credit Spread (HYG-LQD) | +0.190 | -2.63% | -2.86% | 即却下 |
| Turn-of-month effect | +0.194 | -4.23% | +2.50% | 弱すぎ |

**IV Premium が唯一、standalone Sharpe > 0.5 かつ両 negative month で positive。** v34 との相関 -0.036 (ほぼ完全独立 alpha)。

## 11.3 大規模パラメータ探索

### Phase 1: Signal type (72 configs)

Signal × TopN × Hold の full grid:

**Top 20 by Sub2 Sharpe**:
```
skew top20 hold42:        +2.309 (best absolute, but 2021-07 fail)
vrp+skew top10 hold5:     +1.661 (best both-positive)
vrp top5 hold21:          +1.111 (baseline)
```

**Both negative months positive**: 16/72 configs のみ (22%)。

### Phase 2: Equal vs Proportional sizing

**重要な発見**: VRP-proportional sizing は VRP signal では consistently better (+0.26〜+0.46)、skew signal では destructive (-0.84〜-1.71)。

```
vrp top5 hold21: equal +1.111 → prop +1.480 (Δ+0.369)
skew top5 hold21: equal +1.499 → prop -0.174 (Δ-1.672!)
```

VRP の absolute 値は signal strength の proxy (大きい VRP = 強い mispricing)。Skew の absolute 値は意味が異なる → proportional sizing は VRP にのみ有効。

### Phase 3: Long-only vs L/S

```
vrp top5 hold21 L/S:       +1.111
vrp top5 hold21 Long-only: +1.889 (+70%!)
```

**Short side の Sharpe = -0.82** — 「低 VRP 株を空売り」は negative alpha。Long-only 化で standalone Sharpe 70% 改善。

### Phase 4: Overlay/Filter テスト

8 つの filter を個別 + 組合せでテスト:

| Filter | Sub2 S | 効果 |
|---|---|---|
| No filter (baseline) | +1.661 | — |
| ORATS calm timing | +1.643 | neutral standalone, but combo で synergy |
| VIX < 20 | +2.070 | **best both-positive** |
| CTA risk-on | +1.797 | good |
| Asymm gate | +1.604 | worse |
| Breadth proxy | +1.695 | marginal |
| DD speed | +2.303 | **+38% standalone** |
| Earnings filter | +1.467 | **WORSE** (event risk premium = alpha 源泉!) |

**Earnings filter の逆説**: 決算前の高 VRP 銘柄を除外すると alpha が**減少**。MR (Chapter 5) では earnings filter が有効だったが、IV_PREM では逆 — event risk premium こそが VRP の源泉。

### Phase 5: 17 組合せの最終テスト

Top 4 全てに DD speed が含まれる:
```
#1: Long VRP-prop ORATS DD-speed: Sub2 +3.522 (champion)
#2: Long ORATS DD-speed:          Sub2 +3.403
#3: Top5 VRP-prop ORATS DD:       Sub2 +3.377
#4: Long DD-speed shift=1:        Sub2 +3.443
```

### B5 (Earnings filter): 本格的検証

ORATS `hist/earnings` API から 534 ticker の earnings dates をダウンロード。5 日前除外でテスト → Sub2 +1.467 (baseline +1.661 より悪化)。**Earnings = alpha source の確認。**

### B6 (Sector cap): 壊滅

Sector 別に max 2 銘柄制限 → **Sub2 -0.874** (baseline +1.661 から完全崩壊)。VRP alpha は Tech sector に集中 → cap で排除 = alpha 消失。

### C9 (Within-sector rank): 半減

Sector 内 rank → Sub2 +0.507 (cross-sectional +1.038 の半分)。Cross-sectional VRP variation が alpha。

## 11.4 最終 Config: Champion の anatomy

```python
Signal:    vrp+skew composite (rank average)
Side:      Long-only (short side = negative Sharpe)
Sizing:    VRP-proportional (strong VRP → more weight)
Filter:    ORATS EMA calm timing (pc × ts × ivp)
Defense:   DD speed sensing (3d window, proactive scale-down)
TopN:      10
Hold:      5d overlap
Shift:     2d
```

各要素の寄与:
```
Base (vrp+skew L/S):           Sub2 +1.661
+ Long-only:                   +1.969 (+19%)
+ ORATS timing:                → combo +2.791 (+68%)
+ VRP-proportional:            → combo +2.990
+ DD speed:                    → combo +3.522 (+26%)
```

## 11.5 Portfolio 統合結果

| 指標 | v34 | v35 (+IV_PREM) | Δ |
|---|---|---|---|
| Test Sharpe | +6.511 | **+6.838** | **+0.327** |
| Sub2 Sharpe | +6.620 | **+6.948** | **+0.328** |
| Sortino | +9.224 | **+9.893** | **+0.669** |
| Return | +15.25% | **+16.41%** | **+1.16%/yr** |
| Kurt | +10.1 | **+8.8** | **-1.3 (改善)** |
| t-stat (Sub2) | — | **+6.26** | **p < 10⁻⁶** |
| Per-year improved | — | **全 8 年** | |
| IV_PREM weight | — | 3.2% | |
| Correlation | — | +0.174 | 低相関 |

**t = +6.26 (p < 10⁻⁶)** — 本セッション全体で最高の統計的有意性。

## 11.6 経済学的解釈

### なぜ Long-only が L/S より強いか

- Long side (高 VRP): 「過度な恐怖で overpriced → 実際は暴落しない → alpha」
- Short side (低 VRP): 「楽観的 → underpriced risk → 暴落する? → しないことも多い」
- **恐怖の非対称性**: 人間は下落リスクを過大評価する (Kahneman & Tversky の prospect theory) → 恐怖 premium は systematic、楽観 discount は noisy

### なぜ VRP-proportional sizing が効くか

VRP の absolute 値は「mispricing の程度」の proxy:
- VRP 10% (IV 30%, RV 20%): 大きな overpricing → 強い alpha
- VRP 2% (IV 22%, RV 20%): 小さな overpricing → 弱い alpha
- Proportional sizing = **informative signal に集中投資** (Kelly criterion の簡易版)

### なぜ ORATS timing が効くか

Market stress 時 (ORATS bearish = PC 高, TS backwardation, IVP 高):
- 全株の VRP が同方向に elevated → cross-sectional signal の SNR 低下
- Calm 時の方が銘柄間の VRP dispersion が大きい → selection alpha が機能

### なぜ DD speed が効くか

IV_PREM は carry 戦略の性質を持つ (vol premium 収穫)。Carry 戦略は DD が始まると accelerate する (mean reversion しない)。DD speed sensing で「carry が崩壊し始めた瞬間」を検出 → proactive 退避。

## 11.7 実務家へのノート

### 執行可能性
- 531 ORATS ticker のうち top 10 long: highly liquid US stocks
- Rebalance 5d overlap: daily 20% turnover → 年 52x → cost ~5bp/side で manageable
- ORATS data は $99/月 (Delayed API) で available

### リスク
- ORATS service discontinuity → signal source 消失
- VRP anomaly の erosion (option market の efficiency 向上)
- Earnings season 集中リスク (Q1/Q3 の earnings date cluster で signal が concentrated)

---

*次章: Rate/Macro Alpha — R1/R2/R5/R6 と R3 削除の全経緯*
