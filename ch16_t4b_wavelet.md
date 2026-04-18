# Chapter 16: T4b — Wavelet Multi-Resolution Signal

## 16.1 学術的根拠

### Wavelet 分解

Mallat (1989) の multi-resolution analysis: 時系列を異なる周波数帯域に分解し、各帯域の情報を独立に利用する。Gencay, Selcuk & Whitcher (2002) が金融への応用を体系化。

### 本戦略のアイデア

SPY (S&P 500) の対数価格を Daubechies-4 wavelet (db4, level 5) で分解:
- **A5 (Approximation 5)**: 長期 trend (数ヶ月スケール)
- **D3 (Detail 3)**: 中期 cycle (数週間スケール)

Signal logic:
```python
trend = a5[-1] - a5[-10]    # 10 日間の trend 方向
cycle = d3[-1]               # cycle の位相

if trend > 0.01 and cycle < 0:   # 上昇 trend + cycle bottom → Long
    signal = +1
elif trend < -0.01 and cycle > 0: # 下降 trend + cycle top → Short
    signal = -1
```

**解釈**: trend が上向きで cycle が底打ちした時に買う = 「trend 方向への cycle の反転」を捕捉。

## 16.2 実装

```python
# portfolio.py: _build_wavelet()
window = 256
coeffs = pywt.wavedec(log_p, "db4", level=5)
a5 = pywt.waverec([coeffs[0]] + [np.zeros_like(c) for c in coeffs[1:]], "db4")
d3 = coeffs[3]
```

SPY のみの単一銘柄 signal。shift=2 (2 日遅延) で look-ahead 防止。

## 16.3 パフォーマンス

```
Standalone Test Sharpe: +0.68
IV weight: ~4.4%
Ablation: Drop T4b → Sub2 -0.041 (marginal)
```

T4b は standalone alpha は modest。Portfolio への寄与も marginal。

## 16.4 Turnover 問題の調査

Cost stress test で T4b の turnover を **50%/日** と想定していたが、実測は **10.8%/日**。

想定 50% は「signal が -1/0/+1 を毎日 flip」の assumption だったが、実際の trend+cycle 条件は**大半の日で未発火** (signal = 0) → turnover は想定の 1/5。

### T4b variants テスト (8 variants)

```
baseline (current):       Test +0.683  Sub2 +0.779
widen threshold 0.02:     Test +0.152  Sub2 -0.200
hysteresis 0.02/0.005:    Test +0.033  Sub2 -0.298
rebalance 3d:             Test +0.718  Sub2 +0.396
```

**Baseline が最良**。Threshold を広げると signal が希薄化、rebalance 3d は Sub2 で半減。

## 16.5 T4b の学術的意義

Wavelet decomposition は**技術的に面白い**が、本 portfolio での実用的寄与は limited:
- Weight 4.4% × standalone +0.68 → portfolio Sharpe +0.03 程度の寄与
- 他 component (MR, GTAA, JP_LS) の方が alpha が大きい

**但し**: T4b は他の全 component と低相関 → **分散の「穴埋め」** として機能。特定の market microstructure (trend + cycle interaction) を capture する唯一の component。

### 実務家へのノート

T4b は `pywt` (PyWavelets) library に依存。Production deployment では library の安定性を確認すること。Wavelet 分解は計算量が moderate (window=256 × daily = O(N log N)) で latency は問題なし。

---

*次章: JP_LS — 日本株 10-factor PnL-stacking*
