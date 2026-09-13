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

### Implementation Decision Trace

実装結果だけでなく、後続の人間・AIが同じ判断を再探索しないために、重要な判断があった場合は以下もGitHubから追跡可能にする。

- Rejected Approaches: 検討したが採用しなかった方法と、その理由（技術・仕様・Scope）
- Deliberate Non-Goals: 意図的に実装しなかったこと、Scope外とした理由、将来対応か明示的不採用か
- Residual Risks: 完了時点で残るリスク、発生条件、影響範囲、将来確認すべき事項

記録対象は、後続作業の判断を変えうるものに限る。該当がなければ `None` でよく、軽微な変更に形式的な長文記録を求めない。

- 保存先はPhase Completion Report、PR description、Product Decision、Product Failure logを優先する
- Completion ReportがAI conversation内にしか存在しない場合、Decision TraceはGitHub上の保存先にも残す
- commit messageには必要な場合に要約のみを残し、Decision Trace全文を詰め込まない
- 試して失敗した方法が `learning/FAILURE_SCHEMA.md` に該当する場合は、Failureとしても記録する

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
- clean environmentで必須Validationが再現しない（実装担当環境への隠れた依存）

### advisory

Phase完了を妨げない改善候補。必要に応じて将来対応する。

### Clean-room Verification

独立レビュー担当は、実装担当の作業環境をそのまま信用せず、可能な限り実装担当の作業環境から分離したclean environmentで再検証する。

目的は「実装担当の環境で動いた」ではなく「別環境でも再現できた」を確認することである。

環境の実現方法は問わない（例: fresh checkout、新しいworktree、disposable directory、container）。特定の技術・infrastructureを必須にしない。

最低原則:

1. 対象commit / branch / SHAを明示する
2. clean environmentへ対象コードを取得する
3. Productが定義する手順で依存関係を準備する（実装担当環境のinstall済み依存、未commitファイル、暗黙の設定に依存しない）
4. Productが定義するValidationを実行する
5. 実行結果をVerification Evidenceとして保持する
6. Evidenceを基にPASS / FAILを判断する

clean environmentを用意できない場合は、その理由と、実装担当環境と何を共有したかを明示する。

### Verification Evidence

PASS / FAILは一次情報を根拠に判断する。「AIがPASSと言った」「確認済み」という記述だけをEvidenceとして扱わない。

一次情報の例:

- 実行したcommand
- exit code
- stdout / stderr
- test / build / lint / typecheck / migration / integration の結果

Productに存在しないValidationを、Evidence取得のために強制しない。

### Evidence Integrity

Evidenceの取得と永続化を区別する。

Evidenceを取得
↓
機密を含まないか確認
↓
必要ならredact / sanitize / summarize / safe excerpt化
↓
安全を確認したEvidenceのみGitHubへ永続化

- 生のstdout / stderr等は、検証環境内でEvidenceとして取得・参照してよい
- commit、PR、Issue等へ残す前に、secrets、credentials、API key、token、password、個人情報、機密を含むURL / query parameter、環境変数の値等が含まれていないことを確認する
- 機密を含む可能性がある生ログを、未確認のままGitHubへ残さない
- 自動マスキングの結果だけで安全とみなさない（External Boundary Safetyと同じく、永続化の境界で必要最小限へ縮約する）
- 縮約・伏字化したEvidenceは、その旨を明示する
- Evidenceの価値より機密保護を優先する。安全に残せない場合は、command、exit code、件数等の安全な要約だけを残す

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
