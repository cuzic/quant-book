# Chapter 10: VC — VIX Contango: Crisis Alpha の通説を覆す

本章は本書のハイライトの一つである。VIX Contango 戦略 (short volatility carry) が「crisis alpha」として機能するという通説を、実データに基づき**完全に否定**する。

## 10.1 VIX Contango の基本メカニズム

VIX 先物は通常 contango (期近 < 期先) の term structure を持つ。これは **insurance premium**: 投資家は将来の volatility に対して premium を支払う。

VC 戦略: contango 時に期近 short / 期先 long → roll yield を収穫。
実装: `build_vix_contango()` で VIX/VXV ratio をモニタリング。

## 10.2 通説: "VC は crisis alpha"

ChatGPT (外部レビュー) の初期評価:
> 「VC / VIX は regime-dependent — evaluate by skew, tail protection, convexity, NOT just Sharpe」
> 「平常時: drag、危機時: 爆発的に効く」
> 「Sharpeではなく skew で評価すべき」

**この仮説を実データで検証した。**

## 10.3 実データ: 通説の完全な否定

### Stress event での VC 挙動

| Event | VC Return | SPY Return | VIX max |
|---|---|---|---|
| **2020 COVID crash** (24d) | **-7.05%** | -33.40% | 82.7 |
| 2020 recovery (69d) | +2.12% | +38.92% | 65.5 |
| 2022 bear (196d) | +2.34% | -24.06% | 36.5 |
| 2024 Aug unwind (14d) | +5.56% | +1.59% | 38.6 |

**COVID crash で VC は -7.05% 損失**。SPY のヘッジではなく、**追加の損失源**。

### VIX regime 別分析

```
VIX regime              VC Sharpe    VC return     VC effect on portfolio
Low (< 15)               +4.28       +38.18%/yr    +0.508 Sharpe
Normal (15-20)            +1.00       +10.87%/yr    -0.747 Sharpe ⚠
Elevated (20-25)          -1.24       -15.38%/yr    -0.298 Sharpe ⚠
High (25-35)              -0.67        -9.74%/yr    -0.318 Sharpe ⚠
Crisis (>= 35)            -2.93       -41.66%/yr    -0.467 Sharpe 🚨
```

**VC は VIX < 15 (calm) でのみ alpha、VIX > 20 で全て negative。**

### Distribution 比較

| Metric | v33 (with VC) | v33 - VC | 改善方向 |
|---|---|---|---|
| Sharpe | 6.138 | **6.409** (+0.27) | 除去 |
| Kurtosis | **9.5** | **1.4** | 除去 |
| Skew | +1.26 | +0.39 | 維持 |
| MaxDD | -1.00% | -1.02% | ≈同 |

**VC を除去すると Kurtosis が 9.5 → 1.4 に激減**。VC が portfolio の fat tail の主因。

## 10.4 VC の正体: Negative Convexity Carry

VC は「crisis alpha」ではなく、**negative convexity の carry 戦略**:
- **Calm**: premium を collect (positive carry)
- **Stress**: premium が explode (massive loss)
- 構造的に**保険の売り手** = left tail risk

これは典型的な "picking up pennies in front of a steamroller" パターンである。

## 10.5 なぜ VC を維持するのか

Ablation で VC 除去は Sub2 +0.218 の改善を示したが、**paired t-test で非有意** (p=0.198)。

さらに、Sub2 (2022-2026) では VC effect が**微小 positive** (-0.07 ablation Δ で Sub2 がやや悪化)。VC の calm carry は最近の低 VIX 環境で機能している。

**判断**: VC は維持。理由:
1. 除去の統計的有意性なし
2. Sub2 で neutral-to-positive
3. Kurtosis は vol cap ではなく DD control で対処 (Chapter 23)
4. VC weight は IV-weighting で自動調整 (alpha 低下 → vol 上昇 → weight 減)

## 10.6 研究者への教訓

1. **"Crisis alpha" のラベルを疑え**: Short vol carry は calm alpha + crisis drain
2. **Sharpe 以外の metric で分析**: VIX regime 別分解が本質を明らかにした
3. **通説を data で検証**: ChatGPT の「crisis alpha」仮説は empirically falsified
4. **Kurtosis の source を特定**: 単一 component (VC) が portfolio 全体の tail risk を支配し得る

---

*次章: IV_PREM — Implied Volatility Premium*
