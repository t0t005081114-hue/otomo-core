# OTOMO Remote Review Setup Guide

- Status: Draft v0.1（Pilot: OTOMO LAB）
- 対象: Runnerを登録・運用する人間（Owner）
- Related: `harness/REMOTE_REVIEW.md` / `harness/REMOTE_REVIEW_SECURITY.md`

Claude Code / Codexでは完了できない手動設定をまとめる。Runner登録token・password・Codex認証情報は、ファイル・チャット・Obsidianへ保存しない。

## 0. Pilot values

| 項目 | OTOMO LAB |
|---|---|
| Repository | `t0t005081114-hue/otomo-lab`（private） |
| Workflow | `.github/workflows/remote-review.yml` |
| Runner labels | `self-hosted`, `windows`, `x64`, `otomo-review` |
| Runner directory（推奨） | `C:\actions-runner\<repository>` |
| Runner専用Windowsユーザー（推奨名） | `<runner-user>` |
| Node.js | `package.json` の engines は `>=22.12.0`（Node.js 24 LTS推奨） |

## 1. GitHub: workflowをdefault branchへ入れる

`issue_comment` トリガーは **default branch（main）にあるworkflowだけ** が動く。Remote Review導入PRをmergeするまで `/review` には反応しない。

1. Remote Review導入PRを確認し、mergeする（人間判断）
2. Settings → Actions → General を確認する
   - Actions permissions: 有効
   - Workflow permissions: **Read repository contents and packages permissions**（変更しない。workflowが必要な権限をjob単位で宣言している）

## 2. GitHub: Repository variables

Settings → Secrets and variables → Actions → **Variables** タブ → New repository variable

| Name | 必須 | 値の例 | 説明 |
|---|---|---|---|
| `REMOTE_REVIEW_ALLOWED_USERS` | 必須 | `["<github-login>"]` | `/review` を実行できるGitHub login（JSON配列）。未設定なら誰も起動できない |
| `REMOTE_REVIEW_RUNS_ON` | 任意 | `["self-hosted","windows","x64","otomo-review"]` | Runner labelを変える場合だけ設定 |
| `REMOTE_REVIEW_CODEX_BIN` | 任意 | `C:\Users\<runner-user>\AppData\Roaming\npm\node_modules\@openai\codex\bin\codex.js` | Codex CLIを自動検出できない場合だけ設定（絶対path、`.exe` または `.js`。`.cmd` は不可） |
| `REMOTE_REVIEW_CODEX_MODEL` | 任意 | （空） | Codexのmodelを固定したい場合だけ設定 |

- 機密ではないので **Secrets ではなく Variables** に入れる
- JSONが不正だと、PRへのコメントのたびにworkflowがエラーになる

## 3. Windows: Runner専用ユーザー

普段使いのWindowsユーザーでRunnerを動かさない（`harness/REMOTE_REVIEW_SECURITY.md` §3）。

管理者PowerShellで作成する:

```powershell
net user <runner-user> * /add
```

- passwordは対話で入力する（password manager以外に保存しない）
- Administratorsグループに追加しない
- 一度 `<runner-user>` でWindowsへサインインしてprofileを作る（§4のCodex loginもこのユーザーで行う）
- このユーザーに、Codex以外の認証情報（GitHub CLI、Git Credential Manager、SSH key、cloud CLI、ブラウザのlogin）を置かない

## 4. Windows: 必要なツール

| ツール | install | `<runner-user>` で確認 |
|---|---|---|
| Node.js 24 LTS | 公式installer（全ユーザー向け） | `node -v` |
| Git for Windows | 公式installer（全ユーザー向け） | `git --version` |
| Codex CLI | `<runner-user>` でサインインして `npm install -g @openai/codex` | `codex --version` |

長いpath対策（管理者PowerShell）:

```powershell
git config --system core.longpaths true
```

Codex login（`<runner-user>` のセッションで）:

```powershell
codex login              # ブラウザでChatGPTにlogin
# ブラウザを使えない場合: codex login --device-auth
codex login status       # "Logged in" を確認
```

Codex smoke test（`<runner-user>` のセッションで。**Codexが実際にコマンドを実行できること**まで確認する）:

```powershell
mkdir $env:TEMP\codex-smoke
cd $env:TEMP\codex-smoke
git init
codex exec --sandbox read-only --ephemeral --ignore-user-config -c windows.sandbox=elevated "Run the command: git --version. Reply with its exact output only."
```

- `blocked by policy` と表示される場合は、Codexのread-only sandboxがコマンドを実行できていない。`-c windows.sandbox=elevated` を付けているか確認し、このユーザーの対話セッションでWindows sandboxの初期設定を完了させる（未検証: Runner専用ユーザーで初期設定が求められるかどうかと、その手順）

- `git version ...` が返ればよい
- 文章だけが返り、出力に `host executable was not found` や `Code Mode is unavailable` が含まれる場合は、Codexがコマンドを実行できていない（Remote Reviewでは `ENVIRONMENT_FAILURE` / INCOMPLETE になる）
- Runnerには npm版 `@openai/codex` を使う。デスクトップアプリ同梱のCLIは、補助実行ファイル（tool host）が同じ場所に無いとコマンドを実行できないことを2026-09-13に確認した
- Windows sandboxの初期設定を求められた場合は、このユーザーのセッションで完了させる

- 未検証: Windows service（非対話セッション）から起動したときに、Codexのread-only sandboxが追加設定なしで動くか。§6の実行結果で確認する
- Playwrightを使うProductだけ: `<runner-user>` で対象repoをcheckoutして `npx playwright install`（OTOMO LABは現時点で不要）

PCの電源設定: Runnerを使う間はスリープしない設定にする。

## 5. Runner登録（Windows x64）

1. GitHub: 対象repository → Settings → Actions → Runners → **New self-hosted runner** → **Windows** → **x64**
2. 管理者PowerShellで、短いpathにフォルダを作る:

   ```powershell
   mkdir C:\actions-runner\<repository>
   cd C:\actions-runner\<repository>
   ```

3. GitHub画面の **Download** 欄のコマンド（`Invoke-WebRequest`、hash確認、展開）をそのまま実行する
4. GitHub画面の **Configure** 欄の `./config.cmd --url ... --token ...` を実行し、対話で入力する:

   | 質問 | 入力 |
   |---|---|
   | runner group | Enter（Default） |
   | runner name | 任意（個人名・PC名を含めない名前） |
   | additional labels | `otomo-review` |
   | work folder | Enter（`_work`） |
   | run as service | `Y` |
   | service account | `.\<runner-user>` と、そのpassword |

   - tokenは短時間で失効する一時tokenだが、保存しない
   - `self-hosted` / `Windows` / `X64` のlabelは自動で付く（labelの大文字小文字は区別されない）

5. 確認:

   ```powershell
   Get-Service "actions.runner.*"
   ```

   Status が `Running`、GitHub の Runners 画面で **Idle** になっていればよい。

Node.js・Git・Codexを後からinstall / 更新した場合は、serviceを再起動する:

```powershell
Restart-Service "actions.runner.*"
```

## 6. 動作確認（Acceptance Tests）

`harness/REMOTE_REVIEW.md` §15 のうち、GitHub上でしか確認できない項目を実施する。

1. 確認用PRを作る（mergeしない小さな変更）
2. **AT-01**: PRに `LGTM` とコメント → Actionsの Remote Review run で `Authorize /review` が skipped、PRには何も投稿されない
3. **AT-02 / AT-05〜AT-13 / AT-15**: `/review` とコメント
   - 「Remote Independent Review — queued」コメントが付く
   - 数分後に「Remote Independent Review」結果コメントが付く
   - `Commit:` のSHAがPRの最新commitと一致する
   - Run page → Artifacts → `remote-review-evidence` に `metadata.json`、`review-evidence.md`、`install.log`、`lint.log`、`typecheck.log`、`build.log`、`codex-review.md` がある
4. 重点監査: 次のようにコメントし、Artifactの `codex-prompt.md` の FOCUS 欄に反映されることを確認する

   ```text
   /review
   重点監査：
   - auth
   - error handling
   - mobile runtime
   ```

5. **AT-03**: Issueに `/review` とコメント → run が skipped
6. **AT-04**: allowlist外のアカウント（あれば）で `/review` → skipped。第2アカウントが無い場合は「未検証（unit testのみ）」と記録する
7. **AT-14**: variable `REMOTE_REVIEW_CODEX_BIN` を一時的に `C:\missing\codex.exe` にして `/review` → `VERDICT: INCOMPLETE` / Infrastructure `FAILED` → variableを削除して元に戻す
8. **AT-18**: `/review` を続けて2回投稿 → 古いrunが cancelled になり、新しいrunだけが結果をコメントする
9. **AT-16 / AT-19 / AT-20**: PRと `main` にcommitが増えていない、PRがmergeされていないことを確認する
10. **AT-17**: Artifactの中身を `ghp_`、`ghs_`、`github_pat_`、`sk-`、`Bearer`、`password` で検索し、秘密値が無いことを確認する
11. 結果をProduct側に記録する（記録するかどうか・どこに残すかは人間判断）

## 7. Troubleshooting

| 症状 | 原因候補 | 対処 |
|---|---|---|
| `/review` してもrunが出ない | workflowがmainに無い | §1 |
| runはあるが `Authorize /review` が skipped | `REMOTE_REVIEW_ALLOWED_USERS` 未設定・loginが含まれない | §2 |
| workflow run自体がエラー | variableのJSONが不正 | 値を修正する |
| `Verify and review (self-hosted)` が Queued のまま | Runner offline、label不一致、PCスリープ | Runners画面で Idle を確認、`REMOTE_REVIEW_RUNS_ON` を確認 |
| INCOMPLETE + `AUTHENTICATION_FAILURE` | Codex loginの期限切れ | `<runner-user>` で `codex login` |
| INCOMPLETE + `Codex CLI was not found` | serviceから `codex` が見えない | `REMOTE_REVIEW_CODEX_BIN` を設定、service再起動 |
| Install が INFRA_ERROR（EPERM / EBUSY） | antivirusのスキャン、前回processの残留 | 再度 `/review`、service再起動 |
| Install が INFRA_ERROR（network） | npm registryに届かない | 再度 `/review` |
| Format check が多数のfileでFAILし、Infrastructureに `CRLF` の記録がある | checkoutで改行がCRLFへ変換された | workflowの `GIT_CONFIG_*`（`core.autocrlf=false`）が残っているか確認する |
| path too long | Windows MAX_PATH | Runner directoryを短くする、`core.longpaths` |
| INCOMPLETE + `TOOL_CHECK` の記録（UNAVAILABLE / MISSING） | Codexがread-onlyコマンドを実行できなかった（例: Windows sandbox未設定で `blocked by policy`） | `codex.log` を確認し、§4 のsmoke testをRunnerユーザーでやり直す |
| INCOMPLETE + `Codex could not start its tool host` | Codex CLIの補助実行ファイル（tool host）が無い | npm版 `@openai/codex` を使う、`REMOTE_REVIEW_CODEX_BIN` を確認、§4 のsmoke testをやり直す |
| Codexが境界違反（checkout changed） | Codex CLIの挙動変化 | Runnerを停止し、Codex CLIのversionと `codex.log` を確認 |

## 8. 停止・削除

- 一時停止: `Stop-Service "actions.runner.*"`
- 削除: Runners画面 → 対象Runner → Remove で表示される `./config.cmd remove --token ...` を実行し、`C:\actions-runner\<repository>` を削除する
- 侵害が疑われる場合: service停止 → Runner削除 → ChatGPT側で該当ログインのsessionを確認 → `<runner-user>` ユーザーをprofileごと削除して作り直す

## 9. Cost note

- `authorize` / `report` jobは GitHub-hosted runner（ubuntu）で動くため、private repositoryではGitHub Actionsの利用時間を消費する（1回あたり数分未満の見込み（推測））
- self-hosted runnerの利用に対するGitHub側の課金有無は 未確認。導入前にGitHubの最新のBilling documentationを確認する

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成
