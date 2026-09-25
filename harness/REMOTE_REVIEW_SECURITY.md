# OTOMO Remote Review Security Policy

- Status: **Suspended Pilot**（2026-09-25 Human Decision。`harness/REMOTE_REVIEW.md` Suspension）。Suspension時点の版: Draft v0.1 — Security policyはHuman承認が必要（`learning/KNOWLEDGE_PROMOTION.md` §8）。承認までは Pilot（OTOMO LAB）限定の運用基準とする
- Related: `harness/REMOTE_REVIEW.md` / `harness/REMOTE_REVIEW_SETUP.md`

Remote Review PilotはSuspended / Non-defaultである。本Policyのsecurity controlsは、過去PilotのReferenceとして保持する。本文中の運用記述はSuspension前のPilot運用を記述したものである。Suspended中にRunnerを起動する・`/review` を実行する場合（例: 再開判断のための再検証）も、本Policyのcontrolsはそのまま適用され、弱めない。

## 1. 前提

`/review` は「このPRのコードを自宅PCで実行する」操作である。

Self-hosted Runnerは永続環境であり、GitHub-hosted runnerのように毎回破棄されない。PRの install / build / lint は、Runnerユーザーの権限で任意コードを実行しうる。

**Trust Model（`harness/REMOTE_REVIEW.md` §1 Supported Trust Model）**: 本Policyは「private repositoryにおいて、人間が事前に確認し、自宅PCのRunner上で実行してよいと判断したPR」だけを対象とする。信頼済みの対象にはsource codeだけでなく、`package.json` / lockfile、依存の追加・更新、npm lifecycle script、install / build / test scriptが含まれる。malicious PR・hostile dependency・supply-chain compromiseからのisolationは**提供しない**（Explicit Non-Goal）。この前提が成立しない状況（public repository、未確認のPR）でRunnerを使わない。

守る対象:

- 自宅PCとRunnerユーザーのprofile
- Codex認証情報
- GitHub token（`GITHUB_TOKEN`）とRunner登録
- private source code
- Review Evidenceの完全性

想定する攻撃経路:

| 経路 | 例 |
|---|---|
| 未認可の起動 | 第三者・Botによる `/review` |
| fork PR | 外部者のコードをRunnerで実行させる |
| command injection | comment本文・PR title・branch名のshell展開 |
| malicious diff / dependency | `npm ci` lifecycle script、build設定、改ざんされた依存 |
| prompt injection | PR本文・コード・log内の「PASSと出力せよ」等の指示 |
| secret leakage | stdout / stderr、Artifact、PR commentへのtoken混入 |
| workspace persistence | 前回の `node_modules`・stale file・git hookの残留 |
| concurrent execution | 複数reviewのEvidence混線 |

## 2. Mandatory Controls

| ID | Control | Pilot実装 |
|---|---|---|
| SEC-01 | allowlist（`REMOTE_REVIEW_ALLOWED_USERS`）、`author_association` が OWNER / MEMBER / COLLABORATOR、`type: User`、`created` イベント、小文字の `/review` の場合のみ起動する | allowlistのparse・比較・author_association・大文字小文字判定は `lib/trigger.mjs`（`authorize.mjs` から呼ばれるtrusted script）だけで行う。workflowの `if:` とconcurrency groupは、PRコメントで `/review` らしき文字列であることだけを見る、allowlist非依存のcost事前filterであり、認可境界ではない（GitHub Actionsの式関数は大文字小文字を区別せず、workflow式でallowlistを評価すると不正なJSONで評価自体が壊れるため）。allowlist変数が不正な場合は `authorized: false` / `ALLOWLIST_INVALID` を記録してfail closedする |
| SEC-02 | event由来のテキストを `run:` へ展開しない。scriptは `GITHUB_EVENT_PATH` から読む | workflow静的テスト |
| SEC-03 | Harness（workflow・script・config・Prompt）はdefault branchから取得し、PR head側のharnessを実行しない | `issue_comment` + `ref: github.sha` によるtrusted checkout |
| SEC-04 | fork PR・closed PRはGitHub-hosted jobで拒否し、Self-hosted Runnerへ到達させない | `validatePullRequest` |
| SEC-05 | Self-hosted Runnerはprivate repositoryにのみ登録する | Setup Guide / 運用 |
| SEC-06 | 最小権限: top-level `permissions: {}`、self-hosted jobは `contents: read` のみ、`pull-requests: write` はGitHub-hosted jobのみ、`contents: write` は禁止 | workflow静的テスト |
| SEC-07 | すべてのcheckoutで `persist-credentials: false` | workflow静的テスト |
| SEC-08 | job開始時に前回workspaceを削除し、exact SHAでfresh checkoutして `HEAD` の一致を確認する。終了時にcleanupする | workflow + `review.mjs` |
| SEC-09 | checkはtrusted configのargv配列を、shellを介さずに実行する（`npm` / `node` のみ） | `lib/process.mjs`, `lib/config.mjs` |
| SEC-10 | 子processへ機密名の環境変数・`GITHUB_*`・`ACTIONS_*` を渡さない。Codexには認証用の変数だけを追加する | `buildChildEnv` |
| SEC-11 | job / check / Codexにtimeoutを設け、timeout時はprocess treeごと停止する | workflow `timeout-minutes`, `runProcess` |
| SEC-12 | PR単位のconcurrency、runごとのevidence、report側でのrun ID・PR番号・SHA照合 | workflow + `validateMetadata` |
| SEC-13 | Codexは read-only sandbox・user config / execpolicy rules無視・ephemeral・approval never で動かし、前後で `git status` を比較する。Codexが実際にcheckoutを読めたことをgrounding check（`TOOL_CHECK`）で確認し、確認できないreviewは採用しない | `buildCodexArgs`, `verifyToolCheck`, `review.mjs` |
| SEC-14 | untrusted data（PR本文・diff・log・focus）をnonce付きの区切りで渡し、Promptの優先順位を固定する。harness-sensitive fileの変更を警告する | Prompt template, `collectReviewDiff` |
| SEC-15 | 永続化前にredactionとsize制限を行う。raw logをPR commentへ貼らない。分類理由へ外部の生出力を入れない | `lib/redact.mjs`, `lib/checks.mjs`, `classifyCodexFailure` |
| SEC-16 | report jobはevidenceをuntrustedとして検証し、`@mention` を無効化する | `validateMetadata`, `neutralizeMentions` |
| SEC-17 | commit / push / merge / auto-fix の経路を持たない | 静的テスト（AT-16 / AT-19 / AT-20） |
| SEC-18 | Artifactの保存期間は短期（Evidence 14日、PR context 7日） | workflow |
| SEC-19 | 使用する第三者GitHub Actionsはfull commit SHAへpinし、major tag参照にしない | workflow静的テスト（AT-23） |
| SEC-20 | reportがPRへコメントを投稿する直前に、GitHub APIからcurrent PR HEAD SHAを再取得し、reviewしたSHAと異なる場合は投稿するVERDICTをINCOMPLETEに強制する（stale review protection）。取得したSHAが欠落・不正な場合も「一致」とfail-openせず、同様にINCOMPLETEに強制する（2026-09-14追加。GET/POST間のTOCTOUはResidual Risk） | `report.mjs` + unit test（AT-22） |
| SEC-21 | `codex.context_documents` の `required: true` documentが、authorized head SHAのcheckout内で存在・読取可能・非空であることを、Codex起動前にharnessが確認する。満たさない場合はVERDICTをPASSにしない | `lib/context-documents.mjs` + unit test（AT-21） |
| SEC-22 | report jobは、自身が独立にcheckoutしたtrusted `codex.context_documents` manifestと、Evidenceの `context_documents[]` を照合する。`null`・entry欠落・required flag不一致・required entryが非PRESENT・重複/想定外entryのいずれかがあれば、Evidenceの自己申告verdictを採用せずINCOMPLETEとする（2026-09-14追加、CORE-RR-IR-002） | `lib/context-documents.mjs`（`validateContextDocumentManifest`）+ `report.mjs` + unit test（AT-25） |
| SEC-23 | Product `AGENTS.md` がRemote Reviewの必須Source of Truthとして列挙した文書と、`codex.context_documents` の内容（`required: true` entryの集合）が一致することを、開発時のautomated test suiteで機械的に検証する。必須entryの削除・`required: true → false`への変更・想定外entryの追加・duplicate entryのいずれかがあればtestがFAILする（2026-09-14追加、CORE-RR-IR-002再remediation）。通常PRのCIで自動強制する仕組みはまだ無く、`node --test`の手動/harness実行に依存する | Product `scripts/remote-review/lib/source-of-truth.mjs`（`checkSourceOfTruthManifestConsistency`）+ mutation test |
| SEC-24 | report jobは、`codex.status` が `COMPLETED` のEvidenceについて、`codex.verdict` が非COMPLETEDでnull以外でないこと、`tool_check` が `VERIFIED` であること、`workspace_unchanged` が `true` であること、`approval_policy` / `ignore_user_config` / `ignore_rules` がreview job実装の既定値と一致すること、`effective_sandbox` がtrusted config（`codex.windows_sandbox`）とEvidence自身の `run.runner_os` から導かれる期待値と一致すること、`blocking_count` / `advisory_count` が非負整数かつ `verdict` と整合すること、`codex-review.md` を独立に再parseした結果が `metadata.json` の `verdict` / `blocking_count` / `advisory_count` と一致することを検証する。いずれかを満たさなければEvidenceの自己申告verdictを採用せずINCOMPLETEとする（2026-09-14追加、CORE-RR-RR-001） | `lib/evidence.mjs`（`validateMetadata`）+ `lib/codex.mjs`（`expectedEffectiveSandbox`, `validateCodexReviewConsistency`）+ `report.mjs` + unit test（AT-27） |
| SEC-25 | report jobは、自身が独立にcheckoutしたtrusted `config.checks` manifestと、Evidenceの `checks[]` を照合する。trusted manifestの `required: true` checkがEvidence側に欠落・required flag不一致・label不一致、またはEvidence側に重複/想定外entryがあれば、Evidenceの自己申告verdictを採用せずINCOMPLETEとする。必須check一覧はtrusted `config.json` をSource of Truthとし、report job実装へハードコードしない。**definitionの不整合（この行）と、definitionと一致するcheckの実行結果（`FAIL` / `INFRA_ERROR` / `SKIPPED`）は別の状態であり、後者はSEC-26・`harness/REMOTE_REVIEW.md` §7/§8のVerification / Infrastructure semanticsが扱う（この照合単独ではINCOMPLETEにしない）**（2026-09-14追加、CORE-RR-RR-002。2026-09-15、CORE-RR-FRR-002でrequired checkの非PASSを一律INCOMPLETE扱いしていた文言・実装の誤りを修正） | `lib/checks.mjs`（`validateCheckManifest`）+ `report.mjs` + unit test（AT-26） |
| SEC-26 | report jobは、Evidenceの各checkについて、`command` が非nullならtrusted `config.checks` から導出したcanonical invocation（`canonicalCommand()`。`npm_script` checkは `["npm", "run", <npm_script>]`、argv checkはtrusted `command` 配列そのもの）と一致することを、statusにかかわらず検証する。加えて、`runChecks()`/`classifyCheckResult()`（Product `lib/checks.mjs`/`lib/process.mjs`）が実際に生成しうる状態遷移から導いた不変条件を、PASS/FAIL/INFRA_ERROR/SKIPPEDそれぞれについて検証する（PASSは`exit_code === 0`・`timed_out === false`・`skip_reason === null`・`failure === null`、FAILは`failure`が非null・`timed_out === false`で構造的FAILなら`command`/`exit_code`が共にnull・実行されたFAILなら`skip_reason === null`かつ`exit_code`が非null非ゼロ、INFRA_ERRORは`command`が非null・`skip_reason === null`・`failure`が非null、SKIPPEDは`command`/`exit_code`が共にnullで`timed_out === false`・`failure === null`）。いずれかを満たさなければEvidenceの自己申告verdictを採用せずINCOMPLETEとする。command文字列はreview job自身の実行（`planCheck`）とこの照合の両方が同一関数から導出し、別途ハードコードしない（2026-09-15追加、CORE-RR-FRR-001。2026-09-15、CORE-RR-FINAL-001でPASS限定だった適用範囲を全statusへ拡張） | `lib/checks.mjs`（`canonicalCommand`, `validateExecutionState`, `validateCheckManifest`）+ `report.mjs` + unit test（AT-28） |
| SEC-27 | `summarizeVerification` は、Evidenceの各checkが持つ `required` フラグを見て、`required: true` のcheckが1件でも `PASS` 以外であればVerificationを `PASS` にしない（他のrequired checkがPASSしていても、全体のPASSへ到達させない）。また、Codex出力の認識対象section（`## Blocking Findings` 等5種）が重複した場合は `CODEX_FAILURE`（INCOMPLETE）とし、`VERDICT` 行の内容によらずPASS/FAIL verdictを採用しない（後勝ちで上書きされた重複sectionにより、先行する実際のFindingが隠れたままPASSとしてparseされることを防ぐ）（2026-09-15追加、CORE-RR-FRR-002） | `lib/evidence.mjs`（`summarizeVerification`）+ `lib/codex.mjs`（`extractSections`, `parseCodexReview`）+ unit test（AT-29） |

Controlを弱める変更（例: `contents: write` の追加、fork PRの許可、shell実行の導入）は、本Policyの改訂とHuman承認を先に行う。

## 3. Runner環境

最低限（MVP）:

- Runner専用のWindowsローカルユーザー（Administratorsに入れない）のcontextでRunner processを実行する
- そのユーザーのprofileに、Codex以外の認証情報（Git Credential Manager、`gh`、cloud CLI、SSH key、ブラウザprofile、token入り `.npmrc` 等）を置かない
- **普段使いユーザーのprofile / ファイル側**: runner userからアクセスさせない（そちらは既定のNTFS権限を維持する。runner userへ追加の権限を与えない）
- **Runner directory側**: ACLを明示的にhardeningする（**こちらは既定のNTFS権限のままにしない**）。tree全体のACEとownerを `NT AUTHORITY\SYSTEM` / `BUILTIN\Administrators` / 専用runner user に限定し、通常のauthenticated user・`GITHUB_ActionsRunner_*` groupのACEを残さない（下記 Runner Directory Integrity）

この2つは方向が逆であり、対象directoryが異なる。普段使い側は「runner userに読ませない」、Runner directory側は「普段使いアカウントに書かせない」である。
- Codexはそのユーザーで個別にloginする
- workspaceは短いpathに置く（例 `C:\actions-runner\<repository>\_work`）

このsecurity invariantが要求するのは **専用のsecurity principal** であって、Windows Service等のprocess hosting方式そのものではない。Runner processがWindows Serviceとして動くか、専用userの対話sessionから `run.cmd` で動くかは、このinvariantを満たすかどうかを変えない。

### Runner Directory Integrity（2026-09-23追加、CORE-RR-L2-001）

「専用userが非Administratorである」ことは、Runner directoryの**書き込み側**を守らない。Windowsの既定NTFS権限では `NT AUTHORITY\Authenticated Users` に `Modify` が継承されるため、同一PC上の通常アカウント（普段使いアカウントを含む）が、runner binary・script・`_work`・`_diag` を、専用runner userの権限で実行される **前に** 差し替えられる。これはCodex認証情報を持つprincipalでの任意コード実行に直結し、§5 Residual Risksの「Runnerの永続汚染」「Codex認証情報の窃取」「Evidence偽装」を、PR由来ではなくlocal account由来で成立させる。

したがって、専用user分離は次の2つをセットで要求する。

1. **実行側**: Runner processが専用の非Administrator userのcontextで動く
2. **書き込み側**: Runner directoryのroot・すべてのdescendantのACEとownerが、SYSTEM / Administrators / 専用runner user に限定されている。Service modeでrunnerが作る `GITHUB_ActionsRunner_*` groupも許可principalに含めない（そのgroupとmembershipの正当性を汎用に検証できないため。2026-09-23 Round 2で改訂）

2が無ければ1は保証にならない。手順・検証コマンドは `harness/REMOTE_REVIEW_SETUP.md` §5.1、Acceptance条件は `harness/REMOTE_REVIEW.md` §20。

専用runner userからwrite権限を奪わないこと（`_work` / `_diag` / runner-owned stateへ書けなければrunnerは動かない）。

将来候補（Deliberate Non-Goal）: VM / Windows Sandbox / WSL2 / container / ephemeral runner、egress制限、Environment protection rules。

### Runner Mode（2026-09-23 Human Decision）

Runner Modeの定義とAcceptance要求は `harness/REMOTE_REVIEW.md` §6 / §20が所有する。本節はsecurity上の残差だけを記録する。本Pilotが採用するのは **Interactive mode** である。

どちらのmodeでも、上記の専用ユーザー分離要求は同一であり、弱めない。

| 観点 | Interactive | Service |
|---|---|---|
| 起動条件 | 専用runner userのサインインsessionが必要 | サインインなしで動作しうる |
| 自動起動 | しない。再起動・logoff後は人間が再度起動する | 自動起動を設定しうる |
| 露出時間（unattended availability window） | 人間が `run.cmd` を起動している間だけ。狭い | 常時受付を設定した場合は広い |
| 停止時 | `run.cmd` を停止するとrunnerはOffline / 利用不可 | serviceを停止するとOffline / 利用不可 |
| Codex / sandbox互換性 | 専用userの対話sessionで動くため、`harness/REMOTE_REVIEW_SETUP.md` §4のsmoke testと同じ条件 | 非対話service contextに由来する追加のCodex / sandbox互換性の懸念を持ちうる（未検証） |

**どちらのmodeが普遍的に安全というものではない。** Interactive modeは露出時間を人間が制御できる一方、runnerの可用性は人間の操作に依存する。Service modeは可用性が高い一方、unattendedな受付時間が長くなり、非対話contextの挙動が追加で未検証になる。Mode選択はこのtrade-offに対するHuman Decisionであり、本Policyの他のControlを置き換えない。

## 4. Operator Rules

1. `/review` は、そのPRのcode・`package.json`/lockfile・依存の追加や更新・install/build/testで実行されるscriptを確認し、それらを自宅PCのRunner上で実行してよいと判断したときだけ投稿する（Operator Gate。`harness/REMOTE_REVIEW.md` §1）
2. allowlistは必要最小限にする（原則Owner本人のみ）。allowlistに含まれる全員が、他の全員の進行中reviewをcancelしうることを理解した上で追加する（§5 Residual Risks、`REMOTE_REVIEW.md` §13）
3. Runnerをpublic repositoryへ登録しない
4. Runner登録tokenをファイル・チャット・Obsidianへ保存しない
5. Remote ReviewがPASSでも、merge判断は人間が行う
6. `AUTHENTICATION_FAILURE` の場合は、Runnerユーザーで `codex login` をやり直す
7. 不審なEvidence（予期しないharness変更の警告、read-only境界違反、verdict不整合）が出たら、Runner processを停止し、専用ユーザーのprofileを点検する
8. 使わない期間はRunner processを停止してよい。Interactive modeでは、必要なときだけ起動する運用を前提とする（§3 Runner Mode）

Runner processの停止は、採用しているRunner Mode（`harness/REMOTE_REVIEW.md` §6）に従う。

- Interactive mode: 稼働中のrunner process（`run.cmd`）を停止する
- Service mode: 該当のWindows Serviceを停止する

いずれの場合も、停止後にGitHubのRunners画面でOfflineになっていることを確認する。Offlineであれば新しいRemote Review jobは開始されない。

## 5. Residual Risks（Security）

| Risk | 内容 | 現状の緩和 | 残る理由 |
|---|---|---|---|
| Runnerの永続汚染 | PRの install / build がRunnerユーザー権限で任意コードを実行し、user-levelの git / npm / Codex 設定等を改変して次回以降に影響する | 専用ユーザー、workspace削除、allowlist、fork拒否 | 永続Runnerでは完全に防げない。ephemeral化が必要 |
| Codex認証情報の窃取 | 同上の任意コードが、Runnerユーザーのprofile内にあるCodex認証情報を読める | 専用ユーザー、Operator Rule 1 | CodexとPRコードが同一ユーザーで動く |
| Evidence偽装 | 同上の任意コードが、実行中のharnessやevidence fileを改ざんする | trusted inputを事前にメモリへ読み込む、reportで整合を検証 | 同一ユーザー内での改ざんは検知しきれない |
| Local machine compromise | PC自体の侵害 | OS更新、専用ユーザー | 本仕組みの範囲外 |
| Prompt injection | 監査の見逃しを誘導される | untrusted区切り、Promptの優先順位、Deterministic Verificationとの併用 | LLMの性質上ゼロにできない |
| Redactionの取りこぼし | 未知形式のsecret | 環境にsecretを置かない、Artifactの短期保存 | pattern方式の限界 |
| Supply chain（GitHub Actionsのリリース内容） | pin先のfull commit SHAが指すコード自体が、将来のリリースで悪意ある変更を含む可能性 | GitHub公式actionのみ使用、full commit SHAへpin（pin時に内容を確認） | pinはSHAの指す内容を固定するだけで、そのSHA自体の安全性を継続監査しない |
| Concurrency groupがallowlist非依存 | allowlistに含まれないユーザーの `/review` らしきコメントが、進行中の認可済みreviewをcancelしうる（`REMOTE_REVIEW.md` §13） | 実行済みreviewの喪失・再実行の手間。unauthorized runがrunnerへ到達することはない | private repositoryのcollaborator数を絞る運用 |
| Required Context Documentsの確認範囲 | 「checkout内にfileとして存在し読めるか」だけを確認し、「Codexが実際に読んだか」は確認しない | 必須文書が存在してもCodexが読まずに判断する可能性は残る | Verification Evidence欄の記述を人間が確認する |
| Stale Review Protectionの GET/POST TOCTOU（2026-09-14追加） | report jobが現在のHEAD SHAを確認した直後から、PRコメントをPOSTするまでの間に新しいcommitがpushされる | 投稿されたコメントの「一致」判定は投稿直前の一時点のもの | GitHub REST APIにcompare-and-swapに相当する原子的操作が無い。人間はreviewed SHAを見て必要なら再度 `/review` する |
| Context Documents Manifest Cross-Checkの確認範囲（2026-09-14追加、SEC-23で範囲を更新） | report jobはtrusted manifestとEvidenceの `context_documents[]` の整合だけを確認する。「trusted manifestに載っている文書がProductの実際のSource of Truthとして正しいか」自体はharnessが判断しない。AGENTS.mdとconfig.jsonの一致自体はSEC-23（開発時test）が機械的に検証するため、CORE-RR-IR-002の主因（一方だけ変更してもう一方を更新し忘れる）はSEC-23で塞がれている | SEC-23（開発時test）。AGENTS.mdが列挙する文書の集合そのものがProductの実際のSource of Truthとして正しいかどうかはSEC-23の範囲外 | `harness/REMOTE_REVIEW.md` §9 Source of Truth Manifest Consistencyに基づく人間・Independent Reviewでの確認 |
| Verification Checks / Codex Evidence Semantic Invariantsの確認範囲（2026-09-14追加、SEC-24 / SEC-25） | SEC-24の `effective_sandbox` cross-checkは、Evidenceの `run.runner_os`（review job = harness自身が`RUNNER_OS`から記録する自己申告値）を前提とし、report jobがOSそのものを独立に再測定するわけではない。SEC-24〜SEC-27はいずれも「trusted configとEvidenceの整合」を検証するのであって、「trusted configの必須check一覧・sandbox設定自体が正しいか」は判断しない | harnessそのもの（trusted script）が信頼できる、という既存Trust Model（`harness/REMOTE_REVIEW.md` §1）の範囲内でのみ有効 | 既存Trust Model・Operator Gateの範囲外の脅威として扱う。将来のRunner Acceptance Gate（`harness/REMOTE_REVIEW.md` §20）強化時に合わせて見直す |
| Runner directory ACLの継続性（2026-09-23追加、CORE-RR-L2-001） | ACL hardeningはRunner Acceptance時点の一度きりの検証であり、その後にAdministratorが権限を戻す・runnerを別pathへ再配置する・Windowsの操作で継承が復活する、といった変化を継続監視する仕組みは無い | Acceptance時の必須check（`harness/REMOTE_REVIEW.md` §20）、Durable Acceptance Recordへの記録 | 継続的なACL監視は実装していない。Runner directoryを作り直した場合・Mode変更で再登録した場合は、§20のACL検証を再実施する |
| Principal Identity Evidenceの範囲（2026-09-23追加、CORE-RR-L2-002） | process ownerの確認は、Acceptance時にoperatorが取得した一時点の証跡である。以後のすべてのreview runが同じprincipalで実行されたことを、harnessが自動で再検証するわけではない | Acceptance時の機械的確認とrun bindingにより、「申告だけのPASS」は防ぐ | 各review runごとのprincipal再検証はEvidence schemaの変更を伴うため、本変更の範囲外（`harness/REMOTE_REVIEW.md` §20）。専用userのpasswordが共有されない運用前提に依存する |
| Runner directory hardening中のrace（2026-09-23追加、Round 3） | recursiveなACL / owner変更（`icacls /T`）はdirectory junctionを辿る。hardeningはrootを隔離し、reparse pointを辿らない棚卸しとdirectory footholdの除去を済ませてからrecursiveな変更を行うが、最後の棚卸しと変更の間にAdministrator / SYSTEM、または動作中のrunner userのprocessがjunctionを置けば、変更はそれを辿りうる。また、rootを隔離する前の短い間に、通常アカウントがrootをjunctionへ差し替えると、root単独の変更がその先の1 objectに及びうる | 通常アカウントは、root隔離とfoothold除去の後はtree内に新しいentryを作れない。root差し替えはfile IDとreparse point属性の再確認でFAILになる。変更後の検証（`ACL CHECK`）は引き続き必須 | Administratorsの信頼は既存Trust Modelの前提のまま。runner停止とrunner userのsign outはoperatorが確認し、scriptは確認しない。root差し替えによる1 objectの変更そのものは防げない |
| Runner binding evidenceの前提（2026-09-23追加、CORE-RR-L2-002 Round 2） | installation・process・local登録・GitHub job・job環境の連鎖（`harness/REMOTE_REVIEW.md` §20）は、Administratorsを信頼する既存Trust Modelの範囲内で成立する。Administratorは、登録情報を別pathへ複製して別名の実行fileで動かす・`_diag` のlogを書き換える、といった操作ができる。local traceは runner 2.337.0 の `_diag` Worker logの形式に依存する。timingは、runner PCの時刻同期と60秒の余裕に依存する。Service modeでrunner groupのACEを除去した状態の運用は未検証 | 非Administratorの通常アカウントは、tree全体のACL hardeningにより登録情報を読めず、runner fileを書き換えられない。形式が変わった・時刻を確認できない場合はfail closed（Acceptance FAIL） | Administratorアカウントの管理はTrust Modelの前提のまま。runner更新でWorker logの形式が変わった場合は `harness/REMOTE_REVIEW_SETUP.md` §6.1を更新する。Service modeは、採用時にそのmodeのAcceptanceで確認する |
| Runner Mode判定の前提（2026-09-23追加、CORE-RR-L2-003） | Runner Modeの判定（`harness/REMOTE_REVIEW.md` §20）は、Acceptance時に取得した一時点のprocess情報（親process chain・session・command line）とservice登録状態に基づく。Windowsでは、processの作成者は親processを指定できる（parent process spoofing）。そのため、runner userの権限でListenerを起動できる主体（runner user自身、Administrator / SYSTEM）は、親processの連鎖を偽装しうる。Service modeの判定連鎖は、runner 2.337.0の実行fileと登録済みのservice設定から導いたものであり、実際のService mode runnerでは実行していない | Listenerのownerが専用runner userであることを別に要求するため、通常アカウントが起動したprocessはListenerとして受け入れられない。判定できない・矛盾する場合はfail closed（Acceptance FAIL）。対象installationのservice登録が残っていればInteractiveと判定しない | runner userとAdministratorsの信頼は既存Trust Modelの前提のまま。modeの継続監視はしない（Acceptance後にmodeを変えた場合は §20 Mode変更時に従う）。Service modeを採用するときは、そのmodeのAcceptanceで判定連鎖を確認する |
| Runner Modeの運用差（2026-09-23追加、§3 Runner Mode） | Interactive modeでは、runnerの稼働・停止が人間の操作に依存する。`run.cmd` を起動したまま放置すれば露出時間はService modeと同様に長くなり、逆に起動を忘れればreviewがqueuedのまま残る。またInteractive modeでAcceptanceを通しても、Service modeの非対話context固有の挙動は検証されない | 専用ユーザー分離（§3）はmodeによらず維持される。Mode固有checkとMode変更時の再検証を`harness/REMOTE_REVIEW.md` §20で要求する | 露出時間の制御はoperatorの運用に依存し、技術的に強制していない。Service modeの非対話contextでのCodex / sandbox挙動は未検証のまま残る |
| Canonical Command Bindingの確認範囲（2026-09-15追加、SEC-26） | SEC-26は「自己申告する `command` がtrusted definitionの導出結果と文字列一致するか」を確認するのであって、trusted `config.json` 自身が定義する `command` / `npm_script` の内容が安全・妥当かは判断しない（config.jsonの内容はharness trusted scriptの一部として既存Trust Modelの範囲内） | 既存Trust Modelの範囲内でのみ有効 | config.jsonの変更はProductのcode reviewで確認する（既存運用） |

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成（Human承認前）
- 2026-09-13 v0.1 Draft, Independent Review remediation: Trust Modelを明記し、malicious PR isolation等がExplicit Non-Goalであることを追記。allowlist責務がtrusted scriptに一本化されたことをSEC-01に反映。SEC-19〜SEC-21（Actions SHA pin、Stale Review Protection、Required Context Documents）を追加。Residual RisksとOperator Rulesを更新
- 2026-09-14 文書ドリフト修正（Codex再レビュー前）: §3 Runner環境の将来候補からGitHub Actions SHA pinを削除（SEC-19で実装済みのため重複記述だった）
- 2026-09-14 Independent Re-review remediation（CORE-RR-IR-002未解決分）: SEC-20にhead SHA取得不能時のfail-open除去を追記、SEC-22（Context Documents Manifest Cross-Check）を追加。Residual RisksにGET/POST TOCTOUとManifest Cross-Checkの確認範囲を追加
- 2026-09-14 Independent Re-review remediation（CORE-RR-IR-002再remediation、Blocking）: SEC-23（AGENTS.md ↔ config.json manifest一致の開発時test）を追加。Residual RisksのManifest Cross-Checkの確認範囲を、SEC-23の追加を前提に更新
- 2026-09-14 文書ドリフト修正（Independent Re-review前）: SEC-23の表現を実装実態へ整合（「開発時のtest（CI）」→「開発時のautomated test suite」。通常PRのCIによる自動強制はまだ無く、実装は`node --test`のharness/手動実行に依存する旨を明記。新規のCI強制は実装していない）
- 2026-09-14 Independent Re-review remediation（CORE-RR-RR-001 / CORE-RR-RR-002、Blocking）: SEC-24（report jobによるCodex Evidence semantic invariantの検証 — status/verdict整合、tool_check、workspace_unchanged、invocation policy、effective_sandboxのtrusted config/platform整合、blocking_count/advisory_countの整合、codex-review.mdの再parse照合）とSEC-25（report jobによるtrusted `config.checks` manifestとEvidence `checks[]` のcross-check）を追加。Residual Risksに、SEC-24/SEC-25がEvidenceの自己申告 `run.runner_os` を前提とすることと、trusted config自体の正しさは判断しないことを追加
- 2026-09-15 Independent Re-review remediation（CORE-RR-FRR-001 / CORE-RR-FRR-002、Blocking。ChatGPT独立監査によるVerification semantics regressionの指摘を含む）: SEC-25の文言・実装が「trusted manifestの `required: true` checkがEvidence側で非PASS」であることそのものをEvidence整合性問題としており、正常に実行・記録された `FAIL` / `INFRA_ERROR` までEvidence corruptionと混同していた誤りを修正（definition整合性とexecution result semanticsを分離）。SEC-26（report jobによる、`status: PASS` を自己申告するcheckのcanonical command binding — `canonicalCommand()` で導出したtrusted invocationとの一致、`exit_code`/`timed_out`/`skip_reason`/`failure`の検証）とSEC-27（`summarizeVerification` のrequired-check-all-PASS要件、およびCodex出力の重複section検出）を追加。Residual RisksにCanonical Command Bindingの確認範囲を追加
- 2026-09-15 最終Independent Re-review remediation（CORE-RR-FINAL-001、Blocking）: SEC-26のcanonical command binding検証が `status: PASS` を自己申告するcheckだけに限定されており、`status: FAIL`（`failure`は正当に見える）だが `command` が別checkのものへ差し替えられたEvidenceを素通りさせていた誤りを修正。SEC-26の適用範囲を「`command` が非nullの全check」（statusによらない）へ拡張し、PASS/FAIL/INFRA_ERROR/SKIPPEDそれぞれについて`runChecks()`/`classifyCheckResult()`の実際の状態遷移から導いた不変条件（`lib/checks.mjs` `validateExecutionState`）を追加検証する。新しいSecurity Controlではなく既存SEC-26の適用範囲の是正であり、新規SEC番号は割り当てない
- 2026-09-23 Runner Mode採用（Human Decision、docs-only、Forced Level 2扱い）: §3 Runner環境のsecurity invariantが `Runner serviceを実行する` とprocess hosting方式に結合していたため、`Runner processを実行する` というmode中立な表現へ変更し、invariantが要求するのは専用のsecurity principalであってWindows Service等のhosting方式ではないことを明記した。§3へRunner Mode（Interactive / Service）のsecurity残差表を追加し、起動条件・自動起動・露出時間・停止時・Codex / sandbox互換性の差を記録した（どちらのmodeも普遍的に安全とは主張しない）。§4 Operator Rules 7 / 8の `Runner service` をmode中立な `Runner process` へ変更し、Interactive modeでは `run.cmd` の停止、Service modeではserviceの停止という同等の操作を定義した上で、停止後にGitHub上でOfflineであることの確認を追加した。§5へRunner Modeの運用差をResidual Riskとして追加した。**専用ユーザー分離要求（Administratorsに入れない、Codex以外の認証情報を置かない、普段使いユーザーを使わない）は一切弱めていない。** SEC-01〜SEC-27、Trust Model、Mandatory Controls、Operator Gate、Explicit Non-Goals、Status（Draft v0.1）に変更はない
- 2026-09-23 L2 Independent Review remediation Round 1（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: §3 Runner環境が「既定のNTFS権限を維持する」とだけ書いており、Windows既定では `NT AUTHORITY\Authenticated Users` に `Modify` が継承されるため、専用user分離の**書き込み側**が保護されていなかった（実機のPilot runner directoryで確認）。§3へ Runner Directory Integrity を追加し、専用user分離が「実行側（非Administratorの専用userで動く）」と「書き込み側（Runner directory ACLをSYSTEM / Administrators / 専用runner userへ限定）」の2つをセットで要求することを明記した。ACL hardeningの必須要求を§3の最低限リストへ追加。§5 Residual Risksへ、ACL hardeningがAcceptance時点の一度きりの検証であること（継続監視は未実装）と、Principal Identity EvidenceがAcceptance時の一時点の証跡であること（review runごとの再検証はEvidence schema変更を伴うため範囲外）を追加した。**SEC-01〜SEC-27、Trust Model、Mandatory Controls、Operator Gate、Explicit Non-Goals、専用ユーザー分離の既存要求、Status（Draft v0.1）は変更していない。弱めた項目は無い**
- 2026-09-23 L2 Independent Re-review remediation Round 2（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: §3の最低限リストとRunner Directory Integrityを、Runner directoryのroot・すべてのdescendantのACEとownerを SYSTEM / Administrators / 専用runner user に限定する、tree全体の要求へ改訂した。Service modeでrunnerが作る `GITHUB_ActionsRunner_*` groupを許可principalから外した（groupとmembershipの正当性を汎用に検証できないため。許可集合を狭める変更であり、弱めてはいない）。§5 Residual Risksへ、runner binding evidenceがAdministratorsを信頼するTrust Model・runner 2.337.0 のWorker log形式・時刻同期に依存することを追加した。**SEC-01〜SEC-27、Trust Model、Mandatory Controls、Operator Gate、Explicit Non-Goals、専用ユーザー分離の既存要求、Status（Draft v0.1）は変更していない。弱めた項目は無い**
- 2026-09-23 L2 Independent Re-review remediation Round 3（pre-mutation reparse-point handling、Blocking、docs-only）: §5 Residual Risksへ、Runner directory hardening中のrace（Administrator / SYSTEM・動作中のrunner user・root隔離前のroot差し替え）を追加した。手順は `harness/REMOTE_REVIEW_SETUP.md` §5.1。**SEC-01〜SEC-27、Trust Model、Mandatory Controls、Operator Gate、Explicit Non-Goals、専用ユーザー分離の既存要求、Status（Draft v0.1）は変更していない。弱めた項目は無い**
- 2026-09-23 L2 Independent Re-review remediation Round 4（CORE-RR-L2-003、Blocking、docs-only）: Runner Modeを受け入れ対象Listenerのhostingから機械的に判定する要求（`harness/REMOTE_REVIEW.md` §20、手順は `harness/REMOTE_REVIEW_SETUP.md` §6.1）の追加に合わせて、§5 Residual Risksへ「Runner Mode判定の前提」を追加した（一時点の判定、parent process spoofingはrunner user / Administrator / SYSTEMに限られること、Service modeの判定連鎖が実service runnerでは未実行であること）。**SEC-01〜SEC-27、Trust Model、Mandatory Controls、Operator Gate、Explicit Non-Goals、専用ユーザー分離の既存要求、Status（Draft v0.1）は変更していない。弱めた項目は無い**
- 2026-09-25 Suspended Pilot（Human Decision、docs-only、Forced Level 2扱い）: Remote Review PilotのSuspension（`harness/REMOTE_REVIEW.md` Suspension）に合わせてStatusを更新し、本Policyを過去PilotのReferenceとして保持すること、Suspended中にRunnerを起動する場合も本Policyのcontrolsを弱めずに適用することを明記した。**SEC-01〜SEC-27、Trust Model、Mandatory Controls、Runner環境、Operator Rules、Residual Risksの内容は変更していない。弱めた項目は無い**
