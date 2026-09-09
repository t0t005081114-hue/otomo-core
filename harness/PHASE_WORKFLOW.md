# OTOMO Phase Workflow

## 1. Before Phase

Phase開始前:

- [ ] 対象Productのrequirementsを確認
- [ ] 正式仕様を確認
- [ ] Product固有Ruleを確認
- [ ] OTOMO CORE共通Ruleを確認
- [ ] 関連Failureを確認
- [ ] 未解決blocking Issueを確認
- [ ] 対象Phase Scopeを特定
- [ ] Acceptance Criteriaを確定
- [ ] 本Phaseを止める本当のTBDだけを抽出

無関係なTBDでPhase全体を停止しない。

## 2. Implementation

- 確定Scope内のみ実装する
- 勝手なFeature追加をしない
- 既存責務境界を理由なく変更しない
- 必要なTestを追加する
- 重要判断をGitHubへ残す

## 3. Validation

各Product Repositoryが自身のValidation commandを定義する。

例:

- Unit test
- Integration test
- lint
- typecheck
- build
- visual validation
- accessibility
- security validation

OTOMO COREは特定言語・Frameworkのコマンドを強制しない。

最低原則:

変更対象の検証
↓
全体回帰検証

の順で行う。

## 4. Independent Review

実装後に独立レビューを行う。

確認項目:

- requirement fidelity
- scope creep
- regression
- security
- data integrity
- unnecessary complexity
- test gaps
- incorrect assumptions
- responsibility boundary violation

判定:

- PASS
- FAIL

FAILには少なくとも1件のblocking findingが存在する。

## 5. FAIL Loop

blocking findingがある場合:

1. 原因を確認
2. 修正
3. 必要な回帰Test追加
4. Validation再実行
5. 再レビュー
6. 必要ならFailure記録

blockingが0になるまでPhase完了としない。

## 6. Phase Completion

最低完了条件:

- Acceptance Criteria達成
- 必須Validation PASS
- blocking finding = 0
- 重要Decision記録済み
- 重要Failure記録済み
- Git working stateが理解可能
- commit / branch等の実装Anchorが存在する

## 7. Completion Report

最低報告項目:

- 実装内容
- 変更ファイル
- Validation結果
- 独立レビュー結果
- blocking残件
- advisory残件
- 重要Decision
- Failure / 再発防止
- Knowledge昇格候補
- commit hash
- branch
