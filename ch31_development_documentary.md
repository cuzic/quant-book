# Chapter 31: v1 (Sharpe 0.86) から v35 (Sharpe 6.87) への旅

## 31.1 バージョン進化の全軌跡

```
v1:  0.86  MR+GTAA+LV (3 strategies, equal weight)
v13: 2.54  + J3 overlay + VIX overlay (overlay 時代の始まり)
v18: 4.15  + ORATS overlays + Hybrid shift (EMA speed)
v20: 4.64  + EMA speed optimization (PC14, TS42, IVP4)
v23: 5.02  + JP_LS pnl-stacking 7-factor
v24: 5.06  + FLOW factors (FL_1m, FL_1w, BLG)
v25: 5.40  + smooth tanh DD scaling
v26: 5.70  + IC-weighted Asymm z-sum
v27: 5.96  + tanh DD parameter optimization
v28: 5.92  + JP_LS cost reduction (TopN 30, Hold 21d)
v29: 5.97  + dynamic learned weights
v30: 5.98  + AU_LS (ASIC short positions)
v31: 6.12  + AU_LS VIX-only overlay
v32: 6.35  + per-component overlay subset optimization
v33: 6.20  - R3 removed (structural break) + AU_LS UTF-16 fix
v34: 6.51  + DD depth+speed sensing
v35: 6.87  + IV_PREM (VRP+Skew long-only, ORATS timing, DD speed)
```

### 成長率の分析

| Phase | Versions | Sharpe | Δ | 主な改善軸 |
|---|---|---|---|---|
| 基盤構築 | v1→v13 | 0.86→2.54 | +1.68 | Component 追加 + overlay |
| Overlay 最適化 | v13→v20 | 2.54→4.64 | +2.10 | ORATS + EMA speed |
| JP alpha | v20→v28 | 4.64→5.92 | +1.28 | JP_LS + DD control |
| 精緻化 | v28→v32 | 5.92→6.35 | +0.43 | per-component overlay |
| 洗練 | v32→v35 | 6.35→6.87 | +0.52 | R3 削除 + DD speed + IV_PREM |

**Diminishing returns の法則**: 初期 (+1.68/phase) → 後期 (+0.43-0.52/phase)。v35 以降の改善余地は small。

## 31.2 転換点となった key decisions

### v13: Overlay の発明 (Sharpe 2.5x)
「component の return にマーケット全体の regime information を乗せる」という発想。J3 + VIX の 2 overlay で Sharpe 0.86 → 2.54。

### v23: PnL-stacking (Sharpe +0.38)
Rank-average (0.86) → PnL-stacking (2.55) で JP_LS が 3x 改善。「各 factor の独立 L/S → 加重結合」が rank average の情報圧縮問題を解決。

### v32: Per-component overlay (Sharpe +0.37)
「全 component に同じ overlay を適用」→「component ごとに最適 overlay subset を選択」。US equity → full overlay、rates → NO_OVERLAY、foreign → VIX_ONLY。

### v34: DD depth+speed (Sharpe +0.33)
DD control の proactive 化。「DD が深くなってから縮小」→「DD が加速し始めたら縮小」。Return +0.39%/yr の free lunch。

### v35: IV_PREM (Sharpe +0.33)
125+ configs の大規模探索から「Long-only VRP+Skew, VRP-proportional, ORATS timing, DD speed」の champion を発見。t = +6.26 (p < 10⁻⁶)。

## 31.3 コードの変遷

```
portfolio_v11.py: 1580 行 (v32 開始時)
→ refactor: -293 行 (死コード削除、外部化)
→ v33 R3 削除: -15 行
→ v34 DD speed: +20 行
→ v35 IV_PREM: +100 行
→ safe_backtest.py: +100 行 (新規)

最終: portfolio.py 1341 行 + safe_backtest.py 100 行
     + monitor.py 300 行 + execute.py 250 行
```

ファイル数: 159 → 25 (top-level, -84%)。

---

*次章: 失敗した戦略たち*
