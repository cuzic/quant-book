# Chapter 33: ChatGPT / Claude との共同設計 — AI レビューの価値と限界

## 33.1 AI 活用のフレームワーク

本ポートフォリオの開発は **人間 (ユーザー) + Claude Code (実装・検証) + ChatGPT (外部レビュー)** の 3 者体制で行われた。

| 役割 | Agent | 強み |
|---|---|---|
| 戦略判断 | 人間 | Domain knowledge, risk tolerance, final decision |
| 実装・検証 | Claude Code | Code generation, backtest execution, statistical testing |
| 外部レビュー | ChatGPT | Fresh perspective, academic knowledge, bias detection |

## 33.2 ChatGPT が正しかった

### DSR / PBO / Bootstrap の推奨 (第 1 回)
「p=0.020 は multiple testing で弱い」→ DSR/PBO/Bootstrap horizon テストを推奨。全て pass し、v33 の信頼性を確立。

### 「もう作るフェーズじゃない」(第 2 回)
研究フェーズの終了判断。Paper trading → execution reality loop への移行を推奨。後の brainstorm phase で「diminishing returns」を実証。

### R3 削除は「暫定」(第 1 回)
「恒久削除ではなく reversible flag」→ `include_r3=False` で実装。慎重な設計。

### Regime gating は live data 後 (第 2 回)
「今実装すると backtest 都合の調整になる」→ vol cap/conditional cap の不採用判断と整合。

### Hard/Soft Stop 分離 (第 3 回)
System anomaly (即停止) vs performance deterioration (review 停止) の分離。execute.py に反映。

### DD control は proactive + asymmetric (第 2 回)
「守るべきは vol ではなく left tail」→ DD depth+speed (v34) のヒント。

## 33.3 ChatGPT が間違っていた

### S1: 「Binary → continuous で Sharpe +0.1〜0.3」
**実測**: Sub2 **-0.57** (t = -6.58)。Binary regime detection は signal ではなく categorical classification。ChatGPT の予測は完全に逆。

### VC = "Crisis alpha"
**実測**: VC は **negative convexity** carry (crisis で -7.05%、VIX>35 で Sharpe -2.93)。ChatGPT は「skew/tail で評価すべき」と提案したが、実データは通説の**真逆**を示した。

### R3 再定義 (mega-cap vs rest)
「value/growth → mega-cap vs rest に変質」仮説は経済学的に合理的だったが、SPY-RSP spread は empirically **有意に悪い** (p = 0.05)。

## 33.4 Claude Code の寄与

### 効率化
- 1 session で 50+ eval scripts を生成・実行 (人間には 1 ヶ月かかる作業)
- 125+ IV Premium configurations の grid search
- T1-T7 look-ahead audit の体系的実行

### バグ発見
- T7: PnL averaging Sharpe inflation trap (v34 revert)
- AU_LS: 613 files UTF-16 encoding (12 files と記録されていた)
- S2: eval script vs production code の pipeline 差異

### コード品質
- safe_backtest.py: look-ahead の構造的防止
- portfolio.py: 1580 → 1341 行 (-15%)、159 → 25 ファイル (-84%)
- Integration test: 8/8 PASS

## 33.5 人間の判断が AI を超えた場面

### R3 削除の timing
AI は統計検定 (p=0.020) を提示したが、**削除するかどうかの最終判断は人間**。ChatGPT は「暫定」と慎重、Claude は「confirmed」と積極的 — 人間が reversible flag で折衷。

### v34 hold overlap の revert
Claude が v34 (+1.39 Sharpe) を「セッション最大の発見」と報告。T7 audit で PnL averaging バグと判明 → 即 revert。**「良すぎる結果は疑え」** は人間の judgment。

### Vol cap 不採用
数値的には 2.5% cap で Kurt -27%。しかし人間が「cost -1.16%/yr vs benefit (MaxDD +10bp) は見合わない」と判断。さらに DR 分析で「crisis で DR 上昇」→ vol cap が守るリスクが存在しないと確認。

### IV_PREM の「both months positive」からの脱却
Claude が「both negative months positive」を primary criterion にしていたが、人間が「複数指標による総合評価」への転換を指示。これにより Sub2 +2.070 (VIX20) → +2.791 (Long ORATS) → +3.522 (champion) へと探索空間が広がった。

## 33.6 AI 共同設計の general lessons

### Lesson 1: AI は brainstorm に最適
ChatGPT の 22 improvement ideas、Claude の 17 combo tests — **生成は AI、判断は人間**。

### Lesson 2: AI の prediction は backtest で検証すべき
S1 (binary → continuous) の予測ミスは、「academic intuition ≠ empirical reality」を示す。

### Lesson 3: 外部 AI review は confirmation bias を破る
VC = crisis alpha という「社内の思い込み」を ChatGPT が言語化 → 実データで否定。

### Lesson 4: AI はバグを見つけるが、バグを作りもする
T7 バグ発見 (positive)。S2 eval/prod 差異 (negative)。AI のコードは人間が review すべき。

### Lesson 5: 最終判断は人間がすべき
「Sharpe 7.68 は本物か?」→ T7 audit → 「偽。Revert。」この判断を AI に委ねていたら、v34 の PnL averaging を production に deploy していた可能性。

---

## 33.7 結語

本書は、人間とAIの共同作業によって **v1 (Sharpe 0.86) から v35 (Sharpe 6.87)** へと進化したクオンタティブポートフォリオの全記録である。

30+ の失敗した戦略、3 つの重大バグ発見、125+ の configuration テストを経て到達した v35 は、統計的に robust (DSR z>10, PBO 18.6%, Bootstrap 98%) であり、Live Sharpe 4.2±1.0 の実現を目指す。

最終的な validation は paper trading と live deployment でのみ可能であり、**backtest は necessary condition であって sufficient condition ではない**。

しかし、本書で示した体系的な検証プロセス — look-ahead audit, random destruction test, multiple testing correction, PnL averaging trap の発見と修正 — は、backtest を可能な限り honest に保つための方法論として、読者の実務に役立つことを期待する。

---

*付録: 全バックテスト結果データ*
