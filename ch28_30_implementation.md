# Part V: 実装と運用

# Chapter 28: safe_backtest.py — Look-Ahead の構造的防止

## 28.1 設計思想: Convention → Structure

従来: programmer が `lev[:-1]` を書き忘れなければ安全 (convention-based)
本書: 忘れても安全、間違えたら error (structure-based)

### API 設計

```python
# vol_target_safe(pnl, target=0.10)
# 内部で shift(1) を強制。呼び出し側が shift を忘れることは構造的に不可能。

# shift_forward(arr, n_days)
# n_days < 1 → ValueError("Using today's signal for today's PnL is look-ahead.")

# build_position_held(positions, hold_days)
# POSITION averaging のみ。PnL averaging を docstring で明示的に禁止。

# compute_ls_pnl_safe(positions_held, returns, shift=1)
# shift < 1 → ValueError
```

## 28.2 portfolio.py への適用

Before (convention):
```python
lev_safe = np.concatenate([[1.0], lev[:-1]])  # 忘れやすい
out = sh(arr, 2)  # sh(arr, 0) でも動く
```

After (structure):
```python
scaled = vol_target_safe(pnl)       # shift 内蔵
out = shift_forward(arr, 2)         # shift(0) → ValueError!
```

**8 箇所の vol-target + 8 箇所の overlay shift を置換**。数値検証: ±0.001 以内で一致。

---

# Chapter 29: monitor.py / execute.py — 実運用インフラ

## 29.1 monitor.py: Composite Alpha Decay Gate

### 設計 (ChatGPT review 反映)

Single-metric alert は false positive が多い → **composite gate**:

```
WARN: 3M Sharpe < 1.0  AND  (DD > 1.25× expected  OR  z-score < -1.5)
STOP: 3M Sharpe < 0.0  AND  (DD > 2.0× expected  OR  z-score < -2.0)
```

### 使い方

```bash
python monitor.py                      # 状態レポート
python monitor.py --add-pnl 0.0012     # 実測 PnL 追加
python monitor.py --compare-backtest   # live vs backtest 比較
```

## 29.2 execute.py: Hard/Soft Stop 分離

### Hard Stop (自動停止 — system anomaly)
- signal_freshness (日付不一致)
- data_gap (3 営業日超)
- daily_pnl_worst (-2% = 2× backtest worst-day)

### Soft Stop (review 停止 — performance deterioration)
- cumulative_dd (-5%)
- sharpe_divergence (21d Sharpe が expected から 2σ 乖離)
- `--override-soft-stop` で human decision

### Integration Test

8 scenario × pass/fail 検証:
```
Scenario 1: No live data → all pass ✅
Scenario 2: 30 days normal → no false positive ✅
Scenario 3: -2.5% daily → Hard Stop ✅
Scenario 4: -6% DD → Soft Stop ✅
Scenario 5: Soft Stop override ✅
Scenario 6: Data gap → Hard Stop ✅
All 8/8 PASS.
```

---

# Chapter 30: Paper Trading から Live Deployment へ

## 30.1 PAPER_TRADING.md Spec

### 4-Phase Deployment

| Phase | 期間 | Capital | Leverage |
|---|---|---|---|
| 1. Paper | 4-8 週間 | ¥0 | — |
| 2. Small live | 2-3 ヶ月 | ¥1-3M | 1.0x |
| 3. Scale-up | 3-6 ヶ月 | ¥5-10M | 1.5x |
| 4. Full live | 継続 | ¥10M+ | 2.0x |

### Paper 合格条件 (4 条件中 3 つ以上)

**A. 実行品質**: rejection rate < 1%, fill completeness > 95%, slippage < 1.25× assumption
**B. PnL 整合性**: daily corr > 0, 20d 累積 -2σ 以内, 3d 連続 3σ 乖離なし
**C. リスク整合性**: DD < 1.5× expected, worst-day < 2× expected
**D. 簡易成績**: 20-40d annualized Sharpe > 1.0

### ChatGPT Review Summary

3 回の ChatGPT 相談を通じて:
1. R3 削除: 「暫定 OK、DSR/PBO 必要」 → 全て pass
2. Research phase 完了: 「もう作るフェーズじゃない」 → 同意
3. Execution validation: Paper → Small → Scale → Full の 4 phase
4. 「Regime gating は live data 後」 → 同意 (vol cap/conditional は不採用)

## 30.2 daily_flow.sh

日次運用の chain script:
```bash
./daily_flow.sh                # full flow with prompts
./daily_flow.sh --auto         # cron 用
./daily_flow.sh --add-pnl 0.0015  # PnL 追加のみ
./daily_flow.sh --monitor      # 状態確認のみ
```

## 30.3 Normal Regression vs True Decay

### Level 1: Noise (継続)
- 1M rolling Sharpe 低下、DD 想定内、execution OK

### Level 2: Review
- 3M rolling Sharpe < 1、component の半数超 underperform

### Level 3: Decay 疑い (停止)
- 3M rolling Sharpe < 0、execution 調整後も alpha 消失、component 横断で悪化

**判定軸**: execution-adjusted か? component 局所か横断か? regime 限定か全 regime か?

---

*Part VI 開始: Chapter 31 — 開発ドキュメンタリー*
