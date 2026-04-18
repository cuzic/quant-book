# Chapter 14: MN — Sector Market-Neutral L3/S3

## 14.1 設計

11 US sector ETF (XLK, XLF, XLE, ...) の 252-21d momentum で ranking。Top 3 を long、Bottom 3 を short。

**Market-neutral 設計**: Long 3 + Short 3 → net exposure ≈ 0。市場全体の方向に依存しない alpha。

## 14.2 v32 per-component overlay: NO_OVERLAY に分類

MN は sector-neutral → US equity option overlay (ORATS) は不要。v32 の grid search で NO_OVERLAY が最適 (Δ +0.33 Sharpe vs full overlay)。

## 14.3 PnL averaging バグ (Chapter 13 参照)

MN でも CTA と同じ PnL averaging バグが発生。Hold 10d の試みは:
```
Standalone: 0.58 → 2.69 (PnL averaging, 偽)
Position hold: 0.58 → 0.39 (正しい実装、baseline より悪い)
```

**Daily rebalance が正しい**。Ranking は 252-21d で slow だが、daily の微小変化を追従する value がある。

## 14.4 Ablation

```
Drop MN: Sub2 +0.083 (portfolio 改善 → MN は mild drag)
```

MN は standalone Sharpe +0.55 だが、R3 との相関 +0.484 → **R3 削除後も MN は portfolio で marginal**。

## 14.5 Negative month 分析

2021-07: MN = -4.57% (最大 drag)。Delta variant 恐怖で sector rotation が急反転 → L3/S3 が逆行。

---

# Chapter 15: XLE_T — Energy TSMOM

## 15.1 設計

XLE (Energy Select Sector ETF) の TSMOM 252/21d。単一 sector、単一 asset の directional bet。

## 15.2 v32 overlay: VIX_ONLY

Energy sector は ORATS option overlay の影響を受けにくい (commodity-driven)。VIX overlay のみが有効:
- VIX < 17: energy momentum 1.2x (calm → trend continuation 期待)
- VIX > 25: energy momentum 0.8x (stress → trend reversal リスク)

## 15.3 パフォーマンス

```
Standalone Sub2: +0.50
IV weight: ~1.7% (最小 component)
Ablation: Drop XLE_T → Sub2 +0.018 (ほぼ neutral)
```

Portfolio への寄与は最小。しかし crisis 3 group の一部として diversification に貢献。

---

*次章: T4b — Wavelet Multi-Resolution Signal*
