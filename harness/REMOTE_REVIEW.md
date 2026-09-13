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

## 2. 既存Harnessとの関係

Remote Reviewは新しいReview原則ではなく、既存原則の**実行経路**である。以下を置き換えない。

| 既存Rule | Remote Reviewでの具体化 |
|---|---|
| DEVELOPMENT_STANDARDS §1 Role Separation | Claude Code = 実装、Codex = 独立監査。Remote Reviewは修正しない |
| §4 Implementation Decision Trace | 本書 §16 |
| §5 Independent Review（blocking / advisory） | Codex出力のBlocking / Advisory。FAIL ⇔ Blocking ≥ 1 |
| §5 Clean-room Verification | 対象SHAの明示、fresh checkout、lockfileからのclean install、Productが定義するValidationのみ実行 |
| §5 Verification Evidence | command / exit code / stdout・stderr / 時刻を一次情報として保存。「AIがPASSと言った」だけをEvidenceにしない |
| §5 Evidence Integrity | redaction・size制限後のみ永続化。raw logをPRへ貼らない |
| §9 External Boundary Safety | Codex / npm / GitHub APIの生出力を分類理由へ流さない。transient / permanentを分類 |
| §10 Explicit Invariants / Fail Fast | SHA・PR番号・run IDの不一致、verdict不整合を明示的にINCOMPLETEとする |
| PHASE_WORKFLOW §4 Independent Review | 監査観点をPromptへ固定 |

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
3. コメント本文が `/review` で始まる（大文字小文字は区別しない。`/reviewer` は対象外）
4. 投稿者が `type: User` かつ allowlist（repository variable `REMOTE_REVIEW_ALLOWED_USERS`）に含まれる
5. `author_association` が `OWNER` / `MEMBER` / `COLLABORATOR`
6. PRがopen、head repositoryがbase repositoryと同一（fork PR拒否）、SHAが40桁hex

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

- 想定: 自宅Windows PC / x64 / GitHub Actions Runner（Windows service）/ Node.js / Git / Codex CLI（必要ならPlaywright browsers）
- 推奨label: `self-hosted`, `windows`, `x64`, `otomo-review`
- labelは repository variable `REMOTE_REVIEW_RUNS_ON`（JSON配列）で変更できる
- Runnerは専用Windowsユーザーで動かす（`harness/REMOTE_REVIEW_SECURITY.md` §3）
- 登録対象は **private repositoryのみ**

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
| `metadata.json` | 機械可読Evidence（Schema 1.0） |
| `review-evidence.md` | 人間向けEvidence全体 |
| `<check-id>.log` | 各checkのcommand・時刻・exit code・stdout / stderr（redaction済み）。例 `install.log`, `lint.log`, `typecheck.log`, `build.log` |
| `codex-prompt.md` | Codexへ渡したPrompt（redaction済み） |
| `codex.log` | Codex CLIのstdout / stderr（redaction済み） |
| `codex-review.md` | Codexの最終出力（redaction済み） |

test / Playwrightを導入したProductでは `test.log` / `playwright.log`（必要なら `playwright-report`）が加わる。

### Redaction

永続化前に置換する（best effort）: GitHub token、OpenAI / Anthropic形式のkey、AWS access key、Google API key、Slack token、npm token、JWT、private key block、Authorization / Cookie / Set-Cookie header、Bearer / Basic credential、URL内の `user:password@`、署名付きURL・token系query parameter、`password=` 等の代入、実行環境の機密名環境変数の値。

自動redactionだけで安全とみなさない（DEVELOPMENT_STANDARDS §5 Evidence Integrity）。raw logはPR commentに貼らない。checkやCodexの子processには、機密名の環境変数・`GITHUB_*`・`ACTIONS_*` を渡さない。

### Schema 1.0（`metadata.json`）

| Field | 型 | 内容 |
|---|---|---|
| `schema_version` | `"1.0"` | |
| `generator` | object | `name`, `template_version`, `config_schema_version` |
| `repository` | string \| null | `owner/repo` |
| `pull_request` | object \| null | `number`, `title`, `url`, `base_ref`, `base_sha`, `head_ref`, `head_sha` |
| `request` | object \| null | `requested_by`, `comment_id`, `comment_url`, `requested_at`, `focus_provided` |
| `run` | object | `id`, `attempt`, `url`, `runner_name`, `runner_os`, `node_version` |
| `timing` | object | `started_at`, `finished_at`, `duration_ms` |
| `checks[]` | array | 下表 |
| `verification.status` | `PASS` \| `FAIL` \| `UNKNOWN` | §8 |
| `codex` | object | `status`（`COMPLETED` / `FAILED` / `NOT_RUN`）, `verdict`（`PASS` / `FAIL` / null）, `problem`, `cli_version`, `exit_code`, `duration_ms`, `timed_out`, `prompt_bytes`, `workspace_unchanged`, `tool_check`（`VERIFIED` / `MISSING` / `MISMATCH` / `UNAVAILABLE` / null）, `blocking_count`, `advisory_count` |
| `infrastructure` | object | `status`（`HEALTHY` / `FAILED`）, `failures[]`（`stage`, `category`, `transience`, `reason`） |
| `review_context` | object \| null | `changed_files_count`, `diff_bytes`, `diff_truncated`, `harness_sensitive_changes[]` |
| `verdict` | `PASS` \| `FAIL` \| `INCOMPLETE` | §8 |
| `artifacts[]` | string[] | Artifact内のfile名 |
| `redaction` | object | `applied`, `method` |

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

## 11. PR Comment

```markdown
## Remote Independent Review

**VERDICT: PASS | FAIL | INCOMPLETE**

Verification: … · Codex: … · Infrastructure: …

Commit:
`<HEAD SHA>`

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

- PR単位のgroup `remote-review-pr-<number>`、`cancel-in-progress: true`。同一PRへの新しい `/review` は古いreviewをcancelする
- `/review` 以外のコメントは一意のgroupに入れ、進行中のreviewをcancelできないようにする
- Evidenceはrunごとのdirectoryとartifactに分離し、report jobはrun ID・SHAの一致を検証する
- 1台のRunnerは同時に1 jobだけを実行する（別PRのreviewは直列になる）

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

Runner登録前に検証できない項目は「未検証」と記録し、推測でPASSにしない。

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
| Windows service上のCodex sandbox | Runner専用ユーザー・非対話serviceで `windows.sandbox=elevated` が動くかは未検証 | 全reviewが `TOOL_CHECK` 不成立でINCOMPLETE | Runner登録後の最初の `/review` で確認 |
| GitHub Actionsのtag参照 | `actions/*@vN` の改ざん | supply chain | SHA pin化を将来検討 |

Security関連の詳細: `harness/REMOTE_REVIEW_SECURITY.md` §5。

## 17. Product Adoption

1. Private repositoryであることを確認する
2. Product側に `.github/workflows/remote-review.yml` と `scripts/remote-review/` を置く（Pilot実装を参照）
3. `scripts/remote-review/config.json` にProductの実在commandだけを定義する
4. Promptは本COREのtemplateと同一内容をvendorする
5. `AGENTS.md` にRemote Review mode（read-only / no commit）を明記する
6. workflowをdefault branchへmergeする（`issue_comment` はdefault branchのworkflowだけが動く）
7. `harness/REMOTE_REVIEW_SETUP.md` に従ってRunner登録・variables設定を行う
8. AT-01〜AT-20を実行し、結果をProduct側へ記録する

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成（Independent Review・Human承認前）
