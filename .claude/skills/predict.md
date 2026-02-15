# /predict - 競馬レース予想スキル

## 概要
指定されたレースの予想を、Multi-Agentループで実行するメインスキル。

## 使い方
```
/predict 2026年2月15日 東京11R フェブラリーS
/predict 今週の中山メイン
```

## 実行フロー

ユーザーがレースを指定したら、以下のCentralized Agent Loopを実行する。

### Step 1: レース特定と計画立案（Plan Agent）

`agents/plan.md` のプロンプトに従い、Task toolでPlan Agentを起動する。

```
Task(subagent_type="general-purpose", prompt=<agents/plan.mdの内容 + レース情報>)
```

Plan Agentが以下を決定する:
- 対象レースの正式情報（日付・会場・レース番号・レース名・距離・条件）
- 収集すべき情報リスト
- 注目すべき観点

### Step 2: 情報収集（Collect Agent）

`agents/collect.md` のプロンプトに従い、Task toolでCollect Agentを起動する。

```
Task(subagent_type="general-purpose", prompt=<agents/collect.mdの内容 + Plan Agentの出力>)
```

Collect Agentが以下を実行する:
- WebSearchで出走馬・枠順・オッズなどを収集
- WebSearchで各馬の近走成績を収集
- WebSearchでトラックバイアス・馬場状態を収集
- 過去データ（data/knowledge/）を参照
- 収集結果を `data/races/` にJSON保存

### Step 3: 推論・スコアリング（Reason Agent）

`agents/reason.md` のプロンプトに従い、Task toolでReason Agentを起動する。

```
Task(subagent_type="general-purpose", prompt=<agents/reason.mdの内容 + 収集データ>)
```

Reason Agentが以下を実行する:
- 各馬のファクター別スコア算出
- 総合スコアの算出
- オッズとの比較による期待値計算
- 推奨買い目の作成

### Step 4: 批評・改善（Critique Agent）

`agents/critique.md` のプロンプトに従い、Task toolでCritique Agentを起動する。

```
Task(subagent_type="general-purpose", prompt=<agents/critique.mdの内容 + Reason Agentの出力>)
```

Critique Agentが以下を評価する:
- 推論の論理的整合性
- 見落としているファクター
- 過大/過小評価の指摘
- 追加収集が必要な情報

### Step 5: ループ判定

Critique Agentの出力に基づき:
- 「十分」→ Step 6へ
- 「不十分」→ Step 2に戻る（最大3ループ）

### Step 6: 最終予想出力

最終予想を以下のフォーマットで出力し、`data/predictions/` に保存する:

```
## 🏇 予想結果: {レース名}

### レース概要
- 日付: YYYY-MM-DD
- 会場: XX
- R: XX
- 距離: XXXX m (芝/ダート)
- 馬場状態: X

### 総合スコアランキング
| 順位 | 馬番 | 馬名 | 総合スコア | 期待回収率 |
|------|------|------|-----------|-----------|
| 1    | XX   | XXX  | XX.X      | XXX%      |
| ...  |      |      |           |           |

### 推奨買い目
- 券種: XXX
- 買い目: XXX
- 推奨投資配分: XXX

### 予想根拠（要約）
- ...

### 注意事項・当日補正ポイント
- 馬体重 ±Xkg以上の変動があれば再評価
- トラックバイアスの確認ポイント
```

## 当日補正について

レース直前にユーザーが追加情報（馬体重、パドック情報、トラックバイアス等）を提供した場合:
1. 該当ファクターのスコアを再計算
2. 総合スコアを更新
3. 買い目を必要に応じて修正
4. 補正内容を記録

補正は `/predict` に追加情報を添えて再実行:
```
/predict 補正 東京11R 馬体重: ドウデュース+8kg, トラックバイアス: 内有利
```
