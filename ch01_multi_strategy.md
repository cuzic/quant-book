# Chapter 1: マルチストラテジーポートフォリオとは何か

## 1.1 単一戦略の限界

クオンタティブ投資において、単一のアルファ源泉に依存するポートフォリオは構造的な脆弱性を持つ。どれほど精緻にバックテストされた戦略であっても、特定の市場レジーム（トレンド相場、レンジ相場、クラッシュ）に対する感応度を完全に除去することはできない。

本書で構築する v35 ポートフォリオは、17 のアルファコンポーネントと 5 層のオーバーレイを持つマルチストラテジー構造であり、単一戦略の限界を分散によって克服する設計思想に基づく。

## 1.2 先行事例

### Renaissance Technologies Medallion Fund
- 年率リターン 66% (1988-2018, 手数料前)
- Sharpe Ratio 推定 6-7 (Zuckerman 2019 "The Man Who Solved the Market")
- 数千の独立した統計的シグナルを組み合わせるアプローチ
- 本書の v35 (backtest Sharpe 6.87) はこの水準に近い数値を示すが、ライブ推定は 4.2±1.0 と保守的に見積もる

### AQR Capital Management
- Asness et al. (2013) "Value and Momentum Everywhere"
- 複数資産クラス × 複数ファクターの systematic approach
- 本書の Rate 4 components (R1/R2/R5/R6) はこのアプローチの簡素版

### Two Sigma / D.E. Shaw
- 機械学習 + 伝統的因子の hybrid
- 本書は ML を明示的に使用しないが、ORATS option data を cross-sectional signal として活用

## 1.3 本書のアプローチ: 17 × 5 層構造

```
Layer 1: 17 Alpha Components (逆ボラティリティ加重)
  ├─ Equity 5: MR, GTAA, LV, VC, MV, IM
  ├─ Rate 4:   R5(USD), R2(TIPS), R1(TLT), R6(DBC)
  ├─ Crisis 3:  CTA, MN, XLE_T
  ├─ Wavelet 1: T4b
  ├─ JP L/S 1:  JP_LS (10 factors × pnl-stacking)
  ├─ AU L/S 1:  AU_LS (ASIC short positions)
  └─ IV Prem 1: IV_PREM (VRP+Skew long-only) ← v35 new

Layer 2: J3 Overlay (日米センチメント乖離)
Layer 3: VIX Level Overlay (固定閾値 17/25)
Layer 4: ORATS EMA Overlay (PC×TS×IVP, Hybrid Shift)
Layer 5: Asymmetric Agreement (IC-weighted 5 z-score)
Layer 6: DD Control (Depth + Speed Sensing)
```

## 1.4 設計原則

本書を通じて一貫する設計原則は以下の 5 つである:

### 原則 1: 構造的安全性 (Structural Safety)
Look-ahead バグを convention ではなく structure で防止する。`safe_backtest.py` の `shift_forward()` は shift < 1 で例外を投げ、将来の開発者が accidentally に当日データを使用することを構造的に不可能にする。

### 原則 2: Binary Regime Detection
EMA overlay の 1.2/0.8 binary scaling は、continuous tanh 化よりも一貫して優れる (Chapter 21 で詳述)。これは「signal の情報損失」ではなく「regime detection の decisiveness」が alpha の源泉であるためである。

### 原則 3: 非対称防御 (Asymmetric Defense)
DD control は downside のみに反応し、upside を制限しない。Vol cap (対称) は cost > benefit (Chapter 23)。

### 原則 4: Walk-Forward Everything
全ての parameter 選択は expanding-window walk-forward で検証。Full-period 最適化は DEFAULT_SQRT_S / DEFAULT_IC にのみ残存し、これらは learned_weights で上書き可能。

### 原則 5: Negative Results の価値
失敗した 30+ の戦略・改善案は、成功した改善と同等の学術的価値を持つ。Chapter 32 で全ての negative result を報告する。

## 1.5 本書の構成

Part I (Ch 1-4): 基礎理論
Part II (Ch 5-18): 17 コンポーネントの個別解説
Part III (Ch 19-23): 5 層オーバーレイシステム
Part IV (Ch 24-27): ポートフォリオ構築と検証
Part V (Ch 28-30): 実装と運用インフラ
Part VI (Ch 31-33): 開発ドキュメンタリー

各章は以下の統一フォーマットで記述:
1. 学術的根拠 (先行研究)
2. 実装詳細 (Python コード参照)
3. バックテスト結果 (per-year, Sub2)
4. 試行錯誤経緯 (tested & rejected variants)
5. 他コンポーネントとの相関・相互作用

---

*次章: ポートフォリオ理論の数学的基礎*
