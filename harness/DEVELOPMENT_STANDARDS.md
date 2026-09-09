# OTOMO Development Standards

本ファイルはOTOMO Product共通の開発標準を定義する。

Product固有仕様を追加・変更するものではない。

## 1. Role Separation

### Human

- 最終仕様判断
- Business判断
- Priority判断
- Scope判断
- 重大なCORE Rule変更
- Knowledge昇格承認

### ChatGPT

- 壁打ち
- 要件整理
- 仕様監査
- 全体設計
- 独立レビュー
- Product横断監査
- CORE昇格候補評価

### Claude Code

- 実装
- 修正
- Test
- Refactoring
- 指定されたPhase作業

### Codex

主な利用候補:

- 独立コードレビュー
- 回帰確認
- 実装検証
- 別視点からの設計・コード監査

役割は固定された製品依存仕様ではなく、必要に応じて更新できる。

重要原則: 実装担当と独立レビュー担当を可能な限り分離する。

## 2. Before Implementation

実装・修正前に最低限確認する。

- Product requirements
- Product正式仕様
- Product固有Rule
- 関連Failure
- 未解決blocking Issue
- 直近の重要commit / PR / review
- OTOMO CORE共通Rule

不整合を発見した場合、独断で解釈しない。明示して判断を戻す。

## 3. Scope Discipline

- 仕様にない機能を「便利そう」という理由で追加しない
- TBDを無断で確定しない
- 将来構想を現在Phaseへ持ち込まない
- 最小の実装で現在の目的を満たす

## 4. Durable History

重要情報をAIセッション内だけに残さない。

GitHubから以下を追跡可能にする。

- 何を変更したか
- なぜ変更したか
- どの仕様に基づいたか
- 何で検証したか
- どのレビュー指摘を修正したか

利用候補:

- commit message
- PR description
- Issue
- Product Decision
- Product Failure log

## 5. Independent Review

実装完了とレビュー完了を同一視しない。

レビュー指摘を原則以下へ分類する。

### blocking

解消しなければPhase完了不可。

例:

- requirement違反
- データ破壊
- セキュリティ問題
- 重大な回帰
- Acceptance Criteria未達
- 明確な設計矛盾

### advisory

Phase完了を妨げない改善候補。必要に応じて将来対応する。

## 6. Failure Learning

重要な失敗を修正だけで終わらせない。

`learning/FAILURE_SCHEMA.md` に該当する場合、Product Repositoryへ記録する。

他Productでも再利用可能な場合はKnowledge昇格候補にする。

## 7. Source of Truth

- GitHub: 開発上の事実と履歴
- Obsidian: 一般化された再利用Knowledge
- AI conversation: 一時的な作業空間

重要な事実をAI conversationだけに残さない。

## 8. No Silent Harness Mutation

AIはOTOMO COREの重要Ruleを自動確定・自動変更しない。

改善候補を提示することはできるが、正式反映にはレビューと人間承認を必要とする。
