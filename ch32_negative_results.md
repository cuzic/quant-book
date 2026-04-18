# Chapter 32: 失敗した戦略たち — 徹底的な Negative Result 報告

Publication bias (positive results のみ報告) は academic finance の深刻な問題 (Harvey, Liu & Zhu 2016)。本章では**全ての失敗を honest に報告**する。

## 32.1 過去セッションの Rejected 戦略

| 戦略 | Sharpe | 却下理由 |
|---|---|---|
| Option selling (VRP) | N/A | bid-ask 10.7bp/日 > VRP 5bp/日 で赤字 |
| Leverage ETF | N/A | ボラティリティ減価で長期保有不可 |
| Statistical Arbitrage | <0.2 | 243 ペア中 Sharpe > 0.2 がゼロ |
| Seasonal Rotation | N/A | 13 戦略中 12 本マイナス |
| Rate Regime Bond Short | ~0 | Vol 揃えると効果消失 |
| ML Sizing (RF/XGBoost) | — | Stage 2 gate FAIL |
| GARCH Walk-Forward | — | Rolling std より悪い (null result) |

## 32.2 本セッションの Rejected 改善案

### Signal 情報損失の修復 (全て失敗)

| テスト | 予測 (ChatGPT) | 実測 | t-stat |
|---|---|---|---|
| **S1 EMA continuous** | +0.1〜0.3 | **-0.57** | **-6.58** |
| **S2 Asymm smooth** | +0.05〜0.2 | eval +0.04, prod **-0.22** | — |
| **S3 Overlay geometric** | +0.0〜0.15 | **-1.13** | **-9.79** |
| A6 core_extra continuous | — | cancelled (S1 依存) | — |

**教訓**: 「binary → continuous で情報損失を埋める」は ChatGPT の予測に反し、全て失敗。Binary regime detection は signal ではなく categorical classification として機能。

### Position sizing 改善 (失敗)

| テスト | 実測 |
|---|---|
| A2 Signal-proportional (AU_LS) | Δ -0.02 (neutral) |
| A3 CTA vol-parity | Δ -0.14 (悪化) |

### VIX overlay adaptive 化 (全て失敗)

| テスト | Sub2 |
|---|---|
| Percentile tanh 504d | -0.232 |
| Percentile tanh 252d | -0.239 |
| Percentile binary 504d | -0.189 |
| Z-score tanh 252d | -0.191 |

**固定閾値 17/25 が全 adaptive 版に勝利。** Market convention > data-driven adaptive。

### Hold overlap (致命的バグ)

| テスト | 見かけの改善 | 真実 |
|---|---|---|
| MN hold 10d | +2.11 Sub2 | **PnL averaging artifact** |
| CTA hold 10d | +1.93 Sub2 | **同上** |
| Portfolio v34 | +1.39 Sub2 | **全て偽** → revert |

「PnL smoothing + vol-targeting = Sharpe inflation trap」は本書の**最重要教訓**。

### IV Premium の rejected variants

| Variant | Sub2 | 却下理由 |
|---|---|---|
| VRP ratio (iv/hv) | +0.791 | baseline +1.111 より劣 |
| TS slope only | +0.738 | 両 month fail |
| IVP inverse | -0.801 | negative Sharpe |
| Earnings filter | +1.467 | alpha 減少 (event risk = alpha source!) |
| Sector cap (max 2) | **-0.874** | alpha **完全破壊** |
| Within-sector rank | +0.507 | Sharpe 半減 |
| Skew-proportional sizing | -0.174 | VRP-prop とは逆に壊滅 |

### Weighting scheme (全て baseline 以下)

1/N, HRP, inv-corr, rank selection (top 8/10/12), Sharpe-weighted, sqrt(Sharpe) — 全て inverse-volatility に劣る。

### Vol cap (コスト > ベネフィット)

- 3.5%+ cap: 発動 2-3%、Kurt/MaxDD 変化なし
- 2.5% cap: Kurt -27% だが cost -1.16%/yr
- DR 分析: crisis で DR 上昇 → vol cap が守るリスクが存在しない

## 32.3 失敗のパターン分析

### Pattern 1: 「Information recovery」の失敗
Binary → continuous、rank → z-score、step → smooth — 全て「情報損失を回復する」試みだが、実際には「categorical regime detection」が alpha であり、continuous 化は decisiveness を失う。

### Pattern 2: 「Smart weighting」の失敗
Sharpe-weighted、rank selection、HRP — 「良い component を重く、悪い component を軽く」は直感的に正しいが、estimation noise で逆効果。Simple (1/N or inv-vol) が robust。

### Pattern 3: 「More is better」の失敗
Sector cap、within-sector rank、earnings filter — 「filter を追加すれば精度向上」だが、実際は alpha source を除外。Less constraint = more alpha。

### Pattern 4: 「PnL の smoothing は改善」の失敗
最も危険なパターン。PnL averaging は mean return を維持しつつ vol を低下 → vol-targeting が leverage 増 → Sharpe 膨張。統計検定も pass するため、検出が困難。

## 32.4 「失敗の ROI」

30+ の negative result は v35 の信頼性を**劇的に向上**:
1. 「これらの改善が効かない」= v35 の baseline が robust
2. Binary > continuous の一般原則の発見
3. PnL averaging trap の教訓 (Chapter 13)
4. VRP alpha の本質理解 (earnings = alpha, sector cap = death)

**Negative results なしに positive results は信頼できない。**

---

*最終章: ChatGPT/Claude との共同設計*
