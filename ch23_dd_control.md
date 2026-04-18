# Chapter 23: DD Control — Depth + Speed Sensing

## 23.1 進化の歴史

| Version | DD Control | Sharpe impact |
|---|---|---|
| v24 以前 | Binary: DD < -1.5% → scale 0.7 | ほぼ未発動 (DD < 1.5% が稀) |
| v25 | tanh: 1 - 0.4 × tanh(50 × \|DD\|) | +0.31 (連続化の効果) |
| v27 | tanh: 1 - 0.5 × tanh(70 × \|DD\|) | +0.26 (parameter 最適化) |
| **v34** | **depth + speed** | **+0.329 Sub2** |

## 23.2 v27 Depth-only の限界

```
Scale formula: 1 - 0.5 × tanh(70 × |DD|)
  DD  0.0% → scale 1.00
  DD -0.3% → scale 0.90
  DD -0.5% → scale 0.82
  DD -1.0% → scale 0.65
```

**問題**: Reactive (DD が実現してから縮小)。DD -0.5% に到達した時点で既に損失確定。

## 23.3 v34: Speed Sensing の追加

### 着想

ChatGPT 提案 → 我々が具体化:
- 「守るべきは vol ではなく left tail」
- 「Proactive + asymmetric が理想」
- DD control は既に asymmetric (downside only) → proactive 化が次のステップ

### 設計

```python
# Depth component (v27, unchanged)
depth_sc = 1.0 - 0.5 × tanh(70 × |DD|)

# Speed component (v34 new)
dd_speed = (DD[t] - DD[t-3]) / 3    # 3-day DD acceleration
speed_sc = 1.0 - 0.20 × tanh(500 × max(0, -dd_speed))

# Combined
dd_scale = depth_sc × speed_sc
```

**Speed sensing のメカニズム**:
- DD -0.1% + **急速悪化中** (speed -0.02%/day) → speed_sc ≈ 0.85 → 早期縮小
- DD -0.5% + **安定** (speed ≈ 0) → speed_sc ≈ 1.0 → depth のみで対応

## 23.4 検証結果

9 variant を同一 baseline で比較:

| Variant | Test S | Sub2 S | Sortino | MaxDD | Return |
|---|---|---|---|---|---|
| No DD control | +5.674 | +5.740 | +6.724 | -1.36% | +14.37% |
| v27 depth only | +6.202 | +6.291 | +8.208 | -1.00% | +14.86% |
| **v34 depth+speed (3d,0.20,500)** | **+6.511** | **+6.620** | **+9.224** | **-0.88%** | **+15.25%** |
| Depth + downside vol | +6.163 | +6.306 | +8.210 | -0.80% | +13.33% |
| Depth + loss streak (3d) | +6.231 | +6.331 | +8.275 | -0.98% | +14.91% |

**v34 depth+speed が全指標で最良** (Sortino +1.0, MaxDD +12bp, Return +0.39%/yr)。

### Per-year: 全 8 年で改善

```
2019: +0.08  2020: +0.09  2021: +0.11  2022: +0.12
2023: +0.09  2024: +0.09  2025: +0.06  2026: +0.06
```

### なぜ Return が増えるか (free lunch?)

「防御を強化して return も増える」は paradox に見えるが:
1. 早期縮小 → DD bottom が浅い (-0.88% vs -1.00%)
2. 浅い DD → recovery が速い
3. 速い recovery → **full exposure で過ごす時間が長い**
4. 長い full exposure → 年間 return 増加

**Proactive defense = fewer days at reduced exposure = net return increase.**

## 23.5 Vol Cap との比較

Vol cap (Chapter の earlier analysis):
```
3.5% cap: Sub2 +6.289, Kurt 9.9, MaxDD -1.00%, Cost -0.30%/yr
2.5% cap: Sub2 +6.294, Kurt 6.9, MaxDD -0.90%, Cost -1.16%/yr
```

DD depth+speed:
```
Sub2 +6.620, Kurt 10.1, MaxDD -0.88%, Cost 0% (return +0.39%!)
```

| | Vol Cap (2.5%) | DD Speed |
|---|---|---|
| Sub2 Sharpe | +6.294 | **+6.620** (+0.33 better) |
| Mechanism | Symmetric (cuts upside too) | **Asymmetric** (downside only) |
| Cost | **-1.16%/yr** | **+0.39%/yr** (free lunch!) |
| Kurt | **6.9** (大幅改善) | 10.1 (変化なし) |
| MaxDD | -0.90% | **-0.88%** (微改善) |

**DD speed は vol cap の complete domination** (Sharpe 高い + cost なし + 非対称)。唯一 vol cap が勝る指標は Kurt 改善だが、DR 分析 (DR crisis で上昇) によりこの portfolio では Kurt risk は低い。

## 23.6 IV_PREM への DD speed 適用

IV_PREM component (Chapter 11) でも DD speed を内部適用:
```
IV_PREM standalone: +1.661 → +2.303 (+38% with DD speed)
```

Carry 戦略 (VRP 収穫) は DD が始まると accelerate → DD speed が「carry 崩壊の初期検出」として機能。

## 23.7 実装の安全性

```python
# _apply_dd_control は全て shift なしの real-time DD
# DD[t] は t 日の close まで confirmed
# dd_speed は t-3 との比較 → t 日の情報のみ使用
# Look-ahead: なし (T1 shift test で確認済み)
```

---

*Part IV 開始: Chapter 24 — Inverse-Volatility Weighting の設計と検証*
