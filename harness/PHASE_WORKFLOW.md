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
- 重要判断をGitHubへ残す（該当する場合はImplementation Decision Traceを含む）
- 変更のRisk DriversとReview Assurance Level候補を申告する（`harness/DEVELOPMENT_STANDARDS.md` §5）

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

## 4. Risk Classification and Independent Review

実装後、変更ごとにriskを分類し、Review Assurance Levelに応じたreview / validationを行う。

Level定義、判定Rule、Levelの引き上げ・引き下げ権限、Levelごとに必要なEvidenceは `harness/DEVELOPMENT_STANDARDS.md` §5 に従う。ここでは繰り返さない。

判定単位は原則 logical change / PR change-unitである。1つのPhase内に複数のLevelが混在してよい。

各Level（L0 / L1 / L2）で行うreview・Validation・Evidenceは `harness/DEVELOPMENT_STANDARDS.md` §5 Review Assurance Level と各Levelの節に従う。

独立レビューは `harness/DEVELOPMENT_STANDARDS.md` の Clean-room Verification / Verification Evidence / Evidence Integrity に従う。

確認項目（L2は全項目、L1はReview Scope内で該当する項目）:

- requirement fidelity
- scope creep
- regression
- security
- data integrity
- unnecessary complexity
- test gaps
- incorrect assumptions
- responsibility boundary violation
- clean-room reproducibility
- hidden environment dependency
- verification evidence integrity

判定:

- PASS
- FAIL

FAILには少なくとも1件のblocking findingが存在する。

L0で行うのは実装担当のself-reviewとValidationであり、その結果を独立レビューのPASSと呼ばない。

## 5. FAIL Loop

blocking findingがある場合:

1. Root causeを確認する
2. 同じRoot causeが影響しうる関連経路を列挙する
   - sibling branch / case
   - 共通関数の他の戻り値
   - 同じ処理を使うcall site
   - 類似するdata / error path
3. 修正対象だけでなく、影響が確認された関連経路にも必要な修正を適用する
4. 元のFailureに対する回帰Testを追加する
5. 横展開対象となった関連経路にも、必要な回帰Testまたは明示的な非該当確認を追加する
6. Validationを再実行する
7. 元の判定Level以上で独立再レビューを行う（blocking findingは当初のRisk分類が実態より低かった可能性を示すため、Review Assurance Levelの再評価を含む）
8. 必要ならFailureを記録する

1つの分岐だけが直ったことを、Root cause全体の解消とみなさない。

blockingが0になるまでPhase完了としない。

### Remediation / Re-review Protocol

独立レビューでfindingが出た後の remediation → Validation → re-review の進め方を補足する。上記FAIL Loopを置き換えるものではない。Review Assurance Level、blocking / advisory、Clean-room Verification、Evidenceの要求は `harness/DEVELOPMENT_STANDARDS.md` §5 に従い、ここでは変更しない。

#### Same PR / Branch

既存PRに対するfindingのremediationは、原則として同じPR / branchへ追加commitする。

新しいPRへ分離するのは、finding解消にscope expansionまたは独立したlogical changeが必要な場合に限る。その場合も実装担当が独断で分離せず、Scope判断としてHumanへ戻す（`harness/DEVELOPMENT_STANDARDS.md` §1 / §3）。

#### Minimal Remediation

remediationで修正するのは次に限る。

- findingのRoot cause
- 上記FAIL Loopで、同じRoot causeの影響が確認された関連経路
- 必要な回帰Test / Validation

unrelated cleanup、ついでのrefactor、新Feature追加、未決定仕様の確定をremediationに含めない。

「最小修正」はRoot causeの横展開確認を省略する意味ではない。FAIL Loopの手順2・3・5は、そのまま適用する。

#### Re-review Scope

re-reviewはfix commitだけを見て完了としない。元のfindingが属するlogical change / PR change-unitを再確認し、少なくとも次を確認する。

- previous findingが解消したか
- remediationによるregressionが無いか
- 元のAcceptance Criteriaと責務境界が維持されているか

re-reviewのLevelは上記FAIL Loop手順7と `harness/DEVELOPMENT_STANDARDS.md` §5 に従う。

#### Finding Trace

re-reviewでは、previous findingsそれぞれについて最低限以下を追跡可能にする。Finding自体の定義とseverityは `harness/DEVELOPMENT_STANDARDS.md` §5 Finding / blocking / advisory が所有する。本節は、lifecycle上で追跡する情報とstatusを定める。

- Finding ID / summary
- status: Resolved / Open / Not applicable

Not applicableとする場合は、その理由を一行で残す。記録先は `harness/DEVELOPMENT_STANDARDS.md` §4 Durable History に従う。`/codex:review` を使う場合は同 §5 Codex Review Durable History に従う。

#### New Finding Stop Rule

re-reviewで、今回のremediation roundが対象としたfinding・Root cause・関連経路に含まれない新しいfindingが見つかった場合、実装担当は次のremediation roundへ自動的に進まない。次を行って停止する。

1. findingをDurable Historyへ保存する
2. severity（blocking / advisory）、location、issueを整理する（用語は `harness/DEVELOPMENT_STANDARDS.md` §5 Finding）
3. previous findingsのstatusを明示する
4. 解消にscope expansionが必要かを整理する
5. Humanへ報告する

次のroundのremediationは、Humanが承認してから開始する。

re-reviewで元findingのRoot causeが未解消と判明しただけの場合も、実装担当は独断で修正範囲を広げない。必要な修正範囲を整理してHumanへ報告する。

本Ruleはremediationの進行を止めるものであり、blocking findingのPhase完了への扱いを変えない。blockingが0になるまでPhase完了としない点は上記のとおりである。

## 6. Phase Completion

最低完了条件:

- Acceptance Criteria達成
- 必須Validation PASS
- blocking finding = 0
- Phase内の各changeにReview Assurance Levelが割り当てられている
- 各changeについて、そのLevelが要求するEvidenceが存在する（L0はIndependent Review Exemptionを含む）
- L1 / L2の対象changeは独立レビューがPASSしている
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
- Review Assurance Level（Phase内の最高Levelと、review対象changeごとのLevel）
- Risk Drivers（Levelを決めた根拠。Forced Level 2条件への該当有無を含む）
- Review Scope（L1で独立監査した範囲）
- Independent Review Exemption（L0の免除理由とResidual Risk）
- Independent Verification Environment（Clean-room Verificationを行った場合: 対象SHA、環境の分離方法、実装担当環境と共有したもの）
- Verification Evidence（command、exit code、結果の安全な要約）
- blocking残件
- advisory残件
- 重要Decision
- Rejected Approaches
- Deliberate Non-Goals
- Residual Risks
- Residual Risk from Reduced Review Scope
- Failure / 再発防止
- 横展開確認の有無と対象
- Knowledge昇格候補
- commit hash
- branch

該当がない項目は `None` または一行の非該当理由でよい。形式を満たすための長文記入はしない。

GitHubへ残す内容は `harness/DEVELOPMENT_STANDARDS.md` の Durable History / Evidence Integrity に従う。
