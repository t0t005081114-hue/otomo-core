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
- 重大なReview Level判断（Forced Level 2条件に該当する変更の引き下げ）

### ChatGPT

- 壁打ち
- 要件整理
- 仕様監査
- 全体設計
- 独立レビュー
- Product横断監査
- CORE昇格候補評価
- Risk classification監査
- 独立レビューScope整理

### Claude Code

- 実装
- 修正
- Test
- Refactoring
- 指定されたPhase作業
- self-review
- Review Assurance Level候補とRisk Driversの申告
- Review Handoff作成

### Codex

主な利用候補:

- 独立コードレビュー
- 回帰確認
- 実装検証
- 別視点からの設計・コード監査
- Targeted Review / Full Review
- 申告されたReview Level・Review Scopeの妥当性判断

役割は固定された製品依存仕様ではなく、必要に応じて更新できる。

重要原則: 実装担当と独立レビュー担当を可能な限り分離する。

実装担当は、自分の変更のReview Assurance Levelを単独で引き下げられない（§5 Level Assignment Authority）。

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

すべての変更へ同じ深さの独立レビューを一律に要求しない。変更のriskを分類し、Review Assurance Levelに応じたReview ScopeとEvidenceを要求する。

Risk
↓
Review Assurance Level
↓
Review Scope
↓
Required Evidence

ただし、Independent Reviewの要否判断そのものを黙って省略してはならない。すべての変更はRisk Classificationを通る。

判定の結果としてLevel 0になった変更は、独立レビュアーによるレビュー自体を正式に免除できる。これは「レビュー制度の外にある変更」ではなく、Risk Classificationを通した結果としてのExemptionである。判定を経ずに独立レビューをskipすることは、Level 0ではなく手順違反である。

### Review Assurance Level

| Level | 名称 | Independent Review | Clean-room Verification |
|---|---|---|---|
| L0 | Minimal | 免除（Independent Review Exemptionを記録） | 要求しない |
| L1 | Targeted Independent Review | 必要（Acceptance-critical / risk-criticalを中心とする） | 原則不要。escalation条件に該当する場合に実施 |
| L2 | Full Independent Assurance | 必要（対象changeの影響範囲全体） | 必要 |

判定単位は原則 logical change / PR change-unitであり、Phase単位ではない。Phaseに1件の高risk変更があることを理由に、無関係な変更までL2にしない。

- 複数Levelの条件に該当する場合、最も高いLevelを採用する
- 判断できない場合、下位Levelへ推測で落とさず、上位Levelへescalateする
- changed file数・changed line数だけでLevelを決めない

### Risk Classification

#### Forced Level 2

次のいずれかに該当する変更は、規模にかかわらずL2とする。

- Authentication / Authorization / permission / access control / RLS
- DB schema migration、destructive migration、data deletion
- payment / billing
- secrets / credential handling
- security boundary
- 個人情報・機密情報の取得・保存・送信・表示
- data retention / deletion policy
- infrastructure / deployment security
- irreversible operation
- critical data integrityに影響する変更
- repository / CI / Harness自体のsecurity boundary
- 外部APIに対する不可逆または高影響な副作用

該当判定は「その技術を使っているか」ではなく「失敗したときに何が起きるか」で行う。外部APIを呼ぶ、DBを読む、という事実だけでL2にしない。

判断軸:

1. 失敗時の影響範囲
2. 可逆性（元へ戻せるか、戻すコスト）
3. 権限境界を変えるか
4. dataを破壊・流出・改変しうるか

過剰分類はレビュー資源を薄め、本当に独立性が必要な箇所のレビュー強度を下げる。

#### Level 0の条件

次をすべて満たす場合にのみL0とする。

- 実行時behaviorを変えない、または変更がuser-visibleな表示・文言・styleに閉じている
- 既存Pattern内の局所的変更であり、新しい責務境界・data flow・外部連携を作らない
- Forced Level 2条件に1つも該当しない
- Productが定義するValidationがすべてPASSしている
- Acceptance Criteriaへの影響が確認済みで、未確認の仮定が残っていない

例: documentation-only、文言修正、単純なUI / style修正、behaviorを変えない機械的変更。

「軽微だから」「小さいから」「時間がないから」はL0の根拠にならない。上記条件のどれに該当するかを具体的に示せない変更をL0にしない。

#### Level 1がdefault

Forced Level 2条件に該当せず、Level 0の条件も満たさない変更はL1とする。新機能、複数ファイルのbehavior変更、新しいComponent、新しいdata flow、application logic追加等、既存責務内の通常実装はここに入る。

迷った場合のdefaultはL1であり、L0ではない。

### Level Assignment Authority

- 実装担当は、自分の変更のReview Levelを単独で引き下げられない。Level候補とRisk Driversを申告する
- 実装担当以外によるRisk Classification監査で、通常のL0 / L1判定を確定できる
- 独立レビュー担当・ChatGPT・HumanはLevelを引き上げられる
- Forced Level 2条件に該当する変更をL1以下へ引き下げる場合のみ、Human判断を必要とする
- 「独立レビューの利用量を減らしたい」「コストを下げたい」はLevel引き下げの理由として認めない
- 判断できない場合は上位Levelへescalateする

実装担当
→ Level候補とRisk Driversを申告

実装担当以外のRisk Classification監査
→ L0 / L1を確定、または上位Levelへescalate

Forced Level 2条件に該当する変更の引き下げ
→ Human判断

Level判定そのものもレビュー対象である。Levelが引き上げられた場合、その理由を記録する。

### Level 0 — Minimal

要求:

- 実装担当のself-review
- Productが定義するValidation
- Acceptance確認
- diff全体の確認（意図しない変更が混入していないこと）

独立レビュアーによるレビューは免除できる。ただし黙って省略しない。免除したこと自体をIndependent Review Exemptionとして、GitHubから追跡可能な形で残す。

最低記録項目:

- Review Level
- Exemption reason（Level 0の条件のどれに該当するか。具体的に）
- Validation evidence（実行したcommand、結果、Acceptance確認）
- Residual Risk（独立レビューを行わなかったことによる残存risk）

保存先は§4 Durable Historyと同じ候補（PR description、Phase Completion Report、Issue、commit message等）から選ぶ。COREは特定の記録形式を強制しない。GitHubから追跡可能であることを要求する。

後からForced Level 2条件への該当が判明した場合、その免除は無効になる。該当変更を独立レビューへ戻す。

### Level 1 — Targeted Independent Review

実装担当は、独立レビューの前にself-review、Productが定義するValidation、Acceptance確認を完了させる。独立レビューは実装担当のValidation不足を補う工程ではない。

独立レビュー担当は、PR全体を同じ深さで読むのではなく、次を中心に監査する。

- Acceptance-critical path
- risk-critical diff
- 新設・変更された責務境界
- 新設・変更されたdata flow
- 上記に対応するtestが、主張する失敗を実際に検出できるか

Review Scopeをchanged file数・changed line数だけで決めない。

#### Review Handoff

実装担当は独立レビュー担当へ最低限以下を渡す。

- changed behavior
- Acceptance-critical files / paths
- 新設・変更された責務境界
- 重要なdata flow
- 関連test
- 前提としたassumption
- Residual Risks
- mechanical / supporting changeとみなしたfilesと、その理由

Handoffは監査の出発点であり、Review Scopeの上限ではない。

- 独立レビュー担当はHandoff自体を検証対象として扱い、diffと仕様を自分で確認する
- mechanicalと申告された変更に、behavior変更・責務境界変更・risk driverが含まれていないことを確認する
- 必要と判断した場合、Review Scopeを拡張できる。Forced Level 2条件を発見した場合はL2へescalateする
- Handoffが無い、または内容を確認できない場合、Review Scopeを狭める根拠が無いものとして扱う

#### Clean-roomへのescalation

L1でも次の場合はClean-room Verificationへescalateする。

- 実装担当環境への隠れた依存が疑われる
- Validation結果を再現できない
- 独立レビュー担当が必要と判断した

### Level 2 — Full Independent Assurance

要求:

- Full Independent Review
- Clean-room Verification
- Verification Evidence（下記 Verification Evidence のL2要求）

#### 対象範囲

L2が要求するのは、L2に分類されたlogical changeについて、その影響範囲全体をFull Independent Reviewすることである。

- 同一PR内にあるという理由だけで、無関係なfileがL2の対象になるわけではない
- 同一PR内に安全に分離可能な独立したL0 / L1 changeがある場合、それらを自動的にL2へ昇格させない
- ただし、change同士の境界や影響範囲を安全に分離できない場合は、PR全体をL2として扱う
- 分離可能性を判断できない場合は分離しない

L2ではReview Scopeの縮小を認めない。これはL2に分類されたlogical changeの影響範囲内で維持される。影響範囲の外側にある独立changeを別Levelで扱うことは、Scopeの縮小ではなく判定単位の問題である。

分離した場合、何をL2の影響範囲内とし、何を独立changeとして分離したかをEvidenceへ記録する。分離判断自体がレビュー対象である。

#### 実行手段

L2を満たす実行手段を1つに固定しない。`harness/REMOTE_REVIEW.md` のRemote Reviewは、clean checkout → Deterministic Verification → Codex Independent Review → Evidenceを一経路で満たす有力な手段だが、L2で必須の唯一の方法ではない。同等の独立性・再現性・Evidenceを満たす他の手段でもよい。

### blocking / advisory

レビュー指摘を原則以下へ分類する。

#### blocking

解消しなければPhase完了不可。

例:

- requirement違反
- データ破壊
- セキュリティ問題
- 重大な回帰
- Acceptance Criteria未達
- 明確な設計矛盾
- clean environmentで必須Validationが再現しない（実装担当環境への隠れた依存）

#### advisory

Phase完了を妨げない改善候補。必要に応じて将来対応する。

### Codex Review Durable History

`/codex:review` を実施する変更は、AI conversationだけにreview結果を残さない。

- 原則として commit・push・PR作成まで行う
- Codex review結果は、対象PRのConversationまたはReviewへDurable Historyとして保存する（Evidence Integrityに従い、絶対パス・機密等はsanitizeする）
- Findingを修正した場合は、同じPRへ追加commitし、再review結果もPRへ保存する
- `/codex:review` 実行者はPRをmergeしない。mergeはHumanの明示承認後にのみ行う

本節は記録と権限の運用であり、Review Assurance Levelやblocking / advisoryの判定基準を変更しない。

### Clean-room Verification

L2では必須とする。L1ではescalation条件に該当する場合に行う。L0では要求しない。

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

Review Assurance Levelは、必要なEvidenceの量と範囲を決める。Evidenceの一次情報性とEvidence Integrityの要求はLevelによって変わらない。

| Level | 最低Evidence |
|---|---|
| L0 | 実行したValidation command、結果 / exit status、Acceptance確認、Exemption reason、Residual Risk |
| L1 | L0の内容に加えて、independent review scope、監査したcritical path、独立レビューのverdictとfinding、Review Scopeを限定したことによるResidual Risk |
| L2 | 対象SHA、clean environmentの分離方法、実行したcommand、exit code、必須Validation結果、独立レビューのverdict、Evidence Integrity |

### Residual Risk from Reduced Review Scope

L0 / L1では、Full Reviewを行わなかったこと自体が残存riskである。これを「riskが無い」と記録しない。

- L0: 独立レビューを行わなかった範囲と、それを許容した理由
- L1: 監査したscope、意図的に深く監査しなかったscope、その限定が妥当だった理由

該当が無ければ `None` でよい。形式を満たすためだけの長文記入はしない（§4と同じ）。

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
