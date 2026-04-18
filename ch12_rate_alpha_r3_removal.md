# Chapter 12: Rate/Macro Alpha と R3 削除の全経緯

## 12.1 Rate Component 群の設計

### R5: USD Momentum (UUP)
- Signal: Absolute momentum 126d
- 根拠: Menkhoff et al. (2012) "Carry Trades and Global Foreign Exchange Volatility"
- FX momentum は equity momentum と低相関
- v32 で J3_VIX overlay に分類 (ORATS は不要、currency → J3+VIX のみ)

### R2: TIP-TLT Spread (TIPS Breakeven)
- Signal: TSMOM 12-1 on TIP/TLT ratio
- 根拠: Breakeven inflation rate の trend = インフレ期待の方向
- v32 で NO_OVERLAY に分類 (rates は option market overlay 不要)

### R1: TLT Dual MA Cross
- Signal: 50/200 日 dual moving average on TLT
- 根拠: 古典的 trend following (Glabadanidis 2015)
- Standalone Sharpe +0.27 (最弱 component)
- v32 で NO_OVERLAY

### R6: DBC Commodity Momentum
- Signal: Absolute momentum 252d on DBC
- 根拠: Erb & Harvey (2006) "The Strategic and Tactical Value of Commodity Futures"
- Commodity momentum は equity cycle と低相関

## 12.2 R3 (VTV-VUG): Structural Break の発見と削除

### R3 の original 設計
- Signal: TSMOM 12-1 on VTV/VUG ratio (Value vs Growth spread)
- 根拠: Fama & French (1993) HML factor の time-series momentum 版
- v32 まで Rate 5 の一つとして含まれていた

### Ablation での発見

v32 portfolio ablation (Chapter 26):
```
Drop R3: Test Sharpe +0.215 改善, Sub2 +0.274 改善
```

**R3 を除去すると portfolio が改善** — R3 は portfolio の drag。

### Structural Break の証拠

VTV-VUG correlation の時系列変化:
```
2015: 0.918    2018: 0.905
2016: 0.893    2019: 0.871
2017: 0.698    2020: 0.867
                2021: 0.472 ⚡ 崩壊
                2022: 0.822
                2023: 0.630
                2024: 0.530
                2025: 0.740
```

**2021 年に correlation が 0.92 → 0.47 に崩壊**。以降、安定せず 0.53-0.82 を変動。

R3 standalone Sharpe per year:
```
2020: +2.52    2021: -1.09    2022: +1.12
2023: -0.38    2024: +1.02    2025: -0.09
```

2021 年以降、signal が noise 化。

### 経済学的説明

ChatGPT の分析:
> 「Value vs Growth」ではなく「Mega-cap vs Rest」構造に変質。Magnificent 7 (AAPL, MSFT, GOOGL, AMZN, NVDA, META, TSLA) の集中度が VUG を支配 → 従来の value/growth rotation が機能不全。

### R3 再定義の試み: SPY vs RSP

仮説: 「mega-cap vs rest」を SPY (cap-weighted) vs RSP (equal-weight) spread で再定義。

結果: **有意に悪化** (paired t = -1.96, p = 0.0495)
- R3_new standalone Sub2: +0.32 (R3_old +0.25 と同等)
- Portfolio 統合: v33 > v34(R3_new) (Sub2 -0.19)
- MN との相関: R3_new +0.41, R3_old +0.48 → **R3 は MN と redundant**

### WF 検証

Train (2010-2018) vs Test (2019-2026) の LOO drag detection:
```
R3: Train Δ = +0.000, Test Δ = +0.215 → "Test-only drag"
VC: Train Δ = +2.488, Test Δ = +0.218 → "Both drag" (OOS-justified)
```

R3 は Train で drag 検出されず → **look-ahead risk**。しかし:
1. Train 前半は warmup (vol-weight Δ=0.000 は warmup artifact)
2. Year-rolling で 2022 以降 consistent drag (+0.02〜+0.05)
3. MN との redundancy は structural (相関 +0.48)

### 統計検証

Sub2 paired t-test:
```
t = +2.33, p = 0.020 (95% 有意)
Bootstrap P(v33 > v32) = 98% (21/63/126d 全 horizon)
```

### 最終判断

**R3 は暫定除外** (`include_r3=False` flag、True で復帰可能)。

判断根拠:
1. Sub2 で統計有意 (p = 0.020)
2. VTV-VUG correlation structural break (0.92 → 0.47)
3. MN との redundancy (+0.48 corr)
4. 再定義 (SPY-RSP) も失敗

**ChatGPT review**: 「暫定採用してよいが恒久削除には一歩足りない」「transition regime であり permanent structural break とは断定できない」

---

*次章: CTA — Multi-Asset Trend Following と PnL Averaging バグ*
