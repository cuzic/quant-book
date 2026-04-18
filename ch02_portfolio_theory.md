# Chapter 2: なぜ分散が効くのか — 相関とポートフォリオ理論

## 2.1 Markowitz の Mean-Variance から始める

Modern Portfolio Theory (Markowitz 1952) の核心は:

$$\sigma_p^2 = \sum_i w_i^2 \sigma_i^2 + 2 \sum_{i<j} w_i w_j \sigma_i \sigma_j \rho_{ij}$$

N 個の等加重 (w = 1/N) コンポーネントで相関 ρ が一様の場合:

$$\sigma_p^2 = \frac{\sigma^2}{N} + \frac{N-1}{N} \rho \sigma^2$$

- ρ = 0 (完全独立): σ_p = σ/√N → N=16 で 75% のリスク削減
- ρ = 0.5 (中程度相関): σ_p ≈ σ × √(0.56) → 25% のリスク削減のみ

**本ポートフォリオの実測**: v35 の 17 components は平均相関 ≈ 0.05-0.10。これにより portfolio vol 2.4% (個別 component vol 10% の 1/4) を達成。

## 2.2 なぜ 1/N が理論最適化に勝つことがあるのか

DeMiguel, Garlappi & Uppal (2009) "Optimal Versus Naive Diversification" は衝撃的な結果を報告した: 14 の最適化手法のうち、**単純な 1/N 等加重が majority を outperform** した。

理由: 推定誤差 (estimation error)。共分散行列の推定には N(N+1)/2 個のパラメータが必要で、サンプルサイズが不十分な場合、最適化の利益よりも推定誤差の害が上回る。

**本書での検証** (Chapter 24):
| Weighting | Sub2 Sharpe |
|---|---|
| Inverse-Volatility (v35) | +6.95 |
| 1/N Equal Weight | +2.53 |
| HRP (López de Prado) | +2.84 |
| Sharpe-Weighted | +2.12 |

本ポートフォリオでは **inverse-volatility が圧勝** (1/N の 2.7 倍)。DeMiguel の結論が覆る理由は:
1. Component vol の分散が大きい (MR 22.9% vs XLE_T 1.7%)
2. Vol 推定は共分散推定より安定 (N 個のパラメータ vs N² 個)
3. 504 日の rolling window が十分なサンプルサイズを提供

## 2.3 Diversification Ratio と危機時の行動

Diversification Ratio (DR) は Choueifaty & Coignard (2008) が提案:

$$DR = \frac{\sum_i w_i \sigma_i}{\sigma_p}$$

DR = 1: 分散効果なし (全 component が完全相関)
DR > 1: 分散が効いている

**本ポートフォリオの VIX regime 別 DR**:
```
VIX regime          Avg DR    解釈
Low (<15)             2.83    平常時: 中程度の分散
Normal (15-20)        3.36    通常: 良い分散
Elevated (20-25)      3.70    やや不安定: 分散向上
High (25-35)          4.03    ストレス: 分散さらに向上
Crisis (>=35)         5.06    危機: 最大の分散!
```

**通説とは逆に、本ポートフォリオは危機時に DR が最大化する**。これは overlay system が equity exposure を自動的に削減し、defensive components (CTA, rate) の比重が相対的に高まるためである。

この発見は Chapter 23 (DD Control) で vol cap 不要の根拠として使用される。

## 2.4 相関の時間変化: R3 削除の数学的根拠

VTV-VUG (Value vs Growth) spread の相関推移:
```
2015-2020: ρ = 0.87-0.92 (安定)
2021:      ρ = 0.47 (構造崩壊)
2022-2025: ρ = 0.53-0.82 (不安定)
```

相関が不安定化すると、spread signal の SNR (Signal-to-Noise Ratio) が低下し、cross-sectional ranking の predictive power が失われる。これが R3 component を drag 化させた mathematical mechanism である (Chapter 12 で実証)。

## 2.5 Inverse-Volatility Weighting の数理

本書で採用する inverse-volatility weighting の定式化:

$$w_i(t) = \frac{1/\hat{\sigma}_i(t)}{\sum_j 1/\hat{\sigma}_j(t)}$$

ここで $\hat{\sigma}_i(t) = \text{std}(r_i[t-504:t]) \times \sqrt{252}$ は 504 日 rolling annualized volatility。

**重要な実装詳細**:
- Python slice `[t-504:t]` は t を含まない → day t の weight は day t-1 までのデータで計算 (look-ahead なし)
- Weight は day t の return に適用 → 「day t の open で rebalance」の前提
- 最初の 504 日は 1/N fallback (warmup)

この実装の時点整合性は T1 shift test (Chapter 26) で検証済み: inv-vol weight を shift(1) しても Sub2 Sharpe は Δ = -0.002 (完全に robust)。

---

*次章: Sharpe Ratio の正しい理解と落とし穴*
