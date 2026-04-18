# Chapter 18: AU_LS — ASIC Short Position と UTF-16 文字化け修正

## 18.1 ASIC (Australian Securities and Investments Commission) データ

オーストラリアの金融規制当局 ASIC は、日次で全上場銘柄の aggregate short position を公開:
- `% of Total Product in Issue Reported as Short Positions` (空売り比率)
- 報告遅延: T+2 (2 営業日)
- 公開期間: 2010 年〜 (ただし初期ファイルに encoding 問題)

**学術的根拠**: Aitken et al. (1998) "Short Sales Are Almost Instantaneously Bad News" — ASX での空売り情報の予測力を実証。Boehmer, Jones & Zhang (2008) "Which Shorts Are Informed?" は機関空売りの情報価値を確認。

## 18.2 UTF-16 文字化け: 613 ファイルの修正

### 問題の発見

開発初期のセッションで「12 ファイルが encoding error」と記録。本セッションで調査すると**実際は 613 ファイル** (2020-01 〜 2022-06) が UTF-16 LE BOM encoded:

```
Good files (2022-06+): UTF-8, comma-separated
Bad files (2020-01〜2022-06): UTF-16 LE BOM (0xff 0xfe), tab-separated
```

**両方とも同じ 5 カラム構造** — encoding と delimiter が異なるだけ。

### 修正

`fix_au_encoding.py`:
1. 613 ファイルを backup (`data/au_short_positions_backup/`)
2. UTF-16 LE + tab → UTF-8 + comma に変換
3. 全 1587 ファイル readable に

### 影響

```
AU_LS data range: 2022-06+ → 2020-01+ (2.5 年追加)
Portfolio impact:
  Sub1 (2019-21): +6.099 → +6.237 (+0.138) ⚡
  Sub2 (2022+):   +6.294 → +6.291 (unchanged)
```

**教訓**: `try/except pass` (silent skip) は技術的負債。skip 件数を log に記録し、定期的に review すべき。

## 18.3 Signal 設計: 4-Feature Rank-Equal Composite

| Feature | 計算 | Direction |
|---|---|---|
| F1 | -short_pct (raw level inverse) | Long LOW (under-shorted) |
| F2 | short_pct.diff(5) (5d flow) | Long NEGATIVE flow (空売り減) |
| F4 | -z-score(short_pct, 63d) | Long LOW z-score |
| F5 | -rank(short_pct, 63d, pct) | Long LOW percentile |

Composite: 4 feature を cross-sectional rank → 平均。

### Rank-equal vs PnL-stacking

JP_LS は PnL-stacking (Chapter 17) で standalone Sharpe 3x 改善。AU_LS でも PnL-stacking テスト (S4):
```
Rank-equal: Sub2 +1.409
PnL-stack:  Sub2 +1.744 (+0.335, 但し p=0.32 非有意)
```

PnL-stacking は promising だが統計的に非有意 → **rank-equal を維持** (conservative)。

## 18.4 Universe

```
ASIC 公表銘柄 ∩ yfinance (.AX) ∩ Volume > 100K/日 ≈ 237 銘柄
```

JP_LS (909 銘柄) より小さいが、AU 市場の liquid segment をカバー。

## 18.5 L/S Portfolio 構築

```python
L/S: Top 30 long / Bottom 30 short (rank composite)
Hold: 21 日 overlap (POSITION averaging — not PnL!)
Shift: 2 日 (ASIC T+2 reporting delay)
Vol target: 10%
```

JP_LS と同じ hold overlap 構造だが、**position averaging** (正しい実装)。

## 18.6 v32 overlay: VIX_ONLY

AU 株には US ORATS option overlay は noise (AU market structure ≠ US)。VIX overlay のみが effective:
```
VIX < 17 → 1.2x (global calm → AU も calm)
VIX > 25 → 0.8x (global stress → AU も影響)
```

v31 でこの発見 → portfolio Sharpe +0.14 (AU_LS VIX-only overlay)。

## 18.7 パフォーマンス

```
Standalone Sharpe (2022-06+): +1.49
Standalone Sharpe (2020-01+): +1.27 (extended data)
IV weight: ~4.3%
Correlation with v34: -0.085 (独立 alpha!)
Ablation: Drop AU_LS → Sub2 -0.133
```

AU_LS の最大の価値は **低相関 (-0.085)** による分散効果。Standalone alpha は modest だが、portfolio level で独立に寄与。

## 18.8 UK FCA データの不採用

オーストラリアに先立ち、UK FCA (Financial Conduct Authority) の short position data も検討:
- 104,625 rows (2012-2026)
- しかし **issuer-ticker mapping が 46% しか match しない**
- Standalone Sharpe +0.44 (AU_LS +1.49 の 1/3)

**判断**: UK_LS は不採用 (data quality 不足)。AU_LS のみ採用。

## 18.9 Edge の持続可能性

### 構造的 edge
1. ASIC 日次公表は比較的新しい制度 → academic exploitation 少ない
2. AU 市場は US/JP より alpha research が少ない → competition 低い
3. Regulatory data = 信頼性高い (manipulation 不可)

### リスク
1. ASIC 公表制度の変更 (遅延拡大、頻度変更)
2. AU CFD の借入コスト (IB 経由で ~5%/yr)
3. AU 市場の liquidity 制約 (237 銘柄、¥100M 規模でも capacity issue)

---

*Part III 開始: Chapter 19 — J3: JP-US Sentiment Divergence Overlay*
