# OTOMO Remote Review

- Status: **Draft v0.1 / Pilot** — Pilot Product: OTOMO LAB (`t0t005081114-hue/otomo-lab`)
- Approval: Independent Review・Human承認前。承認まではOTOMO共通Ruleとして確定しない（`harness/DEVELOPMENT_STANDARDS.md` §8 No Silent Harness Mutation）
- Related: `harness/REMOTE_REVIEW_SECURITY.md` / `harness/REMOTE_REVIEW_SETUP.md` / `harness/templates/codex-independent-review.md`

## 1. Purpose

スマホからGitHub PRへ `/review` とコメントするだけで、自宅PCのSelf-hosted Runner上で

clean checkout → Deterministic Verification → Codex Independent Review → Evidence → PR Comment

を実行する「Remote Independent Review」の実行経路を定義する。

これは完全自律AIではない。実装・修正・merge判断は行わず、最終判断は人間へ戻す。

```text
ChatGPT → Claude Code → GitHub PR → (スマホ) /review → 自宅PC Self-hosted Runner
→ Deterministic Verification → Codex Independent Review → Evidence → GitHub PR → 人間判断
```

### Supported Trust Model（2026-09-13 Independent Review remediation）

Remote Review v0.1が安全に対象とするのは、

**「private repositoryにおいて、人間が事前に確認し、自宅PCのSelf-hosted Runner上で実行してよいと判断したPR」**

だけである。「信頼済み」の対象には、PRのsource codeそのものだけでなく、以下もすべて含まれる。

- `package.json` / lockfile、依存の追加・更新
- npm lifecycle script（`preinstall` / `postinstall` 等）、install script、build script、test script
- Playwright等、install / build / verification中に実行されるruntime code

つまり「PR作者を信頼している」だけでは足りない。「このPRが導入・変更するcodeとdependency chainを、自宅PCのSelf-hosted Runner上で実行するリスクを、人間が受容している」ことが前提になる。

### Operator Gate

`/review` を投稿する前に、人間が次を確認し、「このコードを自宅PCのRunner上で実行してよい」と判断すること（`harness/REMOTE_REVIEW_SECURITY.md` §4 Operator Rulesにも記載）。

- PRの作者・由来
- 変更されたcode
- `package.json` / lockfileの変更
- install / build / testで実行されるscript
- 依存の追加・更新

### Explicit Non-Goals（Security Claimの範囲）

Remote Review v0.1は、次を提供しない。明示的なNon-Goalであり、実装できていないことを隠さない。

- malicious PRのisolation
- hostile dependencyのsandbox化
- supply-chain compromiseのisolation
- 未知・信頼していない第三者PRの安全な実行
- secure multi-tenant execution
- Codex認証とEvidence生成者の権限分離（Residual Risk。§16参照）

これらを安全に扱えるかのように読める記述が他の文書にあれば誤りである。Future Hardening（ephemeral runner、disposable VM、Codex専用security principal、network isolation等）は§16 Deliberate Non-Goalsに列挙し、今回は実装しない。

## 2. 既存Harnessとの関係

Remote Reviewは新しいReview原則ではなく、既存原則の**実行経路**である。以下を置き換えない。

| 既存Rule | Remote Reviewでの具体化 |
|---|---|
| DEVELOPMENT_STANDARDS §1 Role Separation | Claude Code = 実装、Codex = 独立監査。Remote Reviewは修正しない |
| §4 Implementation Decision Trace | 本書 §16 |
| §5 Review Assurance Level | Remote ReviewはL2（Full Independent Assurance）を満たしうる実行経路の一つ。Remote Reviewを使わないとL2にできない、という関係ではない |
| §5 Independent Review（blocking / advisory） | Codex出力のBlocking / Advisory。FAIL ⇔ Blocking ≥ 1 |
| §5 Clean-room Verification | 対象SHAの明示、fresh checkout、lockfileからのclean install、Productが定義するValidationのみ実行 |
| §5 Verification Evidence | command / exit code / stdout・stderr / 時刻を一次情報として保存。「AIがPASSと言った」だけをEvidenceにしない |
| §5 Evidence Integrity | redaction・size制限後のみ永続化。raw logをPRへ貼らない |
| §9 External Boundary Safety | Codex / npm / GitHub APIの生出力を分類理由へ流さない。transient / permanentを分類 |
| §10 Explicit Invariants / Fail Fast | SHA・PR番号・run IDの不一致、verdict不整合を明示的にINCOMPLETEとする |
| PHASE_WORKFLOW §4 Risk Classification and Independent Review | 監査観点をPromptへ固定 |

Remote Reviewは常に固定PromptのMandatory audit scope（`harness/templates/codex-independent-review.md` §8）全体を監査する。`/review` 以降のfocus textは監査範囲を縮小できない（本書§5 Trigger）。L1の変更へRemote Reviewを使うことは可能だが、その場合は要求より広い監査を行っていることになる。Review Assurance LevelはRemote Review側のscope規則を変更しない。

Remote ReviewのPASSはPhase完了を意味しない。Product側のPhase記録（例: `docs/PHASE_RECORDS.md`）へ採用するかは人間判断とする。

## 3. Ownership

| 対象 | Owner |
|---|---|
| 本仕様（Trigger・Failure分類・Evidence Schema・PR Comment形式・AT） | OTOMO CORE `harness/REMOTE_REVIEW.md` |
| Security Policy | OTOMO CORE `harness/REMOTE_REVIEW_SECURITY.md` |
| Setup Guide（Runner登録等） | OTOMO CORE `harness/REMOTE_REVIEW_SETUP.md` |
| Codex Review固定Prompt（version管理） | OTOMO CORE `harness/templates/codex-independent-review.md` |
| 実際のGitHub Actions workflow | Product `.github/workflows/remote-review.yml` |
| Harness実装・check構成 | Product `scripts/remote-review/` |
| install / lint / typecheck / build / test / Playwright のcommand | Product（`package.json` 等の実態） |
| Product固有review context（読むべき正本・review rule） | Product `scripts/remote-review/config.json` の `codex.context_documents` と `AGENTS.md` |

共通Workflow Templateはまだ置かない。Pilot実装を参照実装とし、AT通過後に2つ目のProductへ展開する時点で共通化を判断する（§16）。

## 4. Architecture

```text
PR comment "/review ..."
  │ issue_comment (created)
  ▼
[authorize] GitHub-hosted — contents: read / pull-requests: write
  - allowlist・author_association・PR判定（workflow if で事前filter + scriptで再検証）
  - PR取得、fork / closed PR拒否、HEAD SHA確定
  - pr-context.json をartifact化、"queued" コメント
  ▼
[review] Self-hosted（Windows x64, label otomo-review）— contents: read のみ
  - 前回workspace削除 → trusted harness（default branch）checkout
  - PR head を exact SHA で checkout（persist-credentials: false）、HEAD一致確認
  - Deterministic Verification（shellなし、config順）
  - diff収集 → codex exec（read-only sandbox）→ checkout無変更確認
  - metadata.json / review-evidence.md / logs → Job Summary → Artifact → cleanup
  ▼
[report] GitHub-hosted — contents: read / pull-requests: write
  - evidenceを検証（run ID・PR番号・head SHA・verdict整合）
  - PR comment 投稿（欠落・不整合時は INCOMPLETE）
```

分割理由: PRのコードを実行するSelf-hosted jobに書き込みtokenを置かない。PRへの書き込みは、PRコードが一切実行されないGitHub-hosted jobだけが行う。

## 5. Trigger

起動条件（すべて満たす）:

1. `issue_comment` の `created`（`edited` は対象外）
2. コメント対象がPR（`issue.pull_request` が存在）。Issueでは起動しない
3. コメント本文が `/review`（小文字、大文字小文字を区別する）で始まる。`/reviewer` は対象外
4. 投稿者が `type: User` かつ allowlist（repository variable `REMOTE_REVIEW_ALLOWED_USERS`）に含まれる
5. `author_association` が `OWNER` / `MEMBER` / `COLLABORATOR`
6. PRがopen、head repositoryがbase repositoryと同一（fork PR拒否）、SHAが40桁hex

認可の責務分担（2026-09-13追記）: GitHub Actionsの式関数（`startsWith()` / `contains()`）は常に大文字小文字を区別しない仕様であり、workflow式の側でcase-sensitiveな判定はできない。そのためworkflow側の `if:` とconcurrency groupの条件は「PRコメントで `/review` らしき文字列である」ことだけを見るcostの事前filterであり、認可境界ではない。allowlist（`REMOTE_REVIEW_ALLOWED_USERS`）のparse・比較、大文字小文字を区別する `/review` 判定、`author_association` の確認は、すべてtrusted script（`authorize.mjs`）内で行う。allowlist変数が不正なJSONであっても、workflow式の評価自体を壊さず、`authorized: false` と理由（`ALLOWLIST_INVALID`）を記録して閉じる（fail closed）。

`/review` 以降の文章は「Codex用追加監査context（重点監査指示）」としてのみ扱う。untrusted dataとしてPromptへ区切って渡し、監査範囲の縮小やverdict規則の変更には使えない。

```text
/review
重点監査：
- auth
- error handling
- mobile runtime
```

禁止:

- comment本文・PR title / body・branch名を `run:` へ `${{ }}` で展開すること
- `eval`、shell経由実行、comment由来の任意コマンド実行
- 未認可ユーザー・Bot・fork PRによるSelf-hosted Runner起動
- `pull_request_target` / `workflow_run` による代替実装

未認可コメントはjobがskipされるだけで、PRへは何も返さない（攻撃者へのfeedbackとspamを避ける）。認可済みユーザーの `/review` がfork / closed PRに対するものだった場合は、拒否理由をコメントする。

## 6. Self-hosted Runner

- 想定: 自宅Windows PC / x64 / GitHub Actions Runner / Node.js / Git / Codex CLI（必要ならPlaywright browsers）
- 推奨label: `self-hosted`, `windows`, `x64`, `otomo-review`
- labelは repository variable `REMOTE_REVIEW_RUNS_ON`（JSON配列）で変更できる
- Runnerは専用Windowsユーザーのsecurity principalで動かす（`harness/REMOTE_REVIEW_SECURITY.md` §3）
- 登録対象は **private repositoryのみ**

### Runner Mode（2026-09-23 Human Decision）

Runner processをどのようにhostするかを **Runner Mode** と呼ぶ。security invariantは専用Windowsユーザーというsecurity principalに属するものであり、process hostingの方式そのものには属さない（`harness/REMOTE_REVIEW_SECURITY.md` §3）。

| Mode | 起動 | 停止 | 本Pilotでの扱い |
|---|---|---|---|
| Interactive | 専用runner userでWindowsにサインインし、runner directoryで `run.cmd` を実行する | その `run.cmd` processを停止する | **採用**（2026-09-23 Human Decision） |
| Service | Windows Serviceとして登録し、serviceを開始する | serviceを停止する | 将来利用可能な代替mode。Formal Runner Acceptanceの必須条件ではない |

どちらのmodeでも共通して必須なもの:

- Runner専用のWindowsローカルユーザー（Administratorsに入れない）のcontextで動くこと
- 普段使いのWindowsユーザーを使わないこと
- そのユーザーにCodex以外の認証情報を置かないこと（`harness/REMOTE_REVIEW_SECURITY.md` §3）
- private repositoryにのみ登録すること
- Operator Gate（§1）が必須であること

Interactive modeについての用語上の注意:

**Interactive modeは「review job単位ではunattended」である。** operatorが `run.cmd` を起動した後は、個々の `/review` 実行ごとに人間の操作を必要としない。「Interactive」が指すのは、runner processが専用userの対話sessionから人間の意思で起動・停止されることであり、Codex実行のたびに人間が応答することではない。`run.cmd` 起動後に対話promptでreview jobが停止する状態は、Interactive modeでもAcceptance不成立として扱う（§20）。

2つのmodeは運用特性が同じではない。差分は `harness/REMOTE_REVIEW_SECURITY.md` §3に記録する。Modeを変更する場合、§20のmode固有checkを再実施する。

## 7. Deterministic Verification

AIにテスト実行判断を任せない。Codex起動前に、Productの `scripts/remote-review/config.json` に定義した順序で、通常のprocessとして実行する。

原則:

- 存在しないcommandを作らない。npm scriptは対象SHAの `package.json` に存在する場合だけ実行する
- dependency installはlockfileからのclean install（npmなら `npm ci`）。実装担当環境のinstall済み依存に依存しない
- commandはtrusted config（default branch）のargv配列。shellを使わない
- 依存checkがPASSしなければ、後続checkはSKIPPED
- `required: true` のcheckのscriptが対象SHAに存在しない場合はFAIL（Validationを削除したPRが見かけ上PASSすることを防ぐ）

各checkのEvidence: `command`, `started_at`, `finished_at`, `duration_ms`, `exit_code`, `timed_out`, stdout / stderr（redaction後のlog file）, `status`, `skip_reason`, `failure`。

| status | 意味 |
|---|---|
| PASS | exit code 0 |
| FAIL | 対象コードに起因する非0 exit、またはrequired checkが実行不能 |
| SKIPPED | scriptが未定義、または依存checkが非PASS。**PASSではない** |
| INFRA_ERROR | processを起動できない、timeout、network / file lockによるinstall失敗など、結果を判定できない |

### Verification Checks Manifest Cross-Check（report job、2026-09-14 Independent Re-review remediation、CORE-RR-RR-002）

`summarizeVerification`（§10 `verification.status`。判定条件は下記Required Check PASS Requirementを参照）は、Evidenceの `checks[]` に列挙されているcheckだけを見て判定する。これは、trusted `config.json` が `required: true` と定義したcheckがEvidence側の `checks[]` から欠落している場合、同一idのcheckで `required` / `label` フラグがtrusted manifestと一致しない場合、またはEvidence側に重複・想定外のentryが混入している場合までは検出しない — `checks[]` 自体がtrusted definitionを正しく反映しているかどうかは、`summarizeVerification` の判定範囲外である。

review job自身のcheck実行（§7）はtrusted configの順序で決定論的に走るが、`metadata.json` の `checks[]` はEvidenceとして自己申告される値であり、report jobがそのまま信用してはならない（DEVELOPMENT_STANDARDS §5 Evidence Integrity）。Context Documents Manifest Cross-Check（§9）と同じ構造で、report jobは自身が独立にcheckoutした同一trusted commit（`ref: github.sha`）から `scripts/remote-review/config.json` を読み直し、Evidenceの `checks[]` をこのtrusted `config.checks` に対して照合する。次のいずれかがあれば、Evidenceの `verification.status` / `verdict` をそのまま採用せず、PASS/FAILとしてPRへ投稿しない（INCOMPLETE、§8）。

- trusted manifestにある `required: true` checkがEvidence側の `checks[]` に存在しない（id一致するentryが無い）
- 同一idのcheckで `required` フラグがtrusted manifestと一致しない
- 同一idのcheckで `label` がtrusted manifestと一致しない
- Evidence側にtrusted manifestに存在しないidのcheckがある、またはEvidence側で同一idが重複している

trusted manifestで `required: true` のcheckが、Evidence側で `PASS` 以外のstatus（`FAIL` / `INFRA_ERROR` / `SKIPPED`）になっていること自体は、この照合の対象ではない（下記参照）。

どのcheckが `required: true` かは、この照合対象のtrusted `config.json`（Product `scripts/remote-review/config.json`）自体をSource of Truthとする。本書・report job実装のいずれにも、必須check一覧をハードコードしない — Productがcheck構成を変更した場合、trusted configを読み直すだけでcross-checkが追従する。

**「検証（deterministic check）自体がFAILした」ことと「Evidenceが欠落・改ざん・不整合で判定不能」なことは異なる状態であり、この照合が検出するのは常に後者である**（2026-09-15追加、CORE-RR-FRR-002 semantic regression remediation。以前の記述・実装は、trusted manifestで `required: true` のcheckがEvidence側で `PASS` 以外である場合をすべてこの照合の対象＝Evidence整合性問題として扱っており、正常に実行・記録された `FAIL` / `INFRA_ERROR` / まだ非構造的理由で `SKIPPED` のままの状態までEvidence corruptionと混同していた）。

- **definition integrity**（この照合の責務）: trusted manifestにある `required: true` checkがEvidence側の `checks[]` に存在しない、`required` / `label` フラグが一致しない、Evidence側に重複/想定外entryがある、または（次項）`PASS` と自己申告している場合にその実行結果がtrusted definitionの通りではない — これらはこの照合が検出し、INCOMPLETEとする
- **execution result semantics**（この照合の対象外・§7/§8の責務）: definitionと一致する `FAIL` は §8 のVerification FAILとしてVERDICT: FAILへ、definitionと一致する `INFRA_ERROR` は §8 のInfrastructure FAILEDとして扱う。`SKIPPED` のまま残ったrequired checkは、この照合ではEvidence整合性問題としないが、`summarizeVerification`（`lib/evidence.mjs`）が「required checkは全件PASSでなければVerificationはPASSにならない」という不変条件を持つため、他のrequired checkがPASSしていてもVERDICT: PASSへは到達しない（下記Required Check PASS Requirementを参照）

この責務分離により、「lintが実際にFAILした」場合は正しく `VERDICT: FAIL` として投稿され、「Evidenceが改ざん・欠落・定義不一致」の場合だけがINCOMPLETEになる。

### Canonical Command Binding（report job、2026-09-15 Independent Re-review remediation、CORE-RR-FRR-001 / CORE-RR-FINAL-001）

上記の照合は、Evidenceの `id` / `required` / `label` / `status` がtrusted manifestと一致することまでしか確認していなかった。Evidence側の `checks[]` はEvidenceとして自己申告される値であり、あるcheckが実際にtrusted `config.json` が定義したcommandを実行して得られた結果であることまでは確認していなかった（例えば `lint` のEvidence `command` を別のcommandへ差し替えても、id/required/label/statusさえ一致していればこの照合を通過しえた）。

CORE-RR-FRR-001時点の実装は、この検証を `status: PASS` を自己申告するcheckだけに限定していた。Independent Re-reviewは、`status: FAIL` かつ `failure` が正当な `VERIFICATION_FAILURE` に見えるが `command` が別checkのものに差し替えられている、といったEvidenceがこの検証を素通りすることを指摘した（CORE-RR-FINAL-001）。`command` は「そのcheckが何を実行したかの自己申告」であり、その自己申告の信頼性は `status` の値によらず同じだけ疑わしいため、statusにかかわらず検証する必要がある。

これを閉じるため、report jobは各checkについて、`runChecks()` / `classifyCheckResult()`（`lib/checks.mjs` / `lib/process.mjs`）が実際に生成しうる状態遷移をSource of Truthとして、次をfail-closedに検証する（いずれかを満たさなければEvidence全体を採用せず、INCOMPLETEとする）。

- **command binding（全status共通）**: `command` が `null` でない場合、trusted `config.checks` の同一checkから導出した canonical invocation と一致する。`npm_script` checkは `["npm", "run", <npm_script>]`、argv checkはtrusted `command` 配列をそのまま用いる。この導出は `lib/checks.mjs` の `canonicalCommand()` 一箇所だけで行い、report job・review job・testのいずれにもcommand文字列を別途ハードコードしない（review job自身の実行 `planCheck` も同じ関数を使う）
- **PASS**: `command` が非null（上記に一致）、`exit_code === 0`、`timed_out === false`、`skip_reason === null`、`failure === null`
- **FAIL**: `failure` が非null、`timed_out === false`（`classifyCheckResult()` はtimeoutを常にINFRA_ERRORとして分類し、FAILにはしない）。加えて、`command` が非null（実行されたFAIL）なら `skip_reason === null` かつ `exit_code` が非null・非ゼロの整数、`command` が `null`（required scriptが構造的に存在しないFAIL）なら `skip_reason` が非null かつ `exit_code === null`
- **INFRA_ERROR**: `command` が非null（skip経路からは到達しない）、`skip_reason === null`、`failure` が非null。`exit_code` / `timed_out` はsub-case（invocation準備失敗・spawn失敗・timeout・signal終了）によって正当に異なるため、単一の期待値へは固定しない
- **SKIPPED**: `command === null`、`skip_reason` が非null、`exit_code === null`、`timed_out === false`、`failure === null`

`required: false` のcheckについても同様に検証する（自己申告そのものの信頼性を確認する照合であり、`required` の有無で対象を絞らない）。

### Required Check PASS Requirement（`lib/evidence.mjs` summarizeVerification、2026-09-15 Independent Re-review remediation、CORE-RR-FRR-002）

`summarizeVerification`は以前、「1件もFAILせず・INFRA_ERRORもなく・PASSが1件以上あれば `PASS`」と判定しており、Evidence側の各checkが持つ `required` フラグを見ていなかった。このため、`required: true` のcheckが（非構造的理由で）`SKIPPED` のまま残っていても、別の `required: true` checkが `PASS` していれば全体がVerification `PASS` に到達しえた（§7表の「SKIPPEDは**PASSではない**」という個別check単位の原則が、集計レベルでは保証されていなかった）。

修正後は、Evidenceに `required: true` のcheckが1件以上ある場合、それら**全件**が `PASS` でなければVerificationは `PASS` にならない（`UNKNOWN` になる）。`required: true` のcheckが1件もないEvidence（想定されない構成だが）は、従来どおり「PASSが1件以上」で判定する。

## 8. Failure Classification

Failureは **Category** と **Transience** の2軸で分類する。

| Category | 例 | 典型的Transience |
|---|---|---|
| `VERIFICATION_FAILURE` | lint / typecheck / build / test の非0 exit、required scriptの欠落 | PERMANENT |
| `INFRASTRUCTURE_FAILURE` | timeout、npm registry到達不可、harness内部エラー、review job消失 | TRANSIENT / UNKNOWN |
| `AUTHENTICATION_FAILURE` | Codex未ログイン・login期限切れ | PERMANENT（人間の再ログインが必要） |
| `ENVIRONMENT_FAILURE` | Codex CLI / npm未検出、Node.jsがengines未満、HEAD SHA不一致、EPERM、checkoutの改行変換、Codexのtool host起動失敗 | PERMANENT / UNKNOWN |
| `GITHUB_API_FAILURE` | PR取得・コメント投稿の失敗 | 5xx / 429 / network = TRANSIENT、4xx = PERMANENT |
| `CODEX_FAILURE` | Codex非0 exit、rate limit、VERDICT欠落・矛盾、read-only境界違反 | TRANSIENT / UNKNOWN / PERMANENT |
| `CONTEXT_FAILURE` | `required: true` のcontext documentがreviewed head SHAで存在しない・空・読めない（§9 Required / Optional Context Documents） | PERMANENT |

| Transience | 意味 |
|---|---|
| TRANSIENT | 再実行で解消しうる（再度 `/review`） |
| PERMANENT | 設定・認証・コードの修正が必要。再試行しても解消しない |
| UNKNOWN | 判定できない |

RetryはTRANSIENTに限る（DEVELOPMENT_STANDARDS §9）。Pilotの自動RetryはGitHub APIのtransient失敗のみ。

### Status and Verdict

- **Verification**: `FAIL`（FAILが1件以上）/ `UNKNOWN`（INFRA_ERRORがある、またはPASSが0件）/ `PASS`
- **Infrastructure**: `FAILED`（INFRA_ERROR check、またはstage failureが1件以上）/ `HEALTHY`
- **Codex**: verdict `PASS` / `FAIL`、または status `FAILED` / `NOT_RUN`

| 条件 | VERDICT |
|---|---|
| Verification FAIL、または Codex FAIL | **FAIL** |
| Verification PASS かつ Codex PASS かつ Infrastructure HEALTHY | **PASS** |
| それ以外（Codex利用不可、INFRA_ERROR、evidence欠落・不整合など） | **INCOMPLETE** |

INCOMPLETEは「判定できなかった」ことを示し、PASSとして扱わない。

```text
Verification: FAIL · Codex: PASS · Infrastructure: HEALTHY      → VERDICT: FAIL
Verification: UNKNOWN · Codex: FAILED · Infrastructure: FAILED → VERDICT: INCOMPLETE
```

## 9. Codex Independent Review

Codexは修正者ではなく独立監査者である。

禁止: source code修正、auto fix、commit、push、merge、force push、PR作成、dependency install、build / test再実行、network access。

強制手段（Pilot実装）:

- `codex exec --sandbox read-only --ephemeral --ignore-user-config --ignore-rules -c approval_policy="never"`、Promptはstdin
- 実行前後で `git status --porcelain` を比較し、変化があれば `CODEX_FAILURE`（境界違反）
- checkoutに認証情報を残さない（`persist-credentials: false`）。self-hosted jobのtokenは `contents: read` のみ
- Codexがtool host（コマンド実行基盤）を起動できなかった場合は、exit 0で形式どおりのVERDICTを返していてもreviewとして採用しない（`ENVIRONMENT_FAILURE` / INCOMPLETE）。2026-09-13のlocal E2Eで、Codexがcheckoutを一切読めないままVERDICTを返すことを確認したため
- **Grounding check（TOOL_CHECK）**: harnessはcheckoutの `.git/` 内にランダム値を書き、Promptにはそのfile pathだけを渡す。Codexはread-onlyコマンドでその値を読み、Verification Evidenceの最終行に `TOOL_CHECK: <値>` を出力する。行が無い・値が違う → `CODEX_FAILURE`、`TOOL_CHECK: UNAVAILABLE` → `ENVIRONMENT_FAILURE`。いずれもreviewとして採用しない（INCOMPLETE）。2026-09-13のlocal E2Eで、tool hostが正常でもsandbox policyにより全コマンドが拒否されたまま、Codexが `VERDICT: PASS` を返したため
- Windowsでは `-c windows.sandbox=elevated` を付ける（Productの `config.json` の `codex.windows_sandbox`）。`--ignore-user-config` によりuser configのsandbox設定が読まれず、既定のままではread-only sandboxの全コマンドが `blocked by policy` になることを確認した

Codexへ渡す情報:

- PR title / description（size制限あり）、base branch、base SHA、head SHA
- diff（redaction・size制限・lockfile除外）、diff stat、変更file一覧、harness-sensitive file一覧
- Productの正本・review rule（`codex.context_documents` をCodexがcheckoutから読む）
- Deterministic Verification Evidence（表と、失敗checkのlog末尾抜粋）
- `/review` 後の重点監査指示

Context量の制御（Pilot既定値）: diff 150 KB、PR body 8 KB、focus 2 KB、失敗log抜粋 6 KB/check、変更file一覧 500件。切り詰めた場合はPromptの Context notes に明示する。

### Required / Optional Context Documents（2026-09-13 Independent Review remediation）

`codex.context_documents` の各項目は `{ path, required }` を持つ。`required: true` は「レビューに必須のSource of Truth」（例: 製品のrequirements・specification・review rule）、`required: false` は「存在すれば読む補助文書」を意味する。

harnessはCodex起動前に、trusted checkout（authorized head SHAへexact checkoutした後）上で各documentを確認する。

- path
- 存在するか（`exists`）
- 通常fileとして読めるか（`readable`）
- 内容が空でないか

いずれかの `required: true` documentがこれを満たさない場合:

- `CONTEXT_FAILURE`（Failure Category、§8）をstageする
- VERDICTはPASSにならない（infrastructureがFAILEDになり、§8の表により少なくともINCOMPLETEになる）
- Codex自身のVERDICT出力は、完全なIndependent Reviewの結果として扱わない

`required: false` のdocumentは、存在すれば読み、存在しなければ（`MISSING` / `SKIPPED`）そのままスキップしてよい。いずれの場合もEvidenceに `path` / `required` / `status`（`PRESENT` / `MISSING` / `EMPTY` / `UNREADABLE`）/ `size` を残す（§10 Schema）。

この確認は「checkout時点でfileが存在し読めた」ことの確認であり、「Codexが実際にそのfileを読んだ」ことの証明ではない（次項参照）。

### Source of Truth Manifest Consistency（2026-09-14 Independent Re-review remediation、CORE-RR-IR-002）

上記の`required: true`presence checkは、`codex.context_documents`に**実際に列挙されているentry**についてしかfail-closedにならない。Productが「レビュー前に読むべき」と定義したSource of Truth文書（Product `AGENTS.md` §2等）が、そもそもこのmanifestへ追加されなかった場合、または`required: true`から`required: false`へ静かに格下げされた場合は、presence checkの対象自体が存在しないため検出できず、Evidence上は自己整合的なまま`VERDICT: PASS`に到達しうる。これがCORE-RR-IR-002の指摘した欠落である。

これを閉じるため、次を恒久的な運用規則とする。

- Product `AGENTS.md`（またはこれに相当するReview entrypoint文書）が「レビュー前に読むべき」と定義したfile-backedなSource of Truthは、`scripts/remote-review/config.json`の`codex.context_documents`に`required: true`として存在しなければならない
- `context_documents`の`required`をtrueからfalseへ変更する、またはmanifestからentryを削除するのは、Product側の「読むべきSource of Truth」定義（`AGENTS.md`等）を同じ変更の中で先に、または同時に書き換えてからにする。manifest側だけを単独で緩めない
- Productはこの一致を回帰テストで検証する（2026-09-14 CORE-RR-IR-002再remediationで実装。`otomo-lab`の実装: `scripts/remote-review/lib/source-of-truth.mjs`の`checkSourceOfTruthManifestConsistency`が、Product `AGENTS.md`の該当tableから必須path一覧を抽出し、`config.json`の`codex.context_documents`と照合する。`scripts/remote-review/test/source-of-truth.test.mjs`が、必須entry削除・`required: true → false`・想定外entry追加・duplicate entryの4種のmutationすべてでtestがFAILすることを検証する）。この抽出は、AGENTS.mdの自由な文章（prose）全体をparseするのではなく、AGENTS.md自身が定義する固定table構造（見出しと`| \`path\` | ... |`形式の行）だけに依存し、table自体が見つからない場合は「必須文書ゼロ」と静かに解釈せず例外を投げる
- どのProduct文書が「レビュー前に読むべき」かの判断自体は、既存Harness（本書・Product `AGENTS.md`・`docs/DEVELOPMENT_STANDARDS.md`等）とProduct固有ルールから行う。本書はmanifestとProduct定義を一致させることだけを要求し、どの文書が必須かをCORE側で決定しない

### Context Documents Manifest Cross-Check（report job、2026-09-14 Independent Re-review remediation、CORE-RR-IR-002）

review jobの`CONTEXT_FAILURE`判定（上記）は、review job自身が信頼したconfig（trusted checkoutの`codex.context_documents`）に基づいて生成されたEvidence（`metadata.json`の`context_documents[]`）を前提にしている。しかし、Evidenceは自己申告であり、report jobがそれをそのまま信用してはならない（DEVELOPMENT_STANDARDS §5 Evidence Integrity）。

report jobはreview jobとは別のGitHub-hosted checkoutで、review jobが信頼したのと同じtrusted commit（`ref: github.sha`）を独立にcheckoutする。report jobはこのtrusted checkoutから`scripts/remote-review/config.json`を自分でも読み込み、Evidenceの`context_documents[]`をこのtrusted manifestに対して照合する。次のいずれかがあれば、Evidenceの`verdict`をそのまま採用せず、PASS/FAILとしてPRへ投稿しない（INCOMPLETE、§8）。

- `context_documents`が`null`（trusted manifestが空でない場合）
- trusted manifestにあるentryがEvidence側に存在しない（path一致するentryが無い）
- 同一pathのentryで`required`フラグがtrusted manifestと一致しない
- trusted manifestで`required: true`のentryが、Evidence側で`PRESENT`以外のstatusになっている
- Evidence側にtrusted manifestに存在しないpathのentryがある、またはEvidence側で同一pathが重複している

この照合はreview job側のpresence check（fileが実際に存在し読めたか）を置き換えない。両方が独立に成立して初めて、required context documentが「manifestにも列挙され」かつ「checkout時点で読めた」ことになる。

### TOOL_CHECK Responsibility（境界の明確化）

`TOOL_CHECK`（grounding check）が証明するのは「Codexがこのcheckoutに対してread-onlyコマンドを実行できた」ことだけである。次のいずれも証明しない。

- requirementsを読んだこと
- 全Source of Truthを読んだこと
- 全changed fileを確認したこと
- 正しい仕様を理解したこと

この責務境界は、本書・`harness/REMOTE_REVIEW_SECURITY.md`・`harness/templates/codex-independent-review.md`で一貫させる。「必要な観測ができたか」（TOOL_CHECK）と「必要な資料が実在したか」（Required Context Documents）と「Codexが実際に何を読んだと述べているか」（Verification Evidence欄の記述、人間が確認する）は、別々に記録し、混同しない。

### Model / Prompt Provenance（Advisory）

Evidenceに次を記録する（§10 Schema）。取得できない項目は `UNKNOWN` 等の明示値とし、推測しない。

`requested_model` / `resolved_model`（Codex CLIは実際に解決したmodel名を返さないため常に `UNKNOWN`）/ `reasoning_effort`（harnessは指定していないため `UNKNOWN`）/ `approval_policy` / `ignore_user_config` / `ignore_rules` / `effective_sandbox`（read-only、Windowsでは `windows.sandbox` 設定を含む）/ `prompt_version` / `prompt_hash`（実際にstdinへ渡したprompt全体のSHA-256）。

**report側検証（2026-09-14 Independent Re-review remediation）**: `codex.status` が `COMPLETED` のEvidenceについて、report jobは上記provenance fieldの存在・型を検証する（`prompt_hash` はSHA-256 hex、64桁の16進文字列であることを検証する）。`resolved_model` / `reasoning_effort` が仕様上常に `"UNKNOWN"` になることは、この検証と矛盾しない——`"UNKNOWN"` は有効な文字列値として受理され、欠落や型不一致とは区別する。検証に失敗した場合はEvidence全体を採用せず、INCOMPLETEとして扱う（§10 report jobのEvidence検証）。

### Codex Evidence Semantic Invariants（report job、2026-09-14 Independent Re-review remediation、CORE-RR-RR-001）

上記provenance検証は各fieldの型・非空文字列であることまでしか確認しない。Independent Re-reviewは、`codex.status` / `codex.verdict` / provenance field間の**意味的整合**（例えば `status: FAILED` と `verdict: PASS` が同居していないこと、`workspace_unchanged` や `tool_check` が実際に安全な値であること）までは検証していない欠落を指摘した（CORE-RR-RR-001）。Evidenceは自己申告であり、report jobは値の型だけでなく、review jobの実装（本書§9の強制手段）が実際に保証する不変条件そのものを、Evidenceに対しても独立に確認しなければならない（DEVELOPMENT_STANDARDS §5 Evidence Integrity）。

report jobは、`codex.status` が `COMPLETED` のEvidenceについて、少なくとも次をfail-closedに検証する（いずれかを満たさなければEvidence全体を採用せず、INCOMPLETEとする — VERDICTを `PASS`/`FAIL` としてPRへ投稿しない）。

- `codex.status` が `COMPLETED` 以外（`FAILED` / `NOT_RUN`）のとき、`codex.verdict` は `null` 以外であってはならない（非null verdictは、review job実装上 `COMPLETED` と同時にしか設定されない — §9 出力形式）
- `codex.tool_check` が `VERIFIED` であること（Grounding checkが証明されていないreviewは採用しない、§9 TOOL_CHECK Responsibility）
- `codex.workspace_unchanged` が `true` であること（read-only境界が破られていないreviewは採用しない）
- `codex.approval_policy` が `"never"` であること、`codex.ignore_user_config` / `codex.ignore_rules` がともに `true` であること（§9 強制手段どおりに起動されたことの確認）
- `codex.effective_sandbox` が read-only設定であることを示す文字列であること。report jobはさらに、自身が独立にcheckoutしたtrusted `config.json` の `codex.windows_sandbox` と、Evidence自身の `run.runner_os`（harness-controlled、PR由来ではない）から review job実装が構成するはずの値を再現し、Evidenceの `effective_sandbox` と一致することを確認する（platform / trusted config由来の期待値との整合）
- `codex.blocking_count` / `codex.advisory_count` が非負整数であること。`codex.verdict === "PASS"` なら `blocking_count === 0`、`codex.verdict === "FAIL"` なら `blocking_count >= 1` であること（§9 出力形式のVERDICT / Blocking Findings整合と同じ不変条件を、Evidence側でも再確認する）
- Codex review本文（`codex-review.md`）を report jobが独立に再parseし、その結果（verdict / blocking_count / advisory_count）が `metadata.json` の `codex.verdict` / `codex.blocking_count` / `codex.advisory_count` と一致すること。`codex.status` が `COMPLETED` であるにもかかわらず `codex-review.md` が欠落・読めない場合も同様にINCOMPLETEとする

未決定のまま拡大解釈しない: 上記は現行のreview.mjs実装が実際に構成する値・現行Evidence Schema 1.1が持つfieldからのみ導出しており、新しい `codex` fieldや新しいverdict値を追加しない。`resolved_model` / `reasoning_effort` の `"UNKNOWN"` 許容（上記provenance検証）と、この節の検証は独立であり、互いに矛盾しない。

### Prompt Template

- 正本: `harness/templates/codex-independent-review.md`（先頭commentに `version:`）
- ProductはCOREと同一内容のコピーを使う。変更はCOREで行い、versionを上げてからProductへ反映する
- Pilotでは「CORE sibling checkoutが存在する場合、Productコピーと一致すること」をテストで検査する
- 監査対象（最低限）: requirement deviation / regression / missing validation / boundary violation / auth・permission / security / runtime failure / error handling / state consistency / data integrity / accessibility / responsive behavior / test gap / hidden assumptions / scope creep
- untrusted data（PR本文・diff・log・focus）はnonce付きの区切りで渡し、指示として扱わないことをPromptで固定する

### 出力形式

```text
VERDICT: PASS | FAIL

## Blocking Findings
## Advisory Findings
## Verification Evidence
## Residual Risks
## Recommended Next Action
```

- Finding ID: `PR<PR番号>-IR-<3桁連番>`（例 `PR12-IR-001`）。Blocking / Advisoryで通し番号。OTOMO LABの既存慣行 `<scope>-IR-NNN`（例 `PH01-IR-001`）に揃え、PR番号をscopeとする
- Finding fieldはProductのreview rule（OTOMO LAB `AGENTS.md` §6）に従う
- VERDICT行が無い、複数で矛盾、PASSなのにBlockingあり、FAILなのにBlockingなし → `CODEX_FAILURE`（INCOMPLETE）
- 上記5個のsectionは、それぞれちょうど1回だけ出現しなければならない（2026-09-15追加、CORE-RR-FRR-002）。**同一sectionの重複は `CODEX_FAILURE`（INCOMPLETE）とし、PASS/FAIL verdictを採用しない** — parserが以前、同名sectionを「後勝ち」で上書きしていたため、先行する `## Blocking Findings` に実際のFindingが記載され、後続の（空の）`## Blocking Findings` がそれを上書きした場合、`VERDICT: PASS` と矛盾なくparseできてしまっていた（section本文はチェックせず見出しの出現回数だけ見るため、`## blocking findings` のような大小文字違いの重複も同じ扱いで検出する）

### Verification Capability Principle（Draft / CORE昇格候補）

2026-09-13のPilot local E2Eで、Codexが必要なtool（read-onlyコマンド）を使えない状態のまま、文章上は形式どおりの `VERDICT: PASS` を返した。原因は2種類あった（tool hostの欠落、Windows sandbox設定の欠落）。harnessはexit codeと出力形式だけを見ていたため、当初はreviewとして受け入れていた。Failure原本: `otomo-lab` の `docs/DECISIONS_AND_FAILURES.md`（`[2026-09-13 / Remote Review pilot] Codex returned a verdict without being able to run any tool` とそのaddendum）。

ここから次の原則を導く。

1. **自己申告はverification capabilityの証明にならない**: AI reviewerの「確認した」「PASS」という記述、exit code 0、出力形式の正しさは、そのreviewerが検証に必要な観測（fileを読む・commandを実行する）を実際に行えたことを証明しない
2. **観測能力は外部から検証できる形で確認する**: reviewerが実際にtoolを使わなければ得られない値（harnessだけが知るnonce等）を課し、harness側で照合する。人手による事前確認（Setup Guideのsmoke test）も、reviewerの回答文ではなく実際のcommand出力で判定する
3. **capabilityを確認できないreviewはPASSにしない**: 確認失敗・未確認はINCOMPLETEとし、verdictを採用しない。既知の原因を列挙したpattern検出だけに依存せず、原因が未知でも照合で拒否できる形にする
4. **review resultとreview infrastructure healthを分離する**: 「対象が正しいか（Verification / Codex verdict）」と「reviewを成立させる基盤が健全か（Infrastructure）」を別の状態として記録・表示し、基盤の不健全を対象のPASS / FAILへ混ぜない

Pilotでの実装範囲:

- 原則2・3: `TOOL_CHECK` をreview実行中に課し、結果を採用する前にharnessが照合する。reviewとは別processで開始前にprobeするpreflightは未実装（将来候補）
- 原則4: §8 の Verification / Codex / Infrastructure の状態とVERDICT表
- 限界: `TOOL_CHECK` は「commandを実行できた」ことの証明であり、「必要なfileをすべて読んだ」ことの証明ではない（§16 Residual Risks）

本原則は他のAI検証経路（CI上のAI review、Phase Independent Review等）にも適用できる可能性が高いため、`learning/KNOWLEDGE_PROMOTION.md` に従うCORE昇格候補とする。`harness/DEVELOPMENT_STANDARDS.md` と `learning/PROMOTION_LOG.md` への反映は、Independent ReviewとHuman承認の後に行う（本変更では行わない）。

## 10. Review Evidence

原則: Evidenceをrepositoryへcommitしない。

保存先:

1. GitHub Actions Artifact `remote-review-evidence`（retention 14日）
2. GitHub Actions Job Summary（`review-evidence.md` の内容）
3. PR Comment（要約のみ）

| Artifact file | 内容 |
|---|---|
| `metadata.json` | 機械可読Evidence（Schema 1.1） |
| `review-evidence.md` | 人間向けEvidence全体 |
| `<check-id>.log` | 各checkのcommand・時刻・exit code・stdout / stderr（redaction済み）。例 `install.log`, `lint.log`, `typecheck.log`, `build.log` |
| `codex-prompt.md` | Codexへ渡したPrompt（redaction済み） |
| `codex.log` | Codex CLIのstdout / stderr（redaction済み） |
| `codex-review.md` | Codexの最終出力（redaction済み） |

test / Playwrightを導入したProductでは `test.log` / `playwright.log`（必要なら `playwright-report`）が加わる。

### Redaction

永続化前に置換する（best effort）: GitHub token、OpenAI / Anthropic形式のkey、AWS access key、Google API key、Slack token、npm token、JWT、private key block、Authorization / Cookie / Set-Cookie header、Bearer / Basic credential、URL内の `user:password@`、署名付きURL・token系query parameter、`password=` 等の代入、実行環境の機密名環境変数の値。

自動redactionだけで安全とみなさない（DEVELOPMENT_STANDARDS §5 Evidence Integrity）。raw logはPR commentに貼らない。checkやCodexの子processには、機密名の環境変数・`GITHUB_*`・`ACTIONS_*` を渡さない。

### Schema 1.1（`metadata.json`）

| Field | 型 | 内容 |
|---|---|---|
| `schema_version` | `"1.1"` | |
| `generator` | object | `name`, `template_version`, `config_schema_version` |
| `repository` | string \| null | `owner/repo` |
| `pull_request` | object \| null | `number`, `title`, `url`, `base_ref`, `base_sha`, `head_ref`, `head_sha` |
| `request` | object \| null | `requested_by`, `comment_id`, `comment_url`, `requested_at`, `focus_provided` |
| `run` | object | `id`, `attempt`, `url`, `runner_name`, `runner_os`, `node_version` |
| `timing` | object | `started_at`, `finished_at`, `duration_ms` |
| `checks[]` | array | 下表 |
| `verification.status` | `PASS` \| `FAIL` \| `UNKNOWN` | §8 |
| `codex` | object | `status`（`COMPLETED` / `FAILED` / `NOT_RUN`）, `verdict`（`PASS` / `FAIL` / null）, `problem`, `cli_version`, `exit_code`, `duration_ms`, `timed_out`, `prompt_bytes`, `workspace_unchanged`, `tool_check`（`VERIFIED` / `MISSING` / `MISMATCH` / `UNAVAILABLE` / null）, `blocking_count`, `advisory_count`, `requested_model`, `resolved_model`, `reasoning_effort`, `approval_policy`, `ignore_user_config`, `ignore_rules`, `effective_sandbox`, `prompt_version`, `prompt_hash` |
| `infrastructure` | object | `status`（`HEALTHY` / `FAILED`）, `failures[]`（`stage`, `category`, `transience`, `reason`） |
| `review_context` | object \| null | `changed_files_count`, `diff_bytes`, `diff_truncated`, `harness_sensitive_changes[]` |
| `context_documents[]` \| null | array | `path`, `required`, `status`（`PRESENT` / `MISSING` / `EMPTY` / `UNREADABLE`）, `size` |
| `verdict` | `PASS` \| `FAIL` \| `INCOMPLETE` | §8 |
| `artifacts[]` | string[] | Artifact内のfile名 |
| `redaction` | object | `applied`, `method` |

`schema_version` を1.0から1.1へ上げた変更: `context_documents[]` の追加、`codex` へのModel / Prompt Provenance fieldの追加。既存fieldの意味は変えていない。

`checks[]`:

| Field | 内容 |
|---|---|
| `id`, `label`, `required` | config由来 |
| `command` | 実行command（redaction済み）。未実行時null |
| `status` | `PASS` / `FAIL` / `SKIPPED` / `INFRA_ERROR` |
| `exit_code` | number \| null |
| `started_at`, `finished_at`, `duration_ms`, `timed_out` | |
| `skip_reason` | string \| null |
| `failure` | `{ category, transience, reason }` \| null |
| `log_file`, `output_truncated` | |

`reason` にはraw stdout / stderrを含めない（安全な固定文言と分類のみ）。

report jobはevidenceを採用する前に、run ID・PR番号・head SHAの一致、enum値、checksから再計算したverdictとの整合を検証する。不一致ならevidenceを使わず、INCOMPLETEを投稿する。

### Stale Review Protection（2026-09-13 Independent Review remediation）

authorize時点のhead SHAでreviewを実行するため、review実行中にPRへ新しいcommitがpushされると、古いSHAに対する結果を投稿しようとする可能性がある。PRコメントにはSHAが表示されるが、人間がcurrent HEADへのPASSと誤認しうる。

report jobは、PRコメントを投稿する直前にGitHub APIから現在のPR HEAD SHAを再取得し、reviewしたSHAと比較する。

- 一致: 通常どおり投稿する
- 不一致: 投稿するVERDICTを `INCOMPLETE` に強制する（元のverdictはPASS/FAILであっても、current HEADに対する有効な結果として提示しない）。コメントに reviewed SHA・current SHA・staleである旨を明示する
- **取得不能（2026-09-14 Independent Re-review remediation、Advisory）**: GitHub APIの応答に`head.sha`が欠落している、または40桁hexとして不正な場合、「reviewしたSHAと一致した」ものとしてfail-openしてはならない。この場合も投稿するVERDICTを`INCOMPLETE`に強制する（reason: head SHA未確認）。「一致しない」と確定できたstaleケースと、「一致するかどうか確認できない」ケースは別の条件だが、いずれもPASS/FAILをそのまま投稿しない点で扱いは同じ

`metadata.json`（Artifact）自体は書き換えない。stale判定・head SHA未確認判定はいずれもreport job（PR投稿の直前）だけの責務であり、Evidence Schemaへ新しいverdict値は追加しない。

**Residual Risk（TOCTOU、2026-09-14 Independent Re-review remediation）**: 上記の再取得（GET）とPRコメント投稿（POST）はatomicではない。この2つのAPI呼び出しの間に新しいcommitがPRへpushされる可能性は残り、GitHub REST APIのissue commentにはcompare-and-swapに相当する原子的操作がない。この窓は本Pilotでは埋めない（許容するResidual Risk、§16）。人間は投稿されたコメントのreviewed SHAを確認し、必要なら再度 `/review` する。

### Durable History Boundary（Advisory）

Remote Reviewの成果物（Artifact）は短期保存（Evidence 14日、PR context 7日）であり、OTOMO COREのDurable History（Phase Record、Decision Trace、Failure Log等）そのものではない。Remote Reviewの結果をPhase EvidenceやDecision Evidenceとして正式採用する場合は、Artifactが失効する前に、Product側の既存Durable History（例: `docs/PHASE_RECORDS.md`）へ人間が昇格させる必要がある。Evidenceの自動commitは行わない（禁止のまま）。PRコメントには最低限のcheck名・status・exit code・reviewed SHAだけを残し、巨大なraw logは残さない。

## 11. PR Comment

```markdown
## Remote Independent Review

**VERDICT: PASS | FAIL | INCOMPLETE**

Verification: … · Codex: … · Infrastructure: …

Commit reviewed:
`<HEAD SHA>`

（現在のPR HEADがreviewしたSHAと異なる場合はここに stale 警告と両方のSHAを表示し、VERDICTは INCOMPLETE にする。§10 Stale Review Protection）

### Verification

| Check | Result |
|---|---|
| Install | PASS |
| Lint | PASS |
| Typecheck | PASS |
| Build | PASS |
| Test | SKIPPED |
| Playwright | SKIPPED |

### Blocking Findings

- PR12-IR-001 — …
（Details は折りたたみ）

### Advisory Findings

- None

### Residual Risks

- …

### Infrastructure

HEALTHY | FAILED（FAILED時は stage・category・transience・理由）

### Evidence

- GitHub Actions Run: <url>
- Artifact: `remote-review-evidence`
```

- 長大なlogは貼らない。本文は60,000文字以内に切り詰める
- Codex / PR由来テキストの `@mention` は無効化する
- review-harness file（`.github/`、`scripts/remote-review/`、`AGENTS.md`、`CLAUDE.md`、`package.json` 等）を変更するPRには警告を出す（harnessの変更はmerge後にしか効かないため、人間が確認する）
- 認可直後に「queued」コメントを出し、Runner待ちであることをスマホから確認できるようにする

## 12. Permissions

- workflow top-level `permissions: {}`
- `authorize` / `report`（GitHub-hosted）: `contents: read`, `pull-requests: write`
- `review`（self-hosted）: `contents: read` のみ
- `contents: write` は使わない。auto commit / push / merge の経路を持たない

## 13. Concurrency

- PR単位のgroup `remote-review-pr-<number>`、`cancel-in-progress: true`。同一PRへの新しい `/review` らしきコメントは古いreviewをcancelする
- `/review` 以外のコメントは一意のgroupに入れ、進行中のreviewをcancelできないようにする
- Evidenceはrunごとのdirectoryとartifactに分離し、report jobはrun ID・SHAの一致を検証する
- 1台のRunnerは同時に1 jobだけを実行する（別PRのreviewは直列になる）

### Allowlist責務とConcurrency groupの範囲（2026-09-13 Independent Review remediation）

concurrency groupの計算はworkflow式（`fromJSON` によるallowlist評価を含む）を避け、「PRコメントで `/review` らしき文字列である」ことだけで判定する（§5）。そのため、allowlistに含まれないユーザーの `/review` らしきコメントも同じgroupに入り、進行中の（認可済みの）reviewをcancelしうる。ただしそのユーザー自身のrunはauthorize.mjsで拒否され、self-hosted runnerへは到達しない。cancelされた側のreviewは再実行（再度 `/review`）が必要になる。これは許容するResidual Risk（§16）であり、workflow式でallowlistを評価してこの経路を塞ぐ設計（旧実装）は、allowlist変数が不正なJSONのときworkflow式の評価自体を壊す（§5参照）ため不採用とした。

### Cancellation / Queued Comment Behavior（Advisory）

同一PRで新しい `/review` が実行され、古いrunが `cancel-in-progress` でcancelされた場合、古いrunが投稿した「queued」コメントだけが残ることがある。MVPとして過剰なcomment更新機構は追加しない。最低限次を運用・仕様として扱う。

- 新しいrunは古いrunの結果に優先する（supersede）
- cancelされたrunの「queued」コメントは結果ではない。人間は最新のrun / SHAを基準に判断する
- queuedコメントへの「superseded」表示は将来検討（今回のBlockingにはしない）

## 14. Windows Compatibility

- Linux前提にしない。`chmod`、`/tmp`、bash、grep、sed、awk、Linux pathを使わない
- 処理はNode.js scriptで実装する。`run:` は `node <script>`、またはshellに依存しない単純な `node -e` のみ
- npmは `node <npm-cli.js>` で起動する（Windowsの `.cmd` shimはshellを必要とするため）
- timeout時はprocess tree全体を停止する（Windows: `taskkill /T /F`）
- temp directoryはRunnerの `RUNNER_TEMP` 配下
- checkoutは `core.autocrlf=false` を固定して行う（review jobの `GIT_CONFIG_*` 環境変数）。Git for Windowsの既定 `core.autocrlf=true` はLFのfileをCRLFへ変換し、`.gitattributes` の無いrepositoryでformatter checkを誤ってFAILさせる。変換が残っていた場合は `ENVIRONMENT_FAILURE` として記録する（2026-09-13 Pilotのlocal E2Eで発見）
- workflow YAMLのplain scalarに `: ` を含めない。Windows PowerShell 5.1で壊れる埋め込み二重引用符を使わない

## 15. Acceptance Tests

| ID | 内容 | 検証方法（Pilot） |
|---|---|---|
| AT-01 | 通常コメントでは起動しない | unit test + GitHub上で手動確認 |
| AT-02 | `/review` で起動 | unit test + GitHub上で手動確認 |
| AT-03 | Issueでは起動しない | unit test + GitHub上で手動確認 |
| AT-04 | 未許可ユーザー拒否 | unit test +（第2アカウントがあれば）手動確認 |
| AT-05 | 正しいPR HEAD SHA | exact SHA checkout、`git rev-parse HEAD` 一致確認、report側SHA照合 |
| AT-06 | install結果取得 | unit test + local E2E + Runner実行 |
| AT-07 | lint取得 | 同上 |
| AT-08 | typecheck取得 | 同上 |
| AT-09 | build取得 | 同上 |
| AT-10 | test取得 | script未定義時SKIPPED（unit test）、定義後は実行 |
| AT-11 | Playwright取得 | script / dependency未定義時SKIPPED（unit test）、定義後は実行 |
| AT-12 | 失敗exit code取得 | unit test |
| AT-13 | 失敗時もEvidence生成 | unit test + local E2E |
| AT-14 | Codex利用不可時はInfra Error | unit test + local E2E（Codex未検出） |
| AT-15 | Codex結果をPRへ返却 | comment rendering unit test + Runner実行 |
| AT-16 | Evidenceをcommitしない | 静的テスト（commit / push / merge経路なし）+ Runner実行後の `git log` 確認 |
| AT-17 | secret leakageなし | redaction / env scrub unit test + Artifact目視確認 |
| AT-18 | 複数reviewの混線なし | concurrency設定 + run ID / SHA照合unit test + Runner実行 |
| AT-19 | auto pushしない | 静的テスト + Runner実行後確認 |
| AT-20 | auto mergeしない | 静的テスト + Runner実行後確認 |
| AT-21 | 必須context documentが欠落・空・読めない場合、Codex PASSでもVERDICTはINCOMPLETE | unit test（`CONTEXT_FAILURE`・decideVerdict）+ local E2E |
| AT-22 | review実行中にPRへ新しいcommitが増えた場合、古いSHAへの結果をcurrent HEADのPASS/FAILとして投稿しない（stale → INCOMPLETE） | unit test（renderPrComment stale）。GitHub上での実地確認は未検証（Runner登録後） |
| AT-23 | 使用する third-party GitHub Actions が full commit SHA へpinされている | 静的テスト（`uses:` が40桁hexであることを検査） |
| AT-24 | `/review` は小文字のみ有効（`/Review` 等は無効）。allowlist変数が不正なJSONでもfail closed（Runnerへ到達しない） | unit test |
| AT-25 | report jobは、trusted `codex.context_documents` manifestとEvidenceの `context_documents[]` を照合し、`null`・entry欠落・required flag不一致・required entryが非PRESENT・重複/想定外entryのいずれかがあればINCOMPLETEとする（Codexの自己申告verdictがPASSであっても採用しない） | unit test（`validateContextDocumentManifest` + report jobのfallback経路） |
| AT-26 | report jobは、trusted `config.checks` manifestとEvidenceの `checks[]` を照合し、required check欠落・required flag不一致・label不一致・重複/想定外entryのいずれかがあればINCOMPLETEとする。definitionと一致する `FAIL` / `INFRA_ERROR` / `SKIPPED` はこの照合単独ではINCOMPLETEにせず、§7/§8のVerification / Infrastructure semanticsへ渡す（2026-09-15、CORE-RR-FRR-002で文言・実装とも修正） | unit test（`validateCheckManifest` + report jobのfallback経路） |
| AT-27 | `codex.status` が `COMPLETED` のEvidenceについて、report jobは `codex.verdict` が非COMPLETEDでnullでない・`tool_check` が非VERIFIED・`workspace_unchanged` が非true・`approval_policy`/`ignore_user_config`/`ignore_rules` が既定値と不一致・`effective_sandbox` がtrusted config / platformの期待値と不一致・`blocking_count`/`advisory_count` がverdictと不整合・`codex-review.md` の再parse結果と `metadata.json` が不一致、のいずれかがあればINCOMPLETEとする | unit test（`validateMetadata` の新invariant + `validateCodexReviewConsistency` + `expectedEffectiveSandbox`） |
| AT-28 | Evidenceの各checkについて、`command` が非nullならtrusted `config.checks` から導出したcanonical invocation（`canonicalCommand()`）と一致することを、statusにかかわらず検証する。加えてPASS/FAIL/INFRA_ERROR/SKIPPEDそれぞれについて、`runChecks()`/`classifyCheckResult()`が実際に生成しうる状態（`exit_code`・`timed_out`・`skip_reason`・`failure`の組み合わせ）と一致しないEvidenceをINCOMPLETEとする（2026-09-15追加、CORE-RR-FRR-001。2026-09-15、CORE-RR-FINAL-001でPASS限定だった適用範囲を全statusへ拡張） | unit test（`validateCheckManifest` の `validateExecutionState` — command/exit_code/timed_out/skip_reason/failure mutation、PASS/FAIL/INFRA_ERROR/SKIPPEDそれぞれ） |
| AT-29 | `required: true` のcheckが1件でも `PASS` 以外（`SKIPPED` 含む）であれば、他のrequired checkがPASSしていてもVerificationは `PASS` にならない。Codexの出力に認識対象sectionの重複があれば、`VERDICT` 行の内容によらず `CODEX_FAILURE`（INCOMPLETE）とする（2026-09-15追加、CORE-RR-FRR-002） | unit test（`summarizeVerification` のrequired-SKIPPED mutation + `parseCodexReview` のduplicate-section mutation） |

Runner登録前に検証できない項目は「未検証」と記録し、推測でPASSにしない。単体テスト・mutation testで確認済みの項目（AT-06〜AT-29の多く）と、merge後に実GitHub Actions / 自宅PC Self-hosted Runner上で確認する必要がある項目（AT-01〜AT-05、AT-18の一部、AT-22のGitHub上での実地確認など）は区別する。前者は本書のOTOMO CORE側リポジトリと `otomo-lab` の `node --test` で継続的に確認できるが、後者はRunner登録・実PR経由の `/review` 実行後でなければ「検証済み」と記録しない。

**Runner Modeとの関係（2026-09-23）**: AT-01〜AT-29は、trigger・認可・Verification・Codex Evidence・report semanticsを対象としており、runner processのhosting方式（§6 Runner Mode）に依存しない。Runner Modeを変更してもAT-01〜AT-29の内容・期待結果は変わらない。Runner Mode固有のEvidence（runner userのidentity、採用mode、runner process稼働中のOnline / Idle、停止後のOffline、対話promptによるblockingが無いこと）は§20 Runner Acceptance Gateが所有し、AT番号を新設しない。ただし、あるRunner Modeで実環境ATを通したという記録は、そのmodeについてのみ有効である（§20 Mode変更時）。

## 16. Implementation Decision Trace（2026-09-13 / Pilot）

### Rejected Approaches

| Approach | 不採用理由 |
|---|---|
| Evidence auto commit（`docs/` 等への保存） | `contents: write` が必要になり、PRコードを実行する環境へ書き込み権限が近づく。履歴汚染・push競合も生む。Artifact / Job Summary / PR commentで追跡できる |
| Unrestricted comment execution（comment本文のcommand実行・`${{ }}` 展開） | command injectionそのもの。`/review` 以降はuntrusted contextとしてのみ扱う |
| Automatic source modification（Codexによるauto fix / remediation） | 実装担当と独立レビュー担当の分離（DEVELOPMENT_STANDARDS §1 / §5）に反する。read-only sandboxで禁止 |
| Auto merge（PASSで自動merge） | Remote Review PASSはPhase完了でも人間承認でもない |
| 単一のself-hosted jobでPR commentまで実行 | PRコード実行環境に `pull-requests: write` tokenが存在してしまう。GitHub-hosted jobへ分離した |
| `pull_request_target` / `workflow_run` による実装 | fork PRのコードを特権文脈で扱う典型的な脆弱パターン。スマホからのon-demand起動にも合わない |
| Codexにtest実行を判断させる | 決定論性・再現性を失う。harnessが事前に実行してEvidence化する |
| Codexのexit code・出力形式・自己申告だけでreview成立とみなす | Pilot local E2Eで、toolを使えないCodexが形式どおりの `VERDICT: PASS` を返した。外部照合（`TOOL_CHECK`）を必須にした（§9 Verification Capability Principle） |
| 既知のtool失敗メッセージのpattern検出だけで判定する | 1つ目の原因（tool host欠落）は検出できたが、2つ目の原因（sandbox policy）をすり抜けた。原因を列挙しない照合方式を採用 |
| OTOMO COREに共通workflow templateを先に作る | COREの「先に抽象化しない」原則。Pilotで実証してから昇格する |
| 実行時にotomo-coreからPromptを取得 | ref未固定ならdriftし、networkにも依存する。version header付きvendorと一致テストを採用 |
| Public repository（OTOMO COMES）をPilotにする | 誰でもPRを作れるrepositoryに自宅PCのRunnerを接続しない |
| malicious PR / hostile dependencyのisolationをv0.1で実装する | ephemeral runner・disposable VM・network isolationが必要でMVPの範囲を超える。代わりにTrust Modelを狭め、Non-Goalとして明示した（§1） |
| context_documentsのrequired/optional区別なし、全件skip可能のまま維持する | 必須のSource of Truthが存在しなくてもEvidence上PASSになりうる。「判定不能ならINCOMPLETE」原則（§8）と矛盾するため`CONTEXT_FAILURE`を追加した |
| stale reviewの扱いをreviewed SHAの表示だけに任せる | 人間がSHAを見落として古い結果をcurrent HEADのPASSと誤認しうる。report jobでの再取得・比較を追加した |
| GitHub Actionsをmajor tag参照のまま維持する | supply chain上、tagの指す内容は更新されうる。full commit SHAへpinした |
| workflow式（`fromJSON`）でallowlistを評価し続ける | allowlist変数が不正なJSONのときworkflow式の評価自体が壊れ、「安全に拒否し理由を記録する」設計と不一致になる。allowlist判定をtrusted script（`authorize.mjs`）へ一本化した |
| `/review` の大文字小文字を区別しない仕様を維持する | GitHub Actionsの式関数は大文字小文字を区別しない仕様であり、workflow側では強制できない。認可境界をtrusted scriptに置くなら、仕様も一致させる方が単純である |
| すべてのRemote Review runnerにWindows Serviceを必須とする（2026-09-23 Human Decision） | security invariantは「専用Windowsユーザーというsecurity principal」であって、process hosting方式ではない。Serviceを必須にすると、invariantを1つのhosting modeへ不必要に結合させる。現在の運用では常時受付が不要で、必要なときだけrunnerを起動する方がOperator Gate（§1）と整合し、露出時間も人間が制御できる。またService mode固有の非対話context由来の挙動を、必要になる前に導入することを避けられる。**Service modeが安全でないという判断ではない**（§6 Runner Mode） |

### Deliberate Non-Goals

| 項目 | 扱い |
|---|---|
| Implementation automation | 明示的不採用（Remote Reviewは監査のみ） |
| Auto remediation | 明示的不採用 |
| Auto push / auto commit / auto merge | 明示的不採用 |
| Organization-wide rollout（全Product展開） | 将来対応。Pilot AT通過・Human承認後に1 Productずつ |
| Ephemeral runner（VM / container / Windows Sandbox） | 将来対応候補。MVPは専用Windowsユーザーで運用 |
| Productへのtest / Playwright scriptの追加 | Product側Phaseの判断。Remote Reviewは存在するscriptだけを実行 |
| Remote Review結果の `PHASE_RECORDS` への自動反映 | 明示的不採用（人間判断） |
| Codex・checkの自動Retry | 将来対応。現在は人間が再度 `/review` する |
| Ephemeral runner / disposable VM | 将来対応候補（Future Hardening、§1）。MVPは専用Windowsユーザーで運用 |
| Codex専用security principalによる認証情報分離 | 将来対応候補（Future Hardening、§1）。MVPはCodex認証情報とEvidence生成が同一principal |
| Network egress restriction | 将来対応候補（Future Hardening、§1） |
| Evidence signing / provenance attestation | 将来対応候補（Future Hardening、§1） |
| Malicious / hostile third-party PRの安全な実行 | 明示的不採用（§1 Explicit Non-Goals）。v0.1のTrust Modelの範囲外 |

### Residual Risks

| Risk | 発生条件 | 影響 | 将来確認 |
|---|---|---|---|
| Persistent Windows runner | PRの `npm ci` lifecycle script・build設定がRunnerユーザー権限で実行される | Codex認証情報の読み取り、ユーザー設定（git / npm / Codex config）の改変による次回以降の汚染、Evidence偽装 | 専用ユーザー運用の徹底、ephemeral runner化 |
| Local machine compromise | 自宅PC自体の侵害 | Runner登録・Codex認証の悪用 | Runner削除手順、OS更新 |
| Codex authentication expiry | ChatGPTログインの期限切れ・logout | `AUTHENTICATION_FAILURE` によりINCOMPLETE | Setup Guideの再ログイン手順 |
| GitHub availability | Actions / API障害 | 起動しない・コメントできない | 再度 `/review` |
| Network availability | 自宅回線・npm registry・Codex APIの不通 | INFRA_ERROR / INCOMPLETE | 再度 `/review` |
| Runner offline | PCのスリープ・電源断 | jobがqueuedのまま残る（GitHubは一定時間後に失敗させる） | queuedコメントで可視化、電源設定 |
| Codex CLIの仕様変化 | CLI flag・出力形式の変更 | CODEX_FAILURE | CLI versionをEvidenceに記録、更新時にlocal E2E |
| Redactionの取りこぼし | 未知形式のsecretがlogに出る | Artifact閲覧者（repo read権限者）への露出 | Runner環境にsecretを置かない運用、retention 14日 |
| Prompt injectionによるreview品質低下 | PR本文・コード中の指示文 | 見逃し（誤PASS）の可能性 | Deterministic Verificationと人間判断の併用 |
| Grounding checkの範囲 | `TOOL_CHECK` はCodexがread-onlyコマンドを実行できたことだけを証明する | 必要なfileを読まずにverdictを出す可能性は残る | Verification Evidence欄の「実際に読んだfile・command」を人間が確認 |
| Runner context上のCodex sandbox | Runner専用ユーザーのcontextで `windows.sandbox=elevated` が動くかは、当該Runner Modeで実際に `/review` を通すまで未検証。Service mode（非対話service context）は、対話sessionを前提とする初期設定が必要な場合に追加の非互換を持ちうる | 全reviewが `TOOL_CHECK` 不成立でINCOMPLETE | 採用したRunner Modeでの最初の `/review` で確認（§20 mode固有check） |
| Interactive modeのrunner可用性 | Interactive modeでは、専用userのサインインと `run.cmd` の起動がなければrunnerがOfflineのままになる。再起動・logoff後は自動復帰しない | `/review` のjobがqueuedのまま残る（GitHubは一定時間後に失敗させる） | 意図した設計（Human controlled exposure window、§6）。operatorがrunnerを起動してから `/review` する運用で扱う |
| GitHub Actionsのリリース内容自体の改ざん・乗っ取り | full commit SHAへpinしても、そのSHAが指すコード自体が将来のバージョンで悪意ある変更を含む可能性 | supply chain | pin先SHAの内容をpin時に確認する（実施済み）。更新時は都度GitHub上でtag/commitを確認してからpinし直す |
| Concurrency groupがallowlist非依存（§13） | allowlistに含まれないユーザーの `/review` らしきコメントが、進行中の認可済みreviewをcancelしうる | 実行済みreviewの喪失・再実行の手間（unauthorized runがrunnerへ到達することはない） | private repositoryのcollaborator数を絞る運用、必要なら将来requester単位のgroup分割を検討 |
| Required Context Documentsの範囲 | harnessが確認するのは「checkout内にfileとして存在し読めるか」だけで、「Codexが実際にそのfileを読んだか」は確認しない | 必須文書が存在してもCodexが読まずに判断する可能性は残る | Verification Evidence欄の記述を人間が確認する（TOOL_CHECK Responsibilityと同じ限界） |
| Stale Review Protectionの再取得失敗 | report jobがcurrent HEAD再取得に失敗した場合、その失敗はGitHub API呼び出し全体の失敗として扱われ（job failure）、コメント自体が投稿されない | 結果が全く投稿されない（誤ったPASSが投稿されるより安全側だが、Runner実行の無駄） | workflow runのfailureとして人間が気づく。必要なら再度 `/review` |
| Stale Review Protectionの GET/POST TOCTOU（2026-09-14追加） | report jobが現在のHEAD SHAをGETで確認した直後から、PRコメントをPOSTするまでの間に新しいcommitがpushされる | 投稿されたコメントの「一致」判定は投稿直前の一時点のものであり、投稿完了までの間に発生した新しいpushはそのコメントに反映されない | 人間はコメントのreviewed SHAとPRの実際のHEADを見て判断し、必要なら再度 `/review` する。この窓を埋める原子的なGitHub API操作は無い |
| Verification Checks / Codex Evidence Semantic Invariantsの確認範囲（2026-09-14追加、CORE-RR-RR-001 / CORE-RR-RR-002） | `run.runner_os` はharness（review.mjs）が`RUNNER_OS`から記録する値で、PR由来ではないが、report jobが独立に再測定した値ではない（Evidenceの一部として自己申告される）。`effective_sandbox`のcross-checkは、この値とtrusted configから「review job実装ならこう構成したはず」の期待値を再現して一致を確認するものであり、runner自体が本当にそのOS上で動いたことそのものを独立検証しない | review jobを動かすharness自体（trusted script）が偽装されない、という既存Trust Model（§1）の範囲内でのみ有効。harnessそのものが侵害された場合はこのcross-checkも無効化されうる | 既存Trust Model・Operator Gate（§1）の範囲外の脅威として扱う。将来のRunner Acceptance Gate（§20）強化時に合わせて見直す |

Security関連の詳細: `harness/REMOTE_REVIEW_SECURITY.md` §5。

## 17. Product Adoption

1. Private repositoryであることを確認する
2. Product側に `.github/workflows/remote-review.yml` と `scripts/remote-review/` を置く（Pilot実装を参照）
3. `scripts/remote-review/config.json` にProductの実在commandだけを定義する
4. Promptは本COREのtemplateと同一内容をvendorする
5. `AGENTS.md` にRemote Review mode（read-only / no commit）を明記する
6. workflowをdefault branchへmergeする（`issue_comment` はdefault branchのworkflowだけが動く）
7. `harness/REMOTE_REVIEW_SETUP.md` に従ってRunner登録・variables設定を行う
8. AT-01〜AT-29を実行し、結果をProduct側へ記録する

## 18. GitHub Actions Dependency Pinning（2026-09-13 Independent Review remediation）

Remote ReviewはSecurity Harnessであるため、使用する第三者GitHub Actions（`actions/checkout`, `actions/upload-artifact`, `actions/download-artifact`）はmajor tag参照（`@v7` 等）ではなく、検証済みのfull commit SHAへpinする。

```yaml
uses: actions/checkout@<FULL_COMMIT_SHA> # v7.0.1
```

- pin先SHAは、GitHub API（`gh api repos/<owner>/<repo>/tags`）で当該tagのcommit SHAを直接確認してから使う。推測しない
- human-readable versionをtrailing commentで残す
- 更新するときは同じ手順で確認し直す
- AT-23（静的テスト）で全 `uses:` がfull commit SHAであることを検査する

## 19. Draft vs Formal Promotion Boundary（2026-09-13 Independent Review remediation）

このPR（Remote Review導入）のmergeは、「Draft specificationをmainへ収載する」ことを意味するだけであり、「正式なOTOMO CORE Ruleへの昇格」ではない。

正式昇格（`harness/DEVELOPMENT_STANDARDS.md` / `learning/PROMOTION_LOG.md` への反映）には、別途次が必要になる。

- Statusの変更（`Draft v0.1 / Pilot` → 昇格後のstatus）
- Independent Review
- Human Approval
- `learning/PROMOTION_LOG.md` への記録
- 必要ならPilot Productでの実環境AT結果

`learning/KNOWLEDGE_PROMOTION.md` および `harness/DEVELOPMENT_STANDARDS.md` §8 No Silent Harness Mutationと整合させる。§9 Verification Capability Principleも同様に、本書に記載されているだけではCORE Ruleとして確定しておらず、昇格には同じ手続きが必要（§9参照）。

## 20. Runner Acceptance Gate（2026-09-13 Independent Review remediation）

本番利用（実際の `/review` をSelf-hosted Runnerで受け付けること）の前に、次を必須のAcceptance Gateとする。`harness/REMOTE_REVIEW_SETUP.md` §6の手順で確認し、いずれか1つでも成立しなければRunnerを本番投入しない。

Formal Runner Acceptanceの構成は次のとおりであり、特定のprocess hosting方式を必須条件にしない（2026-09-23 Human Decision）。

```text
選択したRunner Mode（§6）+ 共通security invariant + mode固有check
```

### 共通invariant（Mode非依存）

- **Runner directory tree全体のACLとownerが、SYSTEM / Administrators / 専用runner user の3 principalに限定され、必須rightが揃っている**（下記 Runner Directory ACL Invariant）
- **target installationのRunner process・その実owner・local runner登録・受け入れ対象のGitHub Actions jobが、機械的な1本の連鎖として結び付いている**（下記 Principal Identity Evidence。自己申告・サインイン観察・process名だけの特定では足りない）
- 採用したRunner Mode（Interactive / Service）が明示的に記録されている
- 記録したRunner Modeが、受け入れ対象Listenerの実際のhosting（親process chainとservice登録状態）から機械的に判定したmodeと一致している（下記 Principal Identity Evidence。手入力のmode記録だけではPASSにしない。2026-09-23 Round 4、CORE-RR-L2-003）
- Acceptanceを実施したRunner Modeが、実運用で使うRunner Modeと一致している
- Codexが実際に起動できる（`codex --version` 相当）
- read-onlyコマンドを実行できる（`TOOL_CHECK` がVERIFIEDになる）
- Evidence（`metadata.json` 等）が生成される
- 上記の証跡が、受け入れ対象のGitHub Actions runと結び付けてDurable Acceptance Recordに残されている

### Runner Directory ACL Invariant（2026-09-23追加、CORE-RR-L2-001）

専用runner userが非Administratorであることは、それだけではRunner directoryを保護しない。Windowsの既定NTFS権限では `NT AUTHORITY\Authenticated Users` に `Modify` が継承されるため、**同一PC上の通常アカウントがrunner binary・script・`_work`・`_diag` を、専用runner userの権限で実行される前に差し替えられる**。

Acceptanceの必須条件（2026-09-23 Round 2で改訂。runner directoryのrootと、hidden・system属性を含むすべてのdescendantに適用する）:

- ACEを持つのは `NT AUTHORITY\SYSTEM` / `BUILTIN\Administrators` / 専用runner user の3つだけである。readだけのACEも含め、通常のauthenticated user・普段使いアカウント・`GITHUB_ActionsRunner_*` group等、他のprincipalのACEが残っていない（Service modeでも同じ）
- 3 principalが必須のAllow FullControlを持つ（rootはexplicit、descendantは継承）。専用runner userは `_work` / `_diag` / runner-owned stateへ書ける（読み取り専用にしない）
- Deny ACEが無い
- rootの継承が無効化されている。rootのpolicyを継承しないprotected descendantが無い
- すべてのobjectのownerが上記3 principalのいずれかである
- reparse pointが無い
- **recursiveなACL / owner変更（`/T` 等）を実行する前に**、root・ancestor・すべてのdescendantにreparse pointが無いことを、reparse pointを辿らない走査で確認している。検出した場合、または確認できない場合は、recursiveな変更を実行せずに止める。変更後の検証でreparse pointをFAILにすることは、この事前確認を代替しない（recursiveな変更がjunctionを辿ってtree外を変更しうるため）
- recursiveな変更の前に、rootを隔離し、通常アカウントがtree内に新しいentryを作れるdirectory（許可外principalがowner、または許可外principalのACEを持つdirectory）が残っていないことを確認している
- hardeningのACL変更commandがすべて成功している（exit code 0）
- 検証がtree全体を走査している。rootだけの検証、principal名の有無だけの検証ではPASSにしない
- 検証をhardening直後とAT-02完了後の両方で行い、両方がPASSである

Service modeの登録でrunnerが作る `GITHUB_ActionsRunner_*` groupは、許可principalに含めない。そのgroupが対象installationのものであることとmembershipの正当性を、runnerの内部実装に依存せず汎用に検証する方法を定義できないため、許可集合を狭める（runner userへ直接付与する）。

手順・検証script・negative testは `harness/REMOTE_REVIEW_SETUP.md` §5.1。検証が `ACL CHECK: PASS` にならない場合、**Runnerを本番投入しない**。

### Principal Identity Evidence（2026-09-23追加、CORE-RR-L2-002）

「専用ユーザーで動かしている」という申告や、サインインしているユーザーの観察だけでAcceptanceをPASSにしない。**runner processそのもののownerを確認する。**

必須（2026-09-23 Round 2で改訂）: 次の連鎖を、手作業の対応付けを挟まずに証明する。

```text
target runner directory
  → そのinstallationのListener process（ちょうど1つ）
  → そのprocessの実owner
  → そのprocessのhostingから判定したRunner Mode
  → 同じinstallationのlocal runner登録
  → 受け入れ対象GitHub Actions run attemptのself-hosted job
  → Durable Acceptance Record
```

- Listenerをprocess名ではなく、target installationの実行fileのpathで特定する。一致が0件・複数件ならFAILとし、最初の1件を選ばない。他のListenerのpathを読めず曖昧さを解消できない場合もFAIL
- processの実ownerが専用runner userと一致する（`whoami` はshellの実行者を示すだけで、これを代替しない）
- **Hosting mode**（2026-09-23 Round 4、CORE-RR-L2-003）: そのListenerがどうhostされているかから、Runner Modeを機械的に判定する。一次情報は実行中のprocess（親process chain・session）とし、service登録状態は矛盾の検出に使う。`.service` の有無やserviceの停止だけでmodeを決めない。Interactive modeは、serviceを作らずに登録し（`harness/REMOTE_REVIEW_SETUP.md` §5）、modeの変更に登録のやり直しを要する（同 §5.5）。そのため、Interactive modeは次の両方で判定する
  - Listenerが、対象installationの `run.cmd` を実行するrunner userの対話sessionのprocessからhostされている
  - 対象installationのservice登録が残っていない
- Service modeは、対象installationの唯一の実行中serviceがListenerをhostしていることで判定する。StartNameは専用runner userである。判定できない・証跡が矛盾する・記録したmodeと異なる場合はFAIL
- runner登録のidentityを、同じinstallationのlocal登録情報から導出する。それが、GitHub側で受け入れ対象jobを実行したrunnerの識別子と一致する。runner nameは一意である保証が無いため、nameだけで一致とみなさない
- job環境で記録された run ID / run attempt / runner name（既存Evidenceの `metadata.json` `run`）が、受け入れ対象runとlocal登録に一致する
- 同じinstallationのlocal traceに、受け入れ対象runの記録がある
- **Timing**: process identityをPID + 作成時刻で扱う。そのprocessがjob開始前に作成され、job完了後にも同じprocessとして存在していたこと（同じrunner稼働期間）を示す。PID単独の一致、別の稼働期間に取得した証跡、時刻の重なりを示せない証跡は使わない
- 上記のいずれかを取得できない・曖昧・不一致の場合は **§20 FAIL** とする

取得手順・script・時計のずれに対する余裕・判定規則は `harness/REMOTE_REVIEW_SETUP.md` §6.1。

本要求は **Runner Acceptanceの記録**に対するものであり、Remote ReviewのEvidence schema（§10 Schema 1.1）・`metadata.json` の項目・report jobの検証semantics（SEC-24〜SEC-27）を変更しない。job環境側の証跡には、既存の `metadata.json` の `run` fieldを読むだけである。

### Durable Acceptance Record（2026-09-23追加）

Runner Acceptanceの結果は、AI sessionやSetup作業の手元だけに残さない（`harness/DEVELOPMENT_STANDARDS.md` §4 Durable History）。

- 保存先は §4 Durable Historyの既存候補（Product repositoryのIssue / PR description / Product Decision log 等）から選ぶ。COREは新しい記録層を新設せず、形式も強制しない
- 対象runnerを持つProduct repository側に残す（`architecture/RESPONSIBILITY_BOUNDARIES.md` §3。COREは要求を所有し、個別runnerの運用記録はProductが所有する）

後続のreviewerが次を追えること:

| 項目 | 例 |
|---|---|
| どのrunner directory | path |
| どのmode | Interactive / Service。記録したmodeと、Listenerのhostingから機械的に判定したmode、およびその判定の証跡（親process・service登録状態） |
| どのWindows principal | 専用runner user、Listenerのprocess owner |
| ACL検証 | hardening直後・AT-02完了後それぞれのPASS / FAIL |
| どのlocal runner登録 | local登録情報のrunner識別子・runner name |
| どのprocess | Listenerの実行file path・PID・作成時刻 |
| どのGitHub run | run ID / attempt、jobを実行したrunnerの識別子・name、jobの開始・完了時刻 |
| job環境の証跡 | `metadata.json` の run ID / attempt / runner name |
| local trace | 受け入れ対象runを記録したlocal log |
| timing | 証跡の取得時刻、Timing判定 |
| Codex / `TOOL_CHECK` / Evidence生成 | PASS / FAIL |
| 実施日 | 日付 |
| 総合判定 | PASS / FAIL |

必須のbinding証跡（上表のdirectory・mode・principal・ACL・local登録・process・GitHub run・job環境・local trace・timing）が1つでも欠けている記録は、**§20 FAIL** とする。

password・token・SID値・個人のhome path・無関係な個人情報を記録しない（`harness/DEVELOPMENT_STANDARDS.md` §5 Evidence Integrity）。hostnameは必要なら一般名へ置換してよい。

### Interactive mode固有check

Interactive modeで本番投入する場合、上記に加えて次を確認する。

1. target installationのListener processの実ownerが専用runner userであり、そのprocessが受け入れ対象jobと結び付いていることを、Principal Identity Evidence（上記、`harness/REMOTE_REVIEW_SETUP.md` §6.1）で確認している。`run.cmd` を起動したshellの `whoami` だけでこの項目をPASSにしない
2. `codex --version` 相当が成功する
3. Codexがread-onlyコマンドを実行できる
4. `TOOL_CHECK` がVERIFIEDになる
5. `run.cmd` を起動した後、対話promptでreview jobが停止しない（review job単位でunattendedに完走する）
6. Evidenceが生成される
7. runner processを停止すると、GitHub上でrunnerがOffline / 利用不可になり、新しいRemote Review jobが開始されない
8. service modeでの動作を主張しない（Service modeを実際に検証していない限り、Service mode側をPASSと記録しない）
9. Runner directory ACL検証（`harness/REMOTE_REVIEW_SETUP.md` §5.1、tree全体）が、hardening直後とAT-02完了後の両方で `ACL CHECK: PASS` である
10. 受け入れ対象Listenerのhostingが機械的に `Interactive` と判定され、対象installationのservice登録が残っていない（Principal Identity Evidenceの Hosting mode、`harness/REMOTE_REVIEW_SETUP.md` §6.1 Hosting mode判定）。「Runner Mode = Interactive」という記録だけでこの項目をPASSにしない

### Service mode固有check

Service modeを採用する場合は、Service mode自身のcheck（非対話service contextでのCodex起動・read-onlyコマンド実行・`TOOL_CHECK` VERIFIED・Evidence生成）を別途実施する。受け入れ対象Listenerのhostingが機械的に `Service` と判定されることも必須とする（Principal Identity Evidenceの Hosting mode）。Interactive modeのPASSは、Service modeのAcceptanceを代替しない。

### Mode変更時

Interactive → Service、またはService → Interactiveへ変更する場合、変更後のmodeについて上記のmode固有checkを再実施する。過去に別modeで取得したAcceptance結果を流用しない。

### 記録

実環境の検証はRunner登録・実PR経由の `/review` 実行後でなければ行えないため、未実施の項目は「未検証」と明記する（推測でPASSにしない）。検証していないRunner Modeについて、PASSと記録しない。

次のいずれかに該当する場合、Acceptanceを **FAIL** とする（「未検証」として保留することはできても、PASSにはしない）。

ACL:

- Runner directory ACL検証が `ACL CHECK: PASS` でない、または実施されていない（hardening直後・AT-02完了後のどちらか一方でも）
- hardeningのACL変更commandが失敗した
- recursiveなACL / owner変更の前にreparse pointの事前確認が行われていない、reparse pointを検出した、または走査に失敗した
- 必須principal（SYSTEM / Administrators / 専用runner user）が欠けている、または必須rightが不足している
- 許可外のprincipal（writeを持つ通常アカウント・`GITHUB_ActionsRunner_*` groupを含む）のACEがある
- Deny ACEがある
- protected descendant、許可外のowner、reparse pointがある
- descendantのfile / directoryに許可外のexplicit ACEがある

Identity / run binding:

- target installationのListenerの一致が0件または複数件である、または他のListenerのpathを読めず特定できない
- Listenerの実行file pathがtarget installationと一致しない
- process ownerが専用runner userと一致しない、または取得できない
- local runner登録が、受け入れ対象jobを実行したrunnerと一致しない
- job環境側の証跡（`metadata.json` の `run`）が無い、または一致しない
- run IDが一致しない、または証跡がrun IDに結び付いていない
- job期間とprocessの存続期間の重なりを示せない
- 以前のrunner稼働期間に取得した証跡である

記録:

- 上記の証跡がDurable Acceptance Recordに残っていない、または必須のbinding証跡が欠けている

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成（Independent Review・Human承認前）
- 2026-09-13 v0.1 Draft, Independent Review remediation: Trust Model / Non-Goals / Operator Gateを明記（CORE-RR-IR-001）、Required / Optional Context Documentsと`CONTEXT_FAILURE`を追加（CORE-RR-IR-002）、Stale Review Protection、GitHub Actions SHA pinning、`/review` の大文字小文字判定とallowlist責務の明確化、Model / Prompt Provenance、Durable History Boundary、Draft/Formal Promotion Boundary、Cancellation / Queued Comment Behaviorを追加。Evidence Schemaを1.0から1.1へ
- 2026-09-14 v0.1 Draft, Independent Re-review remediation（CORE-RR-IR-002未解決分の再remediation）: `required: true`presence checkだけでは、Productが必須と定義するSource of Truthがそもそも`codex.context_documents` manifestから漏れること自体を防げない、という残存Blockingを解消。Source of Truth Manifest Consistency原則（本書§9）と、report jobによるtrusted manifest ↔ Evidence `context_documents[]`のcross-check（`null`・entry欠落・required flag不一致・非PRESENT required entry・重複/想定外entryをすべてINCOMPLETEへ倒す、AT-25）を追加。Advisory 3件を解消: (1) report jobの現在HEAD SHA取得が欠落・不正な場合のfail-open除去（GET/POST間のTOCTOUはResidual Riskとして明記）、(2) `COMPLETED` EvidenceのModel / Prompt Provenance field（存在・型・`prompt_hash`のSHA-256形式）をreport側で検証、(3) `metadata.json`の`Schema 1.0`表記を`Schema 1.1`へ修正（§10、`evidence.mjs`）。Pilot実装（`otomo-lab`）のtest suiteを58から66へ拡張
- 2026-09-14 v0.1 Draft, Independent Re-review remediation（CORE-RR-IR-002再remediation、Blocking）: Source of Truth Manifest Consistency原則（本書§9）はAGENTS.md ↔ config.jsonの一致を「恒久的な運用規則」として文章化していたが、それを機械的に検証する回帰テストが存在せず、将来configから必須entryを削除する・`required: false`へ弱めるといった変更が無検出でPASSしうる、という指摘（CORE-RR-IR-002再remediation）を解消。`otomo-lab` `scripts/remote-review/lib/source-of-truth.mjs`（`checkSourceOfTruthManifestConsistency`）と`test/source-of-truth.test.mjs`を追加し、必須entry削除・`required: true → false`・想定外entry追加・duplicate entryの4種のmutationすべてでtestがFAILすることと、現行10文書manifestでPASSすることを確認（SEC-23）。Advisory 2件を解消: (1) Evidence Schema 1.1 provenance — `COMPLETED`時のprovenance文字列を空文字列も拒否するnon-empty必須にし、`prompt_version`をnon-null必須にした上で、null/空/欠落/不正形式の`prompt_hash`に対するnegative testを追加（`resolved_model`/`reasoning_effort`の`"UNKNOWN"`は引き続き許容）、(2) §17 Product Adoptionの`AT-01〜AT-24`表記を、既存AT表と一致する`AT-01〜AT-25`へ修正。Pilot実装（`otomo-lab`）のtest suiteを66から78へ拡張
- 2026-09-14 v0.1 Draft, Independent Re-review remediation（CORE-RR-RR-001 / CORE-RR-RR-002、Blocking）: `report.mjs`のEvidence検証は、`codex.status`/`codex.verdict`と、review job実装（§9強制手段）が実際に保証する不変条件（grounding確認済み、read-only境界維持、固定invocation policy）との意味的整合を検証していなかった（CORE-RR-RR-001）。Codex Evidence Semantic Invariants（本書§9）を追加し、`codex.status`が`COMPLETED`以外なら`codex.verdict`はnull以外を許さない、`tool_check`/`workspace_unchanged`/`approval_policy`/`ignore_user_config`/`ignore_rules`/`effective_sandbox`（trusted config・platformとの整合込み）/`blocking_count`・`advisory_count`（verdictとの整合込み）/`codex-review.md`本文の再parse結果、をすべてfail-closedにした（AT-27）。また、`summarizeVerification`（§10）がEvidence自己申告の`checks[]`だけを見ており、trusted `config.checks`の`required: true`checkがEvidenceから欠落・非構造的にSKIPPEDのままでも単独ではFAILに倒れない欠落があった（CORE-RR-RR-002）。Verification Checks Manifest Cross-Check（本書§7）を追加し、report jobが独立にcheckoutしたtrusted `config.checks`とEvidenceの`checks[]`を照合、required check欠落・required flag/label不一致・required checkの非PASS・重複/想定外entryをすべてINCOMPLETEへ倒す（AT-26）。Residual Risksに、この2つのcross-checkがEvidence自己申告の`run.runner_os`を前提とすることを追加。§17 Product Adoptionの`AT-01〜AT-25`表記を`AT-01〜AT-27`へ更新。Pilot実装（`otomo-lab`）のtest suiteを78から95へ拡張
- 2026-09-15 v0.1 Draft, Independent Re-review remediation（CORE-RR-FRR-001 / CORE-RR-FRR-002、Blocking。ChatGPT独立監査によるVerification semantics regressionの指摘を含む）: CORE-RR-RR-002で追加したVerification Checks Manifest Cross-Check（本書§7、`validateCheckManifest`）は、trusted `config.checks`のid/required/labelまでしか照合しておらず、Evidenceが`status: PASS`と自己申告するcheckが実際にtrusted definitionの`command`/`npm_script`を実行した結果であることは確認していなかった（CORE-RR-FRR-001）。Canonical Command Binding（本書§7）を追加し、`status: PASS`のcheckについて、`command`がtrusted `config.checks`から`canonicalCommand()`（`lib/checks.mjs`、review job自身の実行`planCheck`とreport jobのcross-checkが同じ関数を共有し、command文字列を別途ハードコードしない）で導出した値と一致すること、`exit_code === 0`・`timed_out === false`・`skip_reason === null`・`failure === null`であることをfail-closedに検証する（AT-28）。また、同じcross-checkは「trusted manifestで`required: true`のcheckがEvidence側で`PASS`以外」であることそのものをEvidence整合性問題として扱っており、正常に実行・記録された`FAIL`（本来のVerification FAIL、VERDICT: FAILになるべき）や`INFRA_ERROR`（Infrastructure FAILED）までEvidence corruptionと混同し、常にINCOMPLETEへ倒していた（CORE-RR-FRR-002、Verification semantics regression）。この責務を分離し、definitionと一致する`FAIL`/`INFRA_ERROR`はこの照合単独ではINCOMPLETEにせず、既存の§7/§8 Verification / Infrastructure semanticsへ渡すよう修正した。一方、`summarizeVerification`（`lib/evidence.mjs`）は「1件もFAILせず・INFRA_ERRORもなく・PASSが1件以上」という判定基準しか持たず、`required: true`のcheckが（非構造的理由で）`SKIPPED`のまま残っていても、別のrequired checkがPASSしていれば全体がVerification PASSに到達しうる欠落があった。修正後は、Evidenceに`required: true`のcheckが1件以上ある場合、それら全件がPASSでなければVerificationはPASSにならない（Required Check PASS Requirement、本書§7）。Codex出力の`extractSections`/`parseCodexReview`は、同名section（`## Blocking Findings`等）が重複した場合に後勝ちで上書きしており、先行する`## Blocking Findings`に実際のFindingが記載され、後続の空の`## Blocking Findings`がそれを上書きすると、`VERDICT: PASS`と矛盾なくparseできてしまっていた（CORE-RR-FRR-002）。認識対象5 sectionそれぞれの重複を検出し、重複があれば`CODEX_FAILURE`（INCOMPLETE）としてPASS/FAIL verdictを採用しないよう修正した（本書§9 出力形式、AT-29）。§15 Acceptance TestsにAT-28/AT-29を追加し、AT-26の記述をdefinition integrityとexecution result semanticsの分離を反映するよう修正、§17 Product Adoptionの`AT-01〜AT-27`表記を`AT-01〜AT-29`へ更新。`harness/REMOTE_REVIEW_SECURITY.md`にSEC-26/SEC-27を追加し、SEC-25の記述を同様に修正。Pilot実装（`otomo-lab`）のtest suiteを95から112へ拡張
- 2026-09-15 v0.1 Draft, 最終Independent Re-review remediation（CORE-RR-FINAL-001、Blocking）: Canonical Command Binding（本書§7）の実装は、command/exit_code/timed_out/skip_reason/failureの整合検証を `status: PASS` を自己申告するcheckだけに限定していた。`status: FAIL`（`failure`は正当な`VERIFICATION_FAILURE`に見える）でありながら `command` が別checkのものに差し替えられたEvidenceがこの検証を素通りしうる、という指摘（CORE-RR-FINAL-001）を受け、`lib/checks.mjs` `validateCheckManifest`のPASS限定block を `validateExecutionState()` へ置き換えた。command一致チェックは `command !== null` である限りstatusによらず適用し、加えてPASS/FAIL/INFRA_ERROR/SKIPPEDそれぞれについて、`runChecks()`/`classifyCheckResult()`（`lib/checks.mjs`/`lib/process.mjs`）の実際の状態遷移から機械的に導出できる不変条件（例: FAILは`timed_out`が常にfalse・`failure`が常に非null、構造的FAILは`command`/`exit_code`が常にnull、実行されたFAILは`skip_reason`が常にnullかつ`exit_code`が非null非ゼロ、INFRA_ERRORは`command`が常に非null・`skip_reason`が常にnull、SKIPPEDは`command`/`exit_code`が常にnullかつ`timed_out`が常にfalse・`failure`が常にnull）を検証する。INFRA_ERRORの`exit_code`/`timed_out`は、invocation準備失敗・spawn失敗・timeout・signal終了のsub-caseごとに正当に異なりうるため、単一の期待値へは固定していない。AT-28の記述をこの拡張に合わせて修正。§15 Acceptance Tests・本節（§7）を更新。Pilot実装（`otomo-lab`）のtest suiteを112から122へ拡張（新規10件、すべて`checks.test.mjs`）。同時に検出された`otomo-lab`固有の`LAB-RR-FINAL-002`（`package-lock.json`が一部のnpmバージョンで`npm ci`のlockfile整合性検証に失敗する既存の欠陥。Remote Reviewのreport job／Evidence Schema／Verification semanticsとは無関係）は`otomo-lab`側の`docs/DECISIONS_AND_FAILURES.md` D-020に記録し、本書のharness仕様変更は伴わない
- 2026-09-16 v0.1 Draft, Focused Independent Audit remediation（CORE-RR-AUDIT-001、docs-only）: 本書§7冒頭のVerification Checks Manifest Cross-Check説明に、CORE-RR-FRR-002（2026-09-15）で既に修正されたはずの旧semanticsが現在形のまま残存していた。具体的には、(1) `summarizeVerification`の判定条件を「checkが1件もFAILせず・INFRA_ERRORもなく・PASSが1件以上あればPASS」と、`required`フラグを考慮しない旧ロジックのまま説明していた（Required Check PASS Requirementが導入した現行の「required全件PASS」不変条件と矛盾）、(2) manifest cross-checkのINCOMPLETE条件bulletに「trusted manifestで`required: true`のcheckが、Evidence側で`PASS`以外のstatusになっている」という、CORE-RR-FRR-002がEvidence corruptionと実行結果semanticsを分離したはずの旧記述が残っていた。いずれも同一section内の後続文（definition integrity / execution result semanticsの分離説明、Required Check PASS Requirement）とは矛盾しており、規範本文中に旧仕様と現行仕様が並存していた。実装・SEC-25・AT-26・LAB test suiteに変更はなく、本書の記述をそれらと一致するよう修正しただけ（新規SEC/AT番号追加なし）。§7冒頭の`summarizeVerification`説明を「`checks[]`のみを見る判定である」という現行の限界の説明に置き換え、該当bulletを削除して「required checkの非PASSはこの照合の対象外」という明示的な否定文へ置換した
- 2026-09-17 v0.1 Draft, Risk-based Independent Review整合（docs-only）: `harness/DEVELOPMENT_STANDARDS.md` §5がRisk-based Review Assurance Level（L0 / L1 / L2）へ再設計されたことに伴い、§2の既存Harness対応表へReview Assurance Level行を追加し、`PHASE_WORKFLOW` §4の節名変更を反映。あわせて、Remote Reviewが常にMandatory audit scope全体を監査すること、Review Assurance LevelがRemote Review側のscope規則を変更しないことを明記した。Remote Review自体のStatus（Draft v0.1 / Pilot）、Supported Trust Model、Explicit Non-Goals、Security Claim、Trigger / Verification / Codex / Evidence semantics、SEC・AT番号、Pilot実装のいずれにも変更はない
- 2026-09-23 v0.1 Draft, Runner Mode採用（Human Decision、docs-only、Forced Level 2扱い）: Formal Runner Acceptance Gate（§20）が `Windows Serviceとして動作する` をmode固有の必須条件として要求しており、security invariant（Runner専用Windowsユーザーというsecurity principal）を特定のprocess hosting方式へ結合させていた。§6にRunner Mode（Interactive / Service）の定義を追加し、今回のPilotではInteractive mode（専用runner userでサインインし `run.cmd` を手動起動する）を採用するHuman Decisionを記録した。§20を「選択したRunner Mode + 共通security invariant + mode固有check」の構成へ再定義し、Interactive mode固有check 8項目とService mode固有check、Mode変更時の再検証要求を明記した。Interactive modeが「review job単位ではunattended」であること（`run.cmd` 起動後は個々のCodex実行に人間の操作を要さないこと）を用語として固定した。§15にAT-01〜AT-29がRunner Mode非依存である旨とRunner Mode Evidenceの所有が§20であることを追記（新規AT番号なし）。§16 Rejected Approachesへ「すべてのrunnerにWindows Serviceを必須とする」案の不採用理由を追加し、Residual Risksの `Windows service上のCodex sandbox` 行をmode中立な `Runner context上のCodex sandbox` へ置き換え、Interactive modeのrunner可用性（再起動・logoff後に自動復帰しない）行を追加した。**Service modeを削除・禁止せず、安全でないとも判断していない**。Trigger / Verification / Codex / Evidence semantics、workflow実装、permissions、Codex sandbox、Evidence format、prompt、allowlist、PR write boundary、Product adoption architecture、Status（Draft v0.1 / Pilot）に変更はない
- 2026-09-23 v0.1 Draft, L2 Independent Review remediation Round 1（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: PR #13へのForced L2独立レビューで、Interactive mode採用の手順がdedicated-user security boundaryを保全も証明もしていない、という2件のBlockingを受けた。(1) CORE-RR-L2-001: §20はRunner専用userのcontextで動くことだけを要求し、Runner directoryのACLに要求が無かった。Windows既定NTFS権限では `NT AUTHORITY\Authenticated Users` に `Modify` が継承されるため、同一PC上の通常アカウントがrunner binary・script・`_work`・`_diag` を、専用runner userの権限で実行される前に差し替えられる（実機のPilot runner directoryで `Authenticated Users: Modify` を確認）。§20へ Runner Directory ACL Invariant を追加し、accessを SYSTEM / Administrators / 専用runner user に限定すること・継承無効化・runner userのwrite権限維持を必須条件とし、Interactive mode固有checkへ項目9（ACL検証PASS）を追加した。手順と検証コマンドは `harness/REMOTE_REVIEW_SETUP.md` §5.1。(2) CORE-RR-L2-002: §20はどのWindows principalがrunnerを実行したかをoperatorの申告で足りる形にしており、機械的なidentity checkもacceptance対象runとの結び付けも定義していなかった。§20へ Principal Identity Evidence を追加し、`Runner.Listener` processの実 ownerを確認すること（`whoami` では代替しない）、run ID・runner name・採用modeと結び付けて記録することを必須とし、Interactive mode固有check 1をprocess owner確認へ強化した。加えて Durable Acceptance Record 節を追加し、保存先を既存の §4 Durable History候補（Product repository側）とした（新しい記録層は新設していない）。§20 記録節へFAIL条件4件を明記。**Remote ReviewのEvidence schema（§10 Schema 1.1）・`metadata.json`・report jobの検証semantics（SEC-24〜SEC-27）・AT-01〜AT-29・workflow実装・permissions・Trust Model・Operator Gate・Status（Draft v0.1 / Pilot）は変更していない**。Service modeの扱いも変更していない
- 2026-09-23 v0.1 Draft, L2 Independent Re-review remediation Round 2（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: Round 2のwhole-PR再レビューで、Round 1の対応では両Findingが未解消と判定された。(1) CORE-RR-L2-001: Runner Directory ACL Invariantがrootの継承無効化と許可集合外identityの不在しか要求しておらず、必須principal・必須right・Deny・descendant・ownerを要求していなかったため、安全でない、または使えないtreeがACL CHECKをPASSしえた。Acceptance条件をtree全体のinvariant（許可principal3つだけ・必須FullControl・Deny無し・root protected・protected descendant無し・owner制限・reparse point無し・command成功・hardening直後とAT-02完了後の両方で検証）へ改訂し、`GITHUB_ActionsRunner_*` groupを許可principalから外した（Service modeでも同じ）。(2) CORE-RR-L2-002: Principal Identity Evidenceが、process ownerとrun ID・runner nameを手作業で対応付ければ足りる形であり、どのinstallationのprocessか・どのrunを実行したか・同じ稼働期間かを証明していなかった。target directory → Listener process（pathの一致がちょうど1つ）→ owner → local runner登録 → GitHub job（runner識別子）→ job環境の既存 `metadata.json` `run` → Durable Acceptance Recordの連鎖と、PID + 作成時刻によるtiming要求を必須とした。Durable Acceptance Recordの項目を拡張し、必須のbinding証跡の欠落をFAILとした。§20 記録節のFAIL条件を ACL / Identity・run binding / 記録 に分けて列挙した。Interactive mode固有check 1 / 9を上記に合わせた。手順・scriptは `harness/REMOTE_REVIEW_SETUP.md` §5.1 / §6.1が所有する。**Evidence schema（§10 Schema 1.1）・`metadata.json`・report jobの検証semantics（SEC-24〜SEC-27）・AT-01〜AT-29・workflow実装・permissions・Trust Model・Operator Gate・Runner Mode（Interactive採用）・Status（Draft v0.1 / Pilot）は変更していない**
- 2026-09-23 v0.1 Draft, L2 Independent Re-review remediation Round 3（pre-mutation reparse-point handling、Blocking、docs-only）: Round 3のwhole-PR再レビューで、hardeningがreparse pointの確認より前にrecursiveなACL / owner変更を実行していた点が新しいBlockingとされた（`icacls /reset /T` はtree内のdirectory junctionを辿りtree外を変更することを実機で確認）。§20 Runner Directory ACL Invariantへ、recursiveな変更の前にreparse pointを辿らない走査で確認すること（事後の検証で代替しない）と、rootの隔離・directory footholdの不在を確認することを追加し、記録節のACL FAIL条件へ事前確認の欠落・検出・走査失敗を追加した。手順は `harness/REMOTE_REVIEW_SETUP.md` §5.1が所有する。**Evidence schema（§10 Schema 1.1）・`metadata.json`・SEC-01〜SEC-27・AT-01〜AT-29・workflow実装・permissions・Trust Model・Operator Gate・Runner Mode（Interactive採用）・Principal Identity Evidence・Status（Draft v0.1 / Pilot）は変更していない**
- 2026-09-23 v0.1 Draft, L2 Independent Re-review remediation Round 4（CORE-RR-L2-003、Blocking、docs-only）: Final whole-PR L2再レビューで、§20がRunner Modeの記録とListener → installation → runのbindingを要求する一方、受け入れ対象Listenerが実際にどのmodeでhostされているかを機械的に要求していない点がBlockingとされた。記録値だけのmodeで、Service modeで動くListenerをInteractiveとして受け入れうる。§20 共通invariantへ、記録したRunner Modeがhostingからの機械的判定と一致することを追加した。Principal Identity Evidenceの連鎖とbulletへHosting modeを追加した。判定は親process chainを一次情報とし、service登録状態を矛盾検出に使う。Interactive modeは、既存の `harness/REMOTE_REVIEW_SETUP.md` §5 / §5.5に基づき、対象installationのservice登録が残っていないことも要求する。Durable Acceptance Recordのmode項目、Interactive mode固有check（10項目目）、Service mode固有checkを更新した。手順・scriptは `harness/REMOTE_REVIEW_SETUP.md` §6.1。**Evidence Schema 1.1・`metadata.json`・SEC-01〜SEC-27・AT-01〜AT-29・workflow・permissions・Runner Mode（Interactive採用）・Trust Model・Operator Gate・Status（Draft v0.1 / Pilot）は変更していない**
