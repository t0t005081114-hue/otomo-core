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

## 9. External Boundary Safety

外部プロセス、外部API、外部Service等との境界では、外部から受け取る生の出力を安全な情報として扱わない。

- 機密を含みうる生の標準エラー、URL query、response body、外部exception等を、そのままlog、exception、tracebackへ流さない
- 後段のマスキングだけに依存せず、境界で安全な診断情報と機密を含みうる生出力を分離し、必要最小限へ縮約する
- 外部エラーをRetryする前にtransient / permanentを分類する
- Retry対象はtransientな失敗に限定し、permanentな失敗を一律に再試行しない

安全な診断に不要な外部生出力は、観測経路へ渡さない。

## 10. Explicit Invariants / Fail Fast

正しさ、信頼度、分類、安全性、Business判断に関係する値のうち、仕様が明示的な判断を要求するものへ、単に「よく使う値だから」という理由で暗黙defaultを設定しない。

- 必須の判断が未指定なら、境界で明示的に失敗させる
- data / config invariantを境界で検証する
- 既知のinvariant違反により安全または正しく処理できない場合、警告だけで続行しない
- 無関係な後続処理で失敗させず、原因を特定できる位置でfail fastする

## 11. Aggregation / Confidence Integrity

件数、source数、vote数、event数、record数等を根拠にconfidence、trust、verification、importance等を格上げする場合、件数の多さだけをsemantic confidenceの根拠にしない。

最低限、以下を確認する。

1. 数えている実体がdistinctである
2. duplicate IDやduplicate eventを除外している
3. 同じgroup / cluster / themeに属することと、同じclaim / fact / decisionを裏付けることを区別している
4. 集約単位より細かいsemantic relationが必要な場合、その対応関係を明示的に確認している

集約結果を信頼度や検証状態へ変換する実装とレビューでは、重複排除と意味的な対応関係の両方を検証する。
