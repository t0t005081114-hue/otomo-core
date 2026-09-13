# OTOMO Remote Review Security Policy

- Status: **Draft v0.1** — Security policyはHuman承認が必要（`learning/KNOWLEDGE_PROMOTION.md` §8）。承認までは Pilot（OTOMO LAB）限定の運用基準とする
- Related: `harness/REMOTE_REVIEW.md` / `harness/REMOTE_REVIEW_SETUP.md`

## 1. 前提

`/review` は「このPRのコードを自宅PCで実行する」操作である。

Self-hosted Runnerは永続環境であり、GitHub-hosted runnerのように毎回破棄されない。PRの install / build / lint は、Runnerユーザーの権限で任意コードを実行しうる。

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
| SEC-01 | allowlist（`REMOTE_REVIEW_ALLOWED_USERS`）、`author_association` が OWNER / MEMBER / COLLABORATOR、`type: User`、`created` イベントの場合のみ起動する | workflow `if:` による事前filter + `lib/trigger.mjs` で再検証 |
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

Controlを弱める変更（例: `contents: write` の追加、fork PRの許可、shell実行の導入）は、本Policyの改訂とHuman承認を先に行う。

## 3. Runner環境

最低限（MVP）:

- Runner専用のWindowsローカルユーザー（Administratorsに入れない）でRunner serviceを実行する
- そのユーザーのprofileに、Codex以外の認証情報（Git Credential Manager、`gh`、cloud CLI、SSH key、ブラウザprofile、token入り `.npmrc` 等）を置かない
- 普段使いユーザーのファイルへアクセスさせない（既定のNTFS権限を維持する）
- Codexはそのユーザーで個別にloginする
- workspaceは短いpathに置く（例 `C:\actions-runner\otomo-lab\_work`）

将来候補（Deliberate Non-Goal）: VM / Windows Sandbox / WSL2 / container / ephemeral runner、GitHub Actionsの SHA pin、egress制限、Environment protection rules。

## 4. Operator Rules

1. `/review` は、そのPRのコードを自宅PCで実行してよいと判断したときだけ投稿する
2. allowlistは必要最小限にする（原則Owner本人のみ）
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
| Codex認証情報の窃取 | 同上の任意コードが `~/.codex` を読める | 専用ユーザー、Operator Rule 1 | CodexとPRコードが同一ユーザーで動く |
| Evidence偽装 | 同上の任意コードが、実行中のharnessやevidence fileを改ざんする | trusted inputを事前にメモリへ読み込む、reportで整合を検証 | 同一ユーザー内での改ざんは検知しきれない |
| Local machine compromise | PC自体の侵害 | OS更新、専用ユーザー | 本仕組みの範囲外 |
| Prompt injection | 監査の見逃しを誘導される | untrusted区切り、Promptの優先順位、Deterministic Verificationとの併用 | LLMの性質上ゼロにできない |
| Redactionの取りこぼし | 未知形式のsecret | 環境にsecretを置かない、Artifactの短期保存 | pattern方式の限界 |
| Supply chain（actionsのtag参照） | `actions/*@vN` の改ざん | GitHub公式actionのみ使用 | SHA pinは未実施 |

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成（Human承認前）
