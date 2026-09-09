# OTOMO Knowledge Promotion

## 1. Purpose

各Productで得たFailure・改善知識を、必要に応じてOTOMO全体へ還元する。

## 2. Promotion Candidate

以下に該当する場合、CORE昇格候補とする。

- 複数Productで利用可能
- 再発可能性が高い
- AIが繰り返し間違える
- Humanが繰り返し同じ指示をしている
- セキュリティに関係する
- データ整合性に関係する
- 開発Workflow改善につながる
- Test / Hook等で機械的に防止できる

## 3. Do Not Promote

以下はProduct側に残す。

- Product固有Feature
- Product固有UI
- Product固有Business rule
- Product固有KPI
- 一時的事情
- 一回限りで再発性の低い問題

## 4. Promotion Flow

Product execution
↓
Failure / repeated correction
↓
Product Failure log
↓
Knowledge promotion candidate
↓
一般化
↓
既存CORE Ruleとの重複確認
↓
改善案
↓
独立レビュー
↓
Human approval
↓
OTOMO COREへ反映
↓
`learning/PROMOTION_LOG.md` へ昇格履歴を記録

Product側のFailure原本は移動・削除しない。

昇格後も、どのProductのどのFailureから生まれ、どのCORE Harnessへ反映されたかを追跡可能にする。

## 5. Promotion Destination

### Test

機械的に再現・検出できる問題。

### Hook / Validation

作業前後に自動検査可能。

### Rule

判断原則として残す必要がある。

### Workflow

作業順序を固定すると防止できる。

### Skill

複数Stepの繰り返し作業として再利用価値がある。

### Checklist

完全自動化できないが確認手順で防止できる。

## 6. Prefer Enforcement Over Documentation

可能なら「注意する」だけで終わらせない。

優先順位:

自動防止
>
自動検知
>
Workflow化
>
Checklist化
>
文章Ruleのみ

ただし、仕組みが過剰になる場合は最小構成を優先する。

## 7. Promotion Record

正式昇格したKnowledgeは `learning/PROMOTION_LOG.md` に記録する。

最低記録項目:

- Promotion ID
- Date
- Source Product / Repository
- Source Failure / evidence commit
- Generalized lesson
- Enforcement method
- Promotion destination
- Approval / application status

Promotion LogはProduct Failure logのコピーではない。

COREへ昇格した知識とHarness変更の追跡索引として使用する。

## 8. Human Approval

AIは昇格候補を抽出・一般化・提案できる。

ただし、OTOMO COREの正式Ruleへの昇格は人間承認を必要とする。

特に以下は自動変更しない。

- Scope
- Security policy
- AI responsibility
- Product responsibility
- Git運用
- Data handling
