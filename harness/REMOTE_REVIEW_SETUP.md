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
| Runner Mode | **Interactive**（2026-09-23 Human Decision。`harness/REMOTE_REVIEW.md` §6） |
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
- JSONが不正な場合、workflow自体はエラーにならない。trusted `authorize.mjs` が `ALLOWLIST_INVALID` として fail closed し、Self-hosted review jobへは到達しない（`harness/REMOTE_REVIEW.md` §5）

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

Codex smoke test（`<runner-user>` のセッションで。**Codexが実際にコマンドを実行できること**まで確認する。本番の呼び出し（`lib/codex.mjs` の `buildCodexArgs`）に近づけるため、promptはCLI引数ではなくstdinで渡し、`--ignore-rules` と `approval_policy="never"` も付ける）:

```powershell
mkdir $env:TEMP\codex-smoke
cd $env:TEMP\codex-smoke
git init
"Run the command: git --version. Reply with its exact output only." | codex exec --sandbox read-only --ephemeral --ignore-user-config --ignore-rules -c windows.sandbox=elevated -c approval_policy="never" -
```

- `blocked by policy` と表示される場合は、Codexのread-only sandboxがコマンドを実行できていない。`-c windows.sandbox=elevated` を付けているか確認し、このユーザーの対話セッションでWindows sandboxの初期設定を完了させる（未検証: Runner専用ユーザーで初期設定が求められるかどうかと、その手順）

- `git version ...` が返ればよい
- 文章だけが返り、出力に `host executable was not found` や `Code Mode is unavailable` が含まれる場合は、Codexがコマンドを実行できていない（Remote Reviewでは `ENVIRONMENT_FAILURE` / INCOMPLETE になる）
- Runnerには npm版 `@openai/codex` を使う。デスクトップアプリ同梱のCLIは、補助実行ファイル（tool host）が同じ場所に無いとコマンドを実行できないことを2026-09-13に確認した
- Windows sandboxの初期設定を求められた場合は、このユーザーのセッションで完了させる

- Interactive mode（本Pilotの採用mode）では、runner processはこのsmoke testと同じ「`<runner-user>` の対話セッション」で動く。Service modeを採用する場合にかぎり、非対話service sessionから起動したときにCodexのread-only sandboxが追加設定なしで動くかは 未検証 であり、§6の実行結果で確認する
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

   | 質問 | Interactive mode（本Pilotの採用mode） | Service modeを選ぶ場合 |
   |---|---|---|
   | runner group | Enter（Default） | 同左 |
   | runner name | 任意（個人名・PC名を含めない名前） | 同左 |
   | additional labels | `otomo-review` | 同左 |
   | work folder | Enter（`_work`） | 同左 |
   | `Would you like to run the runner as service? (Y/N)` | **`N`** | `Y` |
   | service account | （訊かれない） | `.\<runner-user>` と、そのpassword |

   - tokenは短時間で失効する一時tokenだが、保存しない
   - `self-hosted` / `Windows` / `X64` のlabelは自動で付く（labelの大文字小文字は区別されない）
   - service化は明示的なopt-inである。非対話で登録する場合、Service modeは `--runasservice` を付けたときだけ有効になり、付けなければInteractive modeで登録される（`config.cmd --help`、runner 2.337.0で確認）
   - Interactive modeでは、runner登録自体は管理者PowerShellで行ってよいが、**runnerの起動は `<runner-user>` のセッションで行う**（§5.1）

5. Runner Modeを記録する

   採用したRunner Modeを、Runner Acceptanceの記録に明示する（`harness/REMOTE_REVIEW.md` §20 共通invariant）。Acceptanceを実施したmodeと、実運用で使うmodeを一致させる。

### 5.1 Interactive mode: 起動

`<runner-user>` でWindowsにサインインし、runner directoryで実行する:

```powershell
cd C:\actions-runner\<repository>
.\run.cmd
```

`config.cmd --help` の記載どおり、`run.cmd` はrunnerを対話的に実行し、オプションを必要としない（runner 2.337.0で確認）。

期待する状態:

- GitHubへ接続され、Listening for Jobs の状態になる
- GitHub の Runners 画面で **Idle** になる
- そのterminalは起動したまま維持される（閉じるとrunnerも停止する）

### 5.2 Interactive mode: 停止

runnerを起動したterminalで `Ctrl+C` を送り、runner processを終了させる。job実行中に停止した場合、そのrunは失敗またはcancelとして残りうるため、Idle（job非実行）の状態で停止する。

期待する状態:

- GitHub の Runners 画面で **Offline** になる
- 新しいRemote Review jobが開始されない（`/review` してもjobはqueuedのまま残る）

### 5.3 Interactive mode: 再起動・logoff

Interactive runnerは、PCの再起動や `<runner-user>` のlogoffのあと **自動では復帰しない**。人間が再度サインインして `run.cmd` を起動する必要がある。

これは意図した設計であり、露出時間を人間が制御するというRunner Mode選択の根拠そのものである（`harness/REMOTE_REVIEW_SECURITY.md` §3 Runner Mode）。Remote Reviewを使うときだけrunnerを起動する。

### 5.4 Mode変更

既に一方のmodeで登録済みのrunnerを他方へ変更する場合:

1. 現在のrunner processを停止する（Interactive: `run.cmd` を停止 / Service: serviceを停止）
2. Runners画面 → 対象Runner → Remove で表示される `./config.cmd remove --token ...` を実行する
3. §5の手順で、目的のmodeとして登録し直す
4. `harness/REMOTE_REVIEW.md` §20のmode固有checkを **変更後のmodeで再実施する**（旧modeのAcceptance結果を流用しない）

Node.js・Git・Codexを後からinstall / 更新した場合は、runner processを再起動する。

- Interactive mode: `run.cmd` を停止して起動し直す
- Service mode: `Restart-Service "actions.runner.*"`

## 6. 動作確認（Acceptance Tests）

`harness/REMOTE_REVIEW.md` §15 のうち、GitHub上でしか確認できない項目を実施する。AT-01〜AT-29はRunner Mode非依存である（`harness/REMOTE_REVIEW.md` §15）。

§20 Runner Acceptance Gate（共通invariant + 採用したmodeのmode固有check）が成立することを、以下のAT-02実行時に確認する。Interactive modeでは、共通invariant（専用runner user context・Codex起動・read-onlyコマンド実行・`TOOL_CHECK` VERIFIED・Evidence生成・採用modeの記録）に加えて次を記録する:

- runner processを実行しているWindowsユーザー（`<runner-user>` であること）
- Runner Mode = Interactive
- `run.cmd` 稼働中にGitHub上でOnline / Idleであること
- `run.cmd` 起動後、対話promptでreview jobが停止しないこと（review job単位でunattendedに完走する）
- runner processを停止するとGitHub上でOfflineになり、新しいjobが開始されないこと

Service modeを検証していない場合、Service modeの項目をPASSと記録しない（`harness/REMOTE_REVIEW.md` §20）。

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
8. **AT-18**: `/review` を続けて2回投稿 → 古いrunが cancelled になり、新しいrunだけが結果をコメントする（allowlist外のユーザーの `/review` らしきコメントでも同じcancelが起きうる。`harness/REMOTE_REVIEW.md` §13 Residual Risk）
9. **AT-16 / AT-19 / AT-20**: PRと `main` にcommitが増えていない、PRがmergeされていないことを確認する
10. **AT-17**: Artifactの中身を `ghp_`、`ghs_`、`github_pat_`、`sk-`、`Bearer`、`password` で検索し、秘密値が無いことを確認する
11. 結果をProduct側に記録する（記録するかどうか・どこに残すかは人間判断）

## 7. Troubleshooting

| 症状 | 原因候補 | 対処 |
|---|---|---|
| `/review` してもrunが出ない | workflowがmainに無い | §1 |
| runはあるが `Authorize /review` が skipped | `REMOTE_REVIEW_ALLOWED_USERS` 未設定・loginが含まれない | §2 |
| `Authorize request and resolve PR` step に `::error::` ログが出るが、run自体は失敗せず、PRには何も投稿されない | `REMOTE_REVIEW_ALLOWED_USERS` のJSONが不正（`authorize.mjs` が `ALLOWLIST_INVALID` としてfail closedし、Self-hosted review jobへは到達しない） | §2の値を修正する |
| `Verify and review (self-hosted)` が Queued のまま | Runner offline、label不一致、PCスリープ。Interactive modeでは `run.cmd` が起動されていない・再起動/logoffで停止した（§5.3） | `<runner-user>` でサインインして `run.cmd` を起動する、Runners画面で Idle を確認、`REMOTE_REVIEW_RUNS_ON` を確認 |
| INCOMPLETE + `AUTHENTICATION_FAILURE` | Codex loginの期限切れ | `<runner-user>` で `codex login` |
| INCOMPLETE + `Codex CLI was not found` | runner processから `codex` が見えない | `REMOTE_REVIEW_CODEX_BIN` を設定、runner processを再起動（§5.4） |
| Install が INFRA_ERROR（EPERM / EBUSY） | antivirusのスキャン、前回processの残留 | 再度 `/review`、runner processを再起動（§5.4） |
| Install が INFRA_ERROR（network） | npm registryに届かない | 再度 `/review` |
| Format check が多数のfileでFAILし、Infrastructureに `CRLF` の記録がある | checkoutで改行がCRLFへ変換された | workflowの `GIT_CONFIG_*`（`core.autocrlf=false`）が残っているか確認する |
| path too long | Windows MAX_PATH | Runner directoryを短くする、`core.longpaths` |
| INCOMPLETE + `TOOL_CHECK` の記録（UNAVAILABLE / MISSING） | Codexがread-onlyコマンドを実行できなかった（例: Windows sandbox未設定で `blocked by policy`） | `codex.log` を確認し、§4 のsmoke testをRunnerユーザーでやり直す |
| INCOMPLETE + `Codex could not start its tool host` | Codex CLIの補助実行ファイル（tool host）が無い | npm版 `@openai/codex` を使う、`REMOTE_REVIEW_CODEX_BIN` を確認、§4 のsmoke testをやり直す |
| Codexが境界違反（checkout changed） | Codex CLIの挙動変化 | Runnerを停止し、Codex CLIのversionと `codex.log` を確認 |

## 8. 停止・削除

- 一時停止:
  - Interactive mode: 稼働中の `run.cmd` を停止する（§5.2）
  - Service mode: `Stop-Service "actions.runner.*"`
  - どちらの場合も、GitHub の Runners 画面で **Offline** を確認する
- 削除: Runners画面 → 対象Runner → Remove で表示される `./config.cmd remove --token ...` を実行し、`C:\actions-runner\<repository>` を削除する
- 侵害が疑われる場合: runner process停止（上記） → Runner削除 → ChatGPT側で該当ログインのsessionを確認 → `<runner-user>` ユーザーをprofileごと削除して作り直す

## 9. Cost note

- `authorize` / `report` jobは GitHub-hosted runner（ubuntu）で動くため、private repositoryではGitHub Actionsの利用時間を消費する（1回あたり数分未満の見込み（推測））
- self-hosted runnerの利用に対するGitHub側の課金有無は 未確認。導入前にGitHubの最新のBilling documentationを確認する

## Change History

- 2026-09-13 v0.1 Draft: OTOMO LAB pilotとして作成
- 2026-09-13 v0.1 Draft, Independent Review remediation: smoke testをstdin経由・`--ignore-rules`・`approval_policy=never` 付きに更新し本番呼び出しへ近づけた。Runner Acceptance Gate（§20）への参照とAT-18のcancellation residual riskの注記を追加
- 2026-09-14 文書ドリフト修正（Codex再レビュー前）: `REMOTE_REVIEW_ALLOWED_USERS` のJSONが不正な場合の説明を現在の実装（`authorize.mjs` がALLOWLIST_INVALIDとしてfail closedし、workflow run自体はエラーにならない）へ修正。§2とTroubleshootingを整合させた
- 2026-09-23 Runner Mode採用（Human Decision、docs-only、Forced Level 2扱い）: 登録手順がService modeを事実上強制していた（`run as service | Y`）ため、Interactive / Service両modeを記述する形へ変更した。§0へRunner Mode行（Interactive）を追加。§4の「Windows service（非対話セッション）でCodex sandboxが動くか未検証」という注記を、Interactive modeではsmoke testと同じ対話セッションで動くこと・未検証はService modeに限る旨へ整理。§5の登録表をmode別に分け、実際のrunner CLIの対話prompt（`Would you like to run the runner as service? (Y/N)`）と、非対話登録では `--runasservice` を付けたときだけService modeになることを、runner 2.337.0の `config.cmd --help` と実バイナリで確認した上で記載（オプションを創作していない）。§5.1〜§5.4として、Interactive modeの起動（`run.cmd`）・停止・再起動/logoff時に自動復帰しないこと・Mode変更手順（再登録とmode固有checkの再実施）を追加。§6のAcceptance手順へ、AT-01〜AT-29がRunner Mode非依存であることと、Interactive modeで記録すべきRunner Mode Evidenceを追加。§7 Troubleshootingと§8 停止・削除のservice前提の記述をmode別へ変更。**専用 `<runner-user>` 運用、Administrators非追加、Codex以外の認証情報を置かない要求は変更していない**
