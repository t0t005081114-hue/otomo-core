# OTOMO Remote Review Security Policy

- Status: **Draft v0.1** — Security policyはHuman承認が必要（`learning/KNOWLEDGE_PROMOTION.md` §8）。承認までは Pilot（OTOMO LAB）限定の運用基準とする
- Related: `harness/REMOTE_REVIEW.md` / `harness/REMOTE_REVIEW_SETUP.md`

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
| SEC-23 | Product `AGENTS.md` がRemote Reviewの必須Source of Truthとして列挙した文書と、`codex.context_documents` の内容（`required: true` entryの集合）が一致することを、開発時のtest（CI）で機械的に検証する。必須entryの削除・`required: true → false`への変更・想定外entryの追加・duplicate entryのいずれかがあればtestがFAILする（2026-09-14追加、CORE-RR-IR-002再remediation） | Product `scripts/remote-review/lib/source-of-truth.mjs`（`checkSourceOfTruthManifestConsistency`）+ mutation test |

Controlを弱める変更（例: `contents: write` の追加、fork PRの許可、shell実行の導入）は、本Policyの改訂とHuman承認を先に行う。

## 3. Runner環境

最低限（MVP）:

- Runner専用のWindowsローカルユーザー（Administratorsに入れない）でRunner serviceを実行する
- そのユーザーのprofileに、Codex以外の認証情報（Git Credential Manager、`gh`、cloud CLI、SSH key、ブラウザprofile、token入り `.npmrc` 等）を置かない
- 普段使いユーザーのファイルへアクセスさせない（既定のNTFS権限を維持する）
- Codexはそのユーザーで個別にloginする
- workspaceは短いpathに置く（例 `C:\actions-runner\<repository>\_work`）

将来候補（Deliberate Non-Goal）: VM / Windows Sandbox / WSL2 / container / ephemeral runner、egress制限、Environment protection rules。

## 4. Operator Rules

1. `/review` は、そのPRのcode・`package.json`/lockfile・依存の追加や更新・install/build/testで実行されるscriptを確認し、それらを自宅PCのRunner上で実行してよいと判断したときだけ投稿する（Operator Gate。`harness/REMOTE_REVIEW.md` §1）
2. allowlistは必要最小限にする（原則Owner本人のみ）。allowlistに含まれる全員が、他の全員の進行中reviewをcancelしうることを理解した上で追加する（§5 Residual Risks、`REMOTE_REVIEW.md` §13）
3. Runnerをpublic repositoryへ登録しない
4. Runner登録tokenをファイル・チャット・Obsidianへ保存しない
5. Remote ReviewがPASSでも、merge判断は人間が行う
6. `AUTHENTICATION_FAILURE` の場合は、Runnerユーザーで `codex login` をやり直す
7. 不審なEvidence（予期しないharness変更の警告、read-only境界違反、verdict不整合）が出たら、Runner serviceを停止し、専用ユーザーのprofileを点検する
8. 使わない期間はRunner serviceを停止してよい

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

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成（Human承認前）
- 2026-09-13 v0.1 Draft, Independent Review remediation: Trust Modelを明記し、malicious PR isolation等がExplicit Non-Goalであることを追記。allowlist責務がtrusted scriptに一本化されたことをSEC-01に反映。SEC-19〜SEC-21（Actions SHA pin、Stale Review Protection、Required Context Documents）を追加。Residual RisksとOperator Rulesを更新
- 2026-09-14 文書ドリフト修正（Codex再レビュー前）: §3 Runner環境の将来候補からGitHub Actions SHA pinを削除（SEC-19で実装済みのため重複記述だった）
- 2026-09-14 Independent Re-review remediation（CORE-RR-IR-002未解決分）: SEC-20にhead SHA取得不能時のfail-open除去を追記、SEC-22（Context Documents Manifest Cross-Check）を追加。Residual RisksにGET/POST TOCTOUとManifest Cross-Checkの確認範囲を追加
- 2026-09-14 Independent Re-review remediation（CORE-RR-IR-002再remediation、Blocking）: SEC-23（AGENTS.md ↔ config.json manifest一致の開発時test）を追加。Residual RisksのManifest Cross-Checkの確認範囲を、SEC-23の追加を前提に更新
