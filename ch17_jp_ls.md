# Chapter 17: JP_LS — 日本株 10-Factor PnL-Stacking

本章は本ポートフォリオで**最も独自性の高い component** を解説する。日本固有の公開データ (J-Quants 信用倍率、機関別空売り報告書) を用い、10 factor の PnL-stacking で standalone Sharpe +2.6 を達成する。

## 17.1 データソースの独自性

### J-Quants 信用倍率

日本証券金融が週次公表する信用取引残高:
- LongVol (買い残): 信用買いの株数
- ShrtVol (売り残): 信用売りの株数
- Credit Ratio = LongVol / ShrtVol: 「楽観度」の proxy

**学術的根拠**: Hirose, Kato & Bremer (2009) が日本市場で margin traders の予測力を実証。しかし cross-sectional selection signal としての活用は本書が初。

### 機関別空売り報告書

金融庁が日次公表する個別銘柄の機関別空売りポジション:
- ShrtPosToSO: 発行済株式に対する空売り比率
- SSName: 機関名 (Goldman Sachs, Morgan Stanley 等)
- 報告頻度: T+2 (2 営業日遅延)

## 17.2 Universe 構築

```
JPX 上場全銘柄 ∩ IB CFD 対象 ∩ Volume > 100K/日 = 909 銘柄
```

IB CFD universe (`data/ib_cfd_all.json`) は IB 証券で実際に取引可能な銘柄リスト。Volume filter で流動性を確保。

## 17.3 10 Factor の設計

### STATIC factors (7 個): Position level ベース

| Factor | Data | Signal | Direction |
|---|---|---|---|
| B1 | credit_ratio (LongVol/ShrtVol) | Raw level | Long HIGH |
| B4 | credit_ratio × 21d momentum | Interaction | Long HIGH |
| B7 | credit_ratio z-score (63d) | Z-score | Long HIGH |
| C5 | short_ratio z-score (63d) | Z-score, EMA(14) | Long HIGH |
| F1 | USDJPY_21d × sector_sign | Cross-asset interaction | Long LOW |
| B5 | short_maturing ratio | Maturity structure | Long LOW |
| B6 | long_maturing ratio | Maturity structure, EMA(63) | Long LOW |

### FLOW factors (3 個): Position change ベース (v24 追加)

| Factor | Data | Signal | Direction |
|---|---|---|---|
| FL_1m | All-institution pos_change 21d sum | 1-month flow | Long HIGH |
| FL_1w | All-institution pos_change 5d sum | 1-week flow | Long HIGH |
| BLG | Bulge-bracket pos_change 21d sum | Smart money flow | Long HIGH |

### CONC (v24 で追加後 dropped)
Top-1 concentration ratio: Sharpe -0.19 で harmful → 除外。残骸コード (`conc_pv`) はセッション中に発見・削除 (Chapter 31)。

## 17.4 Factor 設計の詳細

### F1: USDJPY × Sector 

```python
EXPORT_33 = {'電気機器', '輸送用機器', '精密機器', '機械', '鉄鋼', '非鉄金属', '化学'}
DOMESTIC_33 = {'銀行業', '保険業', 'その他金融業', '不動産業', ...}
sector_sign = export_flag - domestic_flag  # +1, 0, -1
F1 = USDJPY_momentum_21d × sector_sign
```

円安 → 輸出セクター有利、円高 → 内需セクター有利。この interaction を cross-sectional signal として使用。

**注意**: 17 業種以外 (sector_sign = 0) は F1 factor で sorting されない → universe の ~40% が neutral。これは**設計上の意図** (為替感応度がない銘柄には signal を適用し��い)。

### BLG: Bulge Bracket Flow

13 大手機関 (`data/bulge_brokers.json`):
```
Morgan Stanley MUFG, Credit Suisse, JPM Securities, Nomura International,
Goldman Sachs International, Merrill Lynch, UBS, Barclays Capital,
Citigroup, Deutsche Bank London, J.P. Morgan Securities PLC,
BNP Paribas Arbitrage, Barclays Bank PLC
```

「smart money」の flow direction が一般投資家に先行するという仮説。

## 17.5 PnL-Stacking 合成方式

### Rank-average の問題 (v22)

v22 で 7 factor を rank-average した結果: Component Sharpe **0.86** (各 factor standalone 1.0-2.5 より大幅に低下)。

**原因**: Rank-average は「全 factor の合意」を求めるため、individual factor の strong conviction を希釈。Factor A が strongly long でも Factor B が neutral なら、composite は weak long。

### PnL-Stacking の解決 (v23)

各 factor で**独立に L/S portfolio を構築** → 各 factor の PnL stream を結合。

```python
for factor in factor_cfg:
    factor_pnls[name] = build_factor_ls(data, sign, ema, top_n=30, hold=21)

# sqrt(Sharpe) 加重で結合
weights = np.sqrt([sharpe(pnl) for pnl in factor_pnls])
combined = weighted_average(factor_pnls, weights)
```

結果: Component Sharpe **+2.55** (rank-average 0.86 の 3 倍)。

### sqrt(Sharpe) 加重の根拠

- 強い factor (B1: Sharpe 2.15) に多く配分
- 弱い factor (F1: Sharpe 0.54) に少なく��分
- sqrt を使う理由: linear (Sharpe) だと強い factor に過度に集中、sqrt は「diminishing returns to strength」を反映
- learned_weights (`data/learned_weights/`) で monthly 更新可能

## 17.6 Cost Reduction (v28)

v27 → v28 で大幅なコスト削減:
```
TopN: 50 → 30 (銘柄数 40% 減)
Hold: mixed (5d/10d) → 21d 統一
Gross Sharpe: 3.05 → 2.57 (Δ -0.48)
Net Sharpe @ 15bp: 0.70 → 1.53 (Δ +0.83!)
```

Gross Sharpe は下がるが、turnover 半減 (105x → 56x/yr) により **Net Sharpe が +0.83 改善**。

## 17.7 Portfolio 内での役割

```
Standalone Sub2 Sharpe: +2.50
IV weight: ~6.7%
Ablation: Drop JP_LS → Sub2 -0.408 (2 番目に大きい寄与)
Correlation with portfolio: +0.04 (ほぼ独立)
```

JP_LS は **negative month で rescue component** として機能:
- 2023-05: JP_LS = **+4.40%** (portfolio の -0.16% を +0.29% で offset)
- 2021-07: JP_LS = +0.46% (positive)

## 17.8 Edge の持続可能性

### 構造的 edge
1. **日本固有データ**: 信用倍率は日本市場のみ → global quant fund が参入しにくい
2. **機関空売り報告**: 日次公表は 2012 年から → 歴史浅くアカデミック研究少ない
3. **IB CFD via IBSJ**: 日本居住者向けの specific infrastructure

### リスク
1. **J-Quants API 廃止**: データソース消失 → JP_LS 運用不能
2. **信用取引制度変更**: 規制変更で信用倍率の意味が変わる
3. **機関行動の変化**: アルゴ化で「smart money」premium が erosion

---

*次章: AU_LS — オーストラリア株 ASIC Short Position*
