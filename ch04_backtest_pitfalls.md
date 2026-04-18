# Chapter 4: バックテストの罠 — Look-Ahead, Overfitting, Survivorship Bias

## 4.1 なぜバックテストは嘘をつくのか

López de Prado (2018) は "most published backtests are false" と主張する。主な原因:

1. **Look-ahead bias**: 未来データの混入
2. **Selection bias**: 多数の戦略を試して最良を報告 (multiple testing)
3. **Survivorship bias**: 生存企業のみのデータ
4. **Overfitting**: パラメータが特定期間に過剰適合

本章では、v35 ポートフォリオの開発過程で**実際に発見したバグと対策**を報告する。

## 4.2 Look-Ahead Bias: 実際に検出した事例

### Case 1: DD brake same-day look-ahead (開発初期、CLAUDE.md 記録)

初期の DD brake 実装:
```python
if dd_today < -0.015:
    scale = 0.7  # 当日の DD で当日の position を縮小
```

**問題**: `dd_today` は当日終値で確定する → 当日の return を見てから当日の scale を決定 = look-ahead。
**影響**: Sharpe 30x の膨張 (0.86 → 20+)。
**修正**: DD control を翌日適用 → shift(1) 強制。

### Case 2: T7 PnL averaging Sharpe inflation (本セッション発見)

MN/CTA の hold overlap 実装:
```python
mn_held[lag:] += mn_ret[:n - lag] / hold_days  # PnL 平均
```

**問題**: `mn_ret[d]` (当日の realized PnL) を `mn_held[d]` に含む → position と return が同日。
**しかし**: これは look-ahead ではなく **methodological error** — PnL averaging ≠ position holding。
**影響**: 
- Mean return 不変 (7.22% → 7.23%)
- Vol が 3.3x 低下 (17.5% → 5.3%) — 移動平均フィルタの smoothing 効果
- Vol-targeting が低 vol を検出 → leverage 3.3x 増 → Sharpe 人工的膨張
- Standalone: 0.58 → 2.69 (4.6x)
- Portfolio: +1.39 Sharpe (v34 hold overlap)

**修正**: Revert to daily rebalance。正しい position-hold (10d rebalance) は daily より悪い (+0.39 vs +0.58)。

**教訓**: PnL smoothing + vol-targeting = **Sharpe inflation trap**。JP_LS/AU_LS の hold overlap は POSITION averaging (正しい) だが、MN/CTA の daily PnL に同パターンは invalid。

### Case 3: T1 Shift テストの結果

全 signal/weight を一律 +1 day 遅延させた結果:

| Layer | ΔSub2 | 判定 |
|---|---|---|
| ORATS overlay | -0.611 | freshness premium (not look-ahead) |
| J3 overlay | -0.005 | clean |
| VIX overlay | -0.049 | clean |
| Asymm overlay | -0.034 | clean |
| inv-vol weight | -0.002 | clean |
| ALL combined | -0.773 | ORATS 主因 |

**結論**: 主要な look-ahead は存在しない。ORATS の -0.611 は data freshness の value (ORATS EOD は 15 分遅延で当日中に利用可能 → shift=1 は valid)。

## 4.3 構造的防止: safe_backtest.py

本書で提案する look-ahead 防止の設計パターン:

### 原則: Convention-based → Structure-based

```python
# BEFORE (convention): programmer が忘れたら look-ahead
lev_safe = np.concatenate([[1.0], lev[:-1]])  # 忘れやすい
out = sh(arr, 2)  # sh(arr, 0) でも動く → 危険

# AFTER (structure): 忘れても安全、間違えたら error
from safe_backtest import vol_target_safe, shift_forward
scaled = vol_target_safe(pnl)         # shift は内部で強制
out = shift_forward(arr, 2)           # shift_forward(arr, 0) → ValueError!
```

### vol_target_safe()
- Leverage は **必ず前日の vol** で計算
- `lev_safe[1:] = lev[:-1]` を内部で強制
- 呼び出し側が shift を忘れることが**構造的に不可能**

### shift_forward(arr, n_days)
- `n_days < 1` で **ValueError** を raise
- 例外メッセージ: "Using today's signal for today's PnL is look-ahead."
- 正当な shift=0 が必要な場合は `shift_forward_allow_zero()` を使用 (justification 必須)

## 4.4 Multiple Testing と Selection Bias

### 本書の探索空間

本セッションで試行した config 数:
- Ablation: 17 components × 10 overlays = 170
- Weighting: 10 schemes
- IV Premium: 72 (grid) + 36 (full grid) + 17 (combos) = 125
- Signal variants: ~50
- **合計: ~400+ configurations**

Harvey, Liu & Zhu (2016) の multiple testing correction: t-stat > 3.0 を要求。

**v35 の主要 test statistics**:
- R3 削除: t = +2.33 (95% marginal, 99% 未達) → **要注意**
- DD depth+speed: t = +5.41 → **strong** (但し v34 hold bug で revert 後の v34 で)
- IV Premium: t = +6.26 → **very strong**

### Probability of Backtest Overfitting (PBO)

Bailey et al. (2014, 2017) の CSCV (Combinatorially Symmetric Cross-Validation):

手順:
1. Test 期間を S partitions に分割
2. C(S, S/2) 通りの IS/OOS 分割を列挙
3. 各分割で: IS 最良戦略が OOS で中央値以下 → overfitting

**本ポートフォリオの PBO**: 18.6% (8 partitions, 100 combos)
- < 20%: Low overfit risk (López de Prado threshold)
- 20-50%: Moderate
- > 50%: High (backtest untrustworthy)

## 4.5 Survivorship Bias

### Universe 構築時のリスク
- OHLCV cache: yfinance でダウンロード → 上場廃止銘柄はデータ不在の可能性
- IB CFD universe: 現時点の CFD 対象銘柄で固定 → 過去に CFD 対象だったが現在非対象の銘柄を含まない

### 対策
- JP_LS universe: JPX 上場かつ IB CFD 対象かつ volume > 100K → 比較的 survivor bias が小さい (909 銘柄の大部分は生存)
- 但し、JP_LS の alpha source (信用倍率) は上場維持銘柄にのみ適用 → bias の影響は限定的

## 4.6 ランダム破壊テスト (T6)

正常な戦略なら、未来情報を壊すと成績も壊れる:

| 破壊方法 | Sub2 Sharpe | 判定 |
|---|---|---|
| Baseline (正常) | +6.29 | — |
| Shuffle component returns | +2.64 (42%) | **Overlay timing alpha が残存** |
| Shift returns +5 days | +4.17 (66%) | Autocorrelation + overlay |
| Reverse time | -0.12 (0%) | **完全崩壊 → bug なし** |

**重要な発見**: Shuffle test で Sharpe 2.64 が残存 = **overlay layer 単独で Sharpe ~2.6 の alpha source**。これは component alpha とは独立した timing alpha であり、v35 Sharpe の 42% を占める。

## 4.7 本書のバックテストに対する honest assessment

| リスク | 対策 | 残存リスク |
|---|---|---|
| Look-ahead | safe_backtest.py + T1 shift test | ORATS freshness 0.6 Sharpe 依存 |
| Multiple testing | DSR z>10, PBO 18.6% | 400+ configs 探索の influence |
| Survivorship | Large universe, liquid filter | 完全除去は不可能 |
| Overfitting | WF, per-year stability | R3 削除は Sub2-specific か? |
| Execution gap | 未実測 | **最大のリスク** |

**最大の未知**: Live execution cost と slippage。Paper trading で実測するまで、backtest Sharpe 6.87 の信頼性は **conditional** である。

---

*Part II 開始: Chapter 5 — MR: Mean Reversion (CRSI + Profit Target)*
