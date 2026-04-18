# Chapter 13: CTA — Multi-Asset Trend Following と PnL Averaging の罠

本章は本書で最も重要な **negative result** の一つを報告する。「turnover 削減のための hold overlap」が **PnL averaging** という methodological error に陥り、Sharpe を人工的に 4.6 倍に膨張させたメカニズムを解剖する。

## 13.1 CTA の学術的根拠

### Time Series Momentum

Moskowitz, Ooi & Pedersen (2012) "Time Series Momentum": 58 の先物市場で、過去 12 ヶ月のリターンが正なら long、負なら short という単純な戦略が、1965-2009 で年率 Sharpe 1.0 を達成。

Hurst, Ooi & Pedersen (2017): 1880-2016 の 137 年間で trend-following が有効であることを確認。

### 本戦略の CTA 実装

9 資産の AbsMom 252d composite:
```python
assets = {TLT, UUP, DBC, GLD, XLE, VTV, SPY, EFA, EEM}
signal = sig_absolute_momentum(close, lookback=252)
# 各資産: 252d return > 0 → long 1.0, < 0 → short -1.0
# 9 資産の daily return を equal-weight 平均
```

Standalone Sub2 Sharpe: +1.12

## 13.2 「Turnover 削減」の着想と実装

### 背景
Cost stress test (Chapter 27) で CTA の daily turnover が問題視された。252 日 lookback の AbsMom signal は「ゆっくり」変化するのに、daily rebalance は「不必要な turnover」を生むのではないか。

MN (Sector L/S) でも同じ仮説が立てられ、hold 10d overlap の試みが行われた。

### 実装 (誤り)

```python
# MN/CTA の hold overlap (誤った実装)
mn_held = np.zeros(n)
for lag in range(hold_days):
    mn_held[lag:] += mn_ret[:n - lag] / hold_days
```

これは mn_ret (日次の **realized PnL**) の **移動平均** である。

### 衝撃的な結果

```
MN standalone:  Sharpe 0.58 → 2.69 (4.6x!)
CTA standalone: Sharpe 1.12 → 3.05 (2.7x!)
Portfolio:      Sub2 +1.39 Sharpe 改善 (6.29 → 7.68)
```

統計検証: t = +5.41, p < 10⁻⁶, Bootstrap P(new > old) = 100%。

**一見すると完璧な改善。** 全 8 年で改善、return +2.15%/yr 増加、全ての検定に pass。

## 13.3 T7 Audit: バグの発見

ChatGPT の look-ahead 監査テスト計画 (Chapter 4 参照) に基づき、T7 テストを実施:

### PnL averaging ≠ Position holding

**正しい hold overlap** (JP_LS 方式):
```python
# POSITION を overlap 平均 → その後に return を掛ける
pos_held[lag:] += pos[:n-lag] / hold_days  # POSITION averaging
pnl[1:] = (pos_held[:-1] * ret_arr[1:]).sum(axis=1)
```

**誤った hold overlap** (MN/CTA の実装):
```python
# REALIZED PnL を直接平均
mn_held[lag:] += mn_ret[:n-lag] / hold_days  # PnL averaging!
```

### メカニズムの解剖

PnL の 10 日移動平均は:
```
mn_held[d] = (mn_ret[d] + mn_ret[d-1] + ... + mn_ret[d-9]) / 10
```

これは単なる **signal smoothing filter** であり、position holding ではない。

| Metric | Daily (baseline) | PnL averaging (誤) | Position hold (正) |
|---|---|---|---|
| Mean return | **7.22%** | **7.23%** | 4.53% |
| Volatility | **17.51%** | **5.30%** | 17.50% |
| Sharpe (raw) | +0.41 | +1.36 | +0.26 |

**Mean return はほぼ同一** (7.22% vs 7.23%) だが **vol が 3.3x 低下** (17.5% → 5.3%)。これは移動平均フィルタの smoothing 効果。

### Vol-targeting との致命的相互作用

Vol-targeting は低 vol を検出して leverage を増やす:
```
vol_target(10%) / actual_vol(5.3%) = 1.89x leverage
```

結果: Return が 7.23% × 1.89 = 13.7% に amplify、Vol は target の 10% に normalize → Sharpe = 13.7 / 10 = 1.37。

**つまり**: PnL smoothing → vol 低下 → vol-targeting が leverage 増 → Sharpe 人工的膨張。

### 正しい実装との比較

| Method | Sub2 Sharpe | 判定 |
|---|---|---|
| Daily rebalance (baseline) | **+0.581** | ✅ 最良 |
| PnL averaging (10d) | +2.689 | ❌ **artifact** |
| Position hold (10d rebalance) | +0.391 | ❌ baseline より悪い |
| Position overlap (JP_LS style) | +0.363 | ❌ baseline より悪い |

**正しい hold 実装は daily baseline より悪い** → MN の daily rebalance は**必要な** turnover (ranking の微小変化を追従)。

## 13.4 なぜ検出が遅れたか

1. **Per-year test 全年改善** → per-year は PnL averaging でも consistent に見える
2. **t-stat = +5.41** → 統計検定も「有意」と判定 (smoothing による low vol が t-stat を膨張)
3. **Bootstrap 100%** → 同上
4. **直感的に合理的** → 「slow signal に daily rebalance は不要」は一見正しい

### 検出できた理由

ChatGPT の look-ahead テスト計画 (T7) で **明示的に hold overlap の timing 整合性を疑った**:
> 「mn_held[d] に mn_ret[d] (当日 PnL) が含まれる → 当日 PnL を知った上で当日の position weight を作っている」

実際にはこれは look-ahead ではなく **methodology error** だったが、**疑う姿勢** が発見につながった。

## 13.5 教訓

### For practitioners

1. **PnL smoothing + vol-targeting = Sharpe inflation trap**。どんなに美しい backtest 結果でも、underlying のメカニズムを分解して確認すべき。

2. **「Mean return 不変 + vol 低下 = 本物の改善」ではない**。Vol 低下が trading の改善 (turnover 減) ではなく、signal processing の artifact である可能性。

3. **正しい hold overlap は POSITION averaging**。PnL averaging は invalid。JP_LS/AU_LS の実装を参照。

### For researchers

4. **統計検定は methodology error を検出できない**。t-test, bootstrap, per-year 全てが pass しても、underlying の数学的構造が間違っていれば結果は spurious。

5. **Replication の重要性**: 同じ結果を**別の実装** (position hold) で再現できないなら、original 実装に問題がある。

## 13.6 Vol-parity テスト (A3)

CTA の 9 資産を vol-parity (各資産を同 risk に normalize) で配分する variant もテスト。

結果: Sub2 +0.977 → **baseline +1.12 より悪化**。

理由: 高 vol 資産 (DBC, XLE) の momentum signal が強い → vol normalize すると alpha も normalize される。Equal-weight averaging が CTA には最適。

---

*次章: MN — Sector Market-Neutral と XLE_T — Energy TSMOM*
