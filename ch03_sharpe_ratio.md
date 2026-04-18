# Chapter 3: Sharpe Ratio の正しい理解と落とし穴

## 3.1 定義と年率化

Sharpe Ratio (Sharpe 1966, 1994) の定義:

$$SR = \frac{E[R_p - R_f]}{\sigma(R_p - R_f)}$$

日次データからの年率化:

$$SR_{annual} = SR_{daily} \times \sqrt{252}$$

**本ポートフォリオ v35 の実測値**:
```
Daily mean:     +0.065%
Daily std:       0.150%
Daily Sharpe:    0.433
Annual Sharpe:   0.433 × √252 = 6.87
```

## 3.2 年率化の暗黙の仮定

√252 による年率化は、日次リターンが **i.i.d. (独立同分布)** であることを仮定する。実際のリターンは:

1. **自己相関**: Momentum 戦略は正の自己相関、MR 戦略は負の自己相関を持つ
2. **条件付き分散変動**: GARCH 効果で vol clustering が存在
3. **Non-normality**: 歪度 (skew +1.28) と尖度 (kurtosis +8.8)

これらの違反がある場合、年率 Sharpe は **過大または過小評価** される。

**Lo (2002) の補正**: 自己相関を考慮した Sharpe の standard error:

$$SE(SR) \approx \sqrt{\frac{1 + 2\sum_{k=1}^{q} \rho_k}{T}}$$

本ポートフォリオの日次自己相関 ρ₁ ≈ +0.05 (overlay の influence) → 補正は小さい。

## 3.3 Non-normality 下の Sharpe: Deflated Sharpe Ratio

Bailey & López de Prado (2014) の Deflated Sharpe Ratio (DSR) は、multiple testing と non-normality の両方を補正:

$$DSR = \Phi\left[\frac{(SR_{obs} - SR_{exp})\sqrt{T-1}}{\sqrt{1 - \gamma_3 \cdot SR + \frac{\gamma_4 - 1}{4} SR^2}}\right]$$

ここで:
- $SR_{exp}$: N 回の trial で期待される最大 Sharpe (Gumbel 近似)
- $\gamma_3$: skewness
- $\gamma_4$: kurtosis

**本ポートフォリオの DSR 検証結果** (Chapter 26 の詳細):

| N_trials | SR_expected | DSR z-score | 判定 |
|---|---|---|---|
| 100 (conservative) | 1.43 | +13.05 | 有意 |
| 680 (medium) | 1.78 | +12.09 | 有意 |
| 5,000 (high) | 2.08 | +11.24 | 有意 |
| 30,000 (extreme) | 2.33 | +10.56 | 有意 |

**どの trial 数仮定でも z > 10** — 観測 Sharpe 6.87 は偶然の産物ではない。

## 3.4 Sortino Ratio と下方リスク

Sortino Ratio (Sortino & Price 1994):

$$Sortino = \frac{R_p - R_f}{\sigma_{downside}}$$

本ポートフォリオ:
```
Sharpe:  +6.87 (symmetric vol)
Sortino: +9.89 (downside vol only)
```

Sortino > Sharpe は正の歪度 (skew +1.28) を反映: 上方変動が下方変動より大きい。

## 3.5 Sharpe 6+ は信じられるか?

### 学術的ベンチマーク
- 平均的な hedge fund: Sharpe 0.5-1.5
- Top-tier systematic fund: Sharpe 2-3
- Renaissance Medallion (推定): Sharpe 6-7

### 本ポートフォリオが高 Sharpe を達成する構造的理由

1. **多数の独立 alpha** (17 components, corr ≈ 0.05-0.10)
2. **Vol targeting per component** (各 10% → portfolio vol 2.4%)
3. **5 層 overlay** (timing alpha が Sharpe の 42% — T6 shuffle test)
4. **低 vol** (2.4%/yr → modest return 15% でも Sharpe = 15/2.4 = 6.3)

### 主なリスク
- **Live degradation**: Cost drag (-0.8), overfit (-0.4), regime decay (-0.3) → Live 推定 4.2±1.0
- **Correlation spike**: DR 分析で「crisis で DR 上昇」と確認済みだが、未観測レジームのリスク
- **ORATS data 依存**: T1 shift test で ORATS overlay freshness が Sharpe の 0.6 を占めると判明

## 3.6 Calmar Ratio と MaxDD

Calmar Ratio = Return / |MaxDD|:

```
v35: Return +17.14%, MaxDD ≈ -0.9%
Calmar = 17.14 / 0.9 = 19.0
```

これは異常に高い。MaxDD -0.9% は「月次で 2 ヶ月しか negative がない」ことの反映。

**重要な注意**: MaxDD は in-sample metric であり、live での DD は必ず in-sample DD を超える。PAPER_TRADING.md では MaxDD tolerance を -5% に設定 (backtest -0.9% の 5.5 倍のマージン)。

---

*次章: バックテストの罠 — look-ahead, overfitting, survivorship bias*
