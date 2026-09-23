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
   - Interactive modeでは、runner登録自体は管理者PowerShellで行ってよいが、**runnerの起動は `<runner-user>` のセッションで行う**（§5.2）

5. Runner Modeを記録する

   採用したRunner Modeを、Runner Acceptanceの記録に明示する（`harness/REMOTE_REVIEW.md` §20 共通invariant）。Acceptanceを実施したmodeと、実運用で使うmodeを一致させる。

### 5.1 Runner directoryのACL hardening（必須。起動前に行う）

**Windowsの既定NTFS権限のままにしない。** `C:\` 直下に作ったdirectoryは、既定で `NT AUTHORITY\Authenticated Users` に `Modify` が継承される。この状態では、**普段使いを含む同一PC上の通常アカウントが、runner binary・script・`_work`・`_diag` を、専用runner userの権限で実行される前に差し替えられる**。runner登録の認証情報（`.credentials` / `.credentials_rsaparams`）も通常アカウントから読める。専用userが非Administratorであることだけでは、この経路は塞げない（`harness/REMOTE_REVIEW_SECURITY.md` §3）。

Service modeで `--runasservice` 登録した場合、runnerは machine-local group（`GITHUB_ActionsRunner_*`）を作り、runner directoryのrootへ **explicit** な FullControl ACEを付ける（Pilot runnerで確認。継承ACEではないため `/inheritance:r` では消えない）。**本手順はこのgroupを許可principalに含めない。** modeによらずrunner user自身へ直接付与し、groupのACEはhardeningで除去する。groupが対象installationのものであることと、そのmembershipの正当性を、runnerの内部実装に依存せず汎用に検証する手順を定義できないため、許可集合を狭める方を選ぶ。Interactive modeではこのgroupはそもそも作られない。

#### 必須invariant（tree全体。rootだけではない）

runner directoryのrootと、その下のすべてのfile / directory（hidden・system属性を含む）について、次をすべて満たす。

1. **許可principal**: ACEを持ってよいのは `NT AUTHORITY\SYSTEM`（`S-1-5-18`）/ `BUILTIN\Administrators`（`S-1-5-32-544`）/ 専用runner user の3つだけ。readだけのACEも含め、それ以外（`Authenticated Users`・`Users`・普段使いアカウント・`GITHUB_ActionsRunner_*` group・`CREATOR OWNER` 等）を1つも残さない
2. **必須right**: 3 principalそれぞれが、rootに explicit な Allow `(OI)(CI)` FullControlを持ち、すべてのdescendantがそのFullControlを継承している
3. **Deny無し**: Deny ACEを置かない（意図したtreeにDenyは存在しない）
4. **root**: 継承が無効（protected）で、継承ACEを持たない
5. **descendant**: protected（継承を切った）objectが無い。すべてrootのpolicyを継承する
6. **owner**: すべてのobjectのownerが上記3 principalのいずれか（ownerはACEに関係なく自分のobjectのDACLを書き換えられるため）
7. **reparse point無し**: junction / symbolic linkを置かない（検証が辿らない経路を作らない）
8. **ACL変更commandがすべて成功している**（exit code 0）

principalは表示名ではなくSIDで判定する（localeの差と同名アカウントの取り違えを避ける）。

#### `icacls` のsemantics（Microsoft `icacls` documentationと実機で確認）

| option | 動作 | 本手順での扱い |
|---|---|---|
| `/inheritance:r` | **継承ACEだけ**を除去し、継承を無効化する。explicit ACEは残る | rootにだけ使う。単独では既存のexplicit ACE（例: runner groupのACE）を消せない |
| `/grant` | 既存のexplicit grantに**追加**する | 使わない |
| `/grant:r` | 指定principalの既存explicit grantを**置換**する（他principalのACEは残る） | rootの3 principalに使う |
| `/reset` | ACLを既定の継承ACLへ置き換える（explicit ACEを捨て、保護を解除する） | `/T` を付けてtree全体を初期化する |
| `/setowner` | ownerを変更する | `/T` を付けてownerを統一する |
| `/T` | 指定directory配下のすべてのfile / directoryへ適用する | `/reset` / `/setowner` に付ける |

旧手順（`/inheritance:r` + `/grant`）は、rootに既に在るexplicit ACEと、descendantのexplicit ACE・保護を残す。下記negative testで、旧手順の後もそれらが残ることを確認した。

#### 手順（管理者PowerShell、runner停止中）

次を `Harden-RunnerAcl.ps1` として、runner directoryの外の、Administratorだけが書ける場所へ保存し、`powershell -NoProfile -ExecutionPolicy Bypass -File Harden-RunnerAcl.ps1 -RunnerDir 'C:\actions-runner\<repository>' -RunnerUser '<runner-user>'` で実行する。

```powershell
param(
    [Parameter(Mandatory)] [string] $RunnerDir,
    [Parameter(Mandatory)] [string] $RunnerUser,
    [string] $OwnerSid = 'S-1-5-32-544'   # BUILTIN\Administrators
)
$ErrorActionPreference = 'Stop'

function Invoke-Icacls {
    & icacls.exe @args
    if ($LASTEXITCODE -ne 0) { throw "icacls failed (exit $LASTEXITCODE): icacls $($args -join ' ')" }
}

$RunnerDir = (Resolve-Path -LiteralPath $RunnerDir).ProviderPath
$RunnerSid = (New-Object System.Security.Principal.NTAccount($env:COMPUTERNAME, $RunnerUser)).Translate(
    [System.Security.Principal.SecurityIdentifier]).Value

# 1. Whole tree: drop explicit ACEs and protection; inherit from parent only
Invoke-Icacls $RunnerDir /reset /T /Q
# 2. Whole tree: normalize owner (an owner can always rewrite its object's DACL)
Invoke-Icacls $RunnerDir /setowner "*$OwnerSid" /T /Q
# 3. Root: set the 3 required explicit ACEs with replace semantics (/grant:r)
Invoke-Icacls $RunnerDir /grant:r '*S-1-5-18:(OI)(CI)F' '*S-1-5-32-544:(OI)(CI)F' "*${RunnerSid}:(OI)(CI)F"
# 4. Root: disable inheritance and drop inherited ACEs (descendants inherit only the 3)
Invoke-Icacls $RunnerDir /inheritance:r
'HARDENING: DONE'
```

- いずれかの `icacls` が0以外のexit codeを返すと、scriptは例外で止まり `HARDENING: DONE` を出さない。途中で止まったtreeは安全とみなさず、原因を解消して**最初からやり直す**。Denyやownerのせいで走査できない場合は、`takeown /F <runner-dir> /R /A /D Y`（これもexit codeが0であることを確認する）でAdministratorsへ所有権を移してからやり直す
- `-RunnerUser` のアカウントが存在しない場合、SID解決の時点で例外になる（付与先を取り違えない）
- 手順1〜4の間は、一時的にrootが親directoryのACLを継承する。この間に通常アカウントが作ったobjectはownerがそのアカウントになるため、下記検証のowner / principal判定でFAILする
- runner userへ `F` を与えるのは、runnerが `_work` / `_diag` への書き込み、自己更新、runner-owned stateの更新を行うためである。ここを読み取り専用にするとrunnerが動かない
- `.ps1` は **UTF-8 with BOM** で保存するか、commentをASCIIに限る。Windows PowerShell 5.1はBOM無しscriptをANSI codepageで読むため、日本語commentが直後の行を巻き込み、**commandが黙って実行されない**ことを実機で確認した（2026-09-23）

#### 検証（acceptance条件。tree全体を走査する）

次を `Test-RunnerAcl.ps1` として同様に保存し、`-RunnerDir` / `-RunnerUser` を付けて実行する。

```powershell
param(
    [Parameter(Mandatory)] [string] $RunnerDir,
    [Parameter(Mandatory)] [string] $RunnerUser
)
$ErrorActionPreference = 'Stop'
$SidType = [System.Security.Principal.SecurityIdentifier]
$fails = New-Object System.Collections.Generic.List[string]
function Fail([string] $path, [string] $why) { $fails.Add("${path}: $why") }

$RunnerDir = (Resolve-Path -LiteralPath $RunnerDir).ProviderPath
$RunnerSid = (New-Object System.Security.Principal.NTAccount($env:COMPUTERNAME, $RunnerUser)).Translate($SidType).Value
$required = [ordered]@{ 'S-1-5-18' = 'SYSTEM'; 'S-1-5-32-544' = 'Administrators'; $RunnerSid = 'runner user' }
$FullControl = 0x1F01FF; $GenericAll = 0x10000000
$CiOi = [System.Security.AccessControl.InheritanceFlags]'ContainerInherit, ObjectInherit'

function Test-Full($rule) {
    $r = [int64]$rule.FileSystemRights
    (($r -band $FullControl) -eq $FullControl) -or (($r -band $GenericAll) -ne 0)
}

# Root + every descendant (incl. hidden/system). Reparse points are not followed; they FAIL.
$items = New-Object System.Collections.Generic.List[object]
$queue = New-Object System.Collections.Generic.Queue[object]
$queue.Enqueue((Get-Item -LiteralPath $RunnerDir -Force))
while ($queue.Count) {
    $item = $queue.Dequeue(); $items.Add($item)
    if ($item.Attributes -band [IO.FileAttributes]::ReparsePoint) { Fail $item.FullName 'reparse point'; continue }
    if ($item.PSIsContainer) {
        try { Get-ChildItem -LiteralPath $item.FullName -Force | ForEach-Object { $queue.Enqueue($_) } }
        catch { Fail $item.FullName "enumeration failed: $($_.Exception.Message)" }
    }
}

foreach ($item in $items) {
    $p = $item.FullName; $isRoot = ($p -eq $RunnerDir)
    try { $acl = Get-Acl -LiteralPath $p } catch { Fail $p 'ACL unreadable'; continue }
    $owner = $acl.GetOwner($SidType).Value
    if (-not $required.Contains($owner)) { Fail $p "owner not allowed ($($acl.Owner))" }
    $rules = @($acl.GetAccessRules($true, $true, $SidType))
    foreach ($r in $rules) {
        $sid = $r.IdentityReference.Value
        if ($r.AccessControlType -ne 'Allow') { Fail $p "Deny ACE ($sid $($r.FileSystemRights))" }
        if (-not $required.Contains($sid)) { Fail $p "unexpected principal ($sid $($r.FileSystemRights) inherited=$($r.IsInherited))" }
    }
    if ($isRoot) {
        if (-not $acl.AreAccessRulesProtected) { Fail $p 'root inheritance not disabled' }
        foreach ($sid in $required.Keys) {
            $ok = $rules | Where-Object { $_.IdentityReference.Value -eq $sid -and $_.AccessControlType -eq 'Allow' -and
                -not $_.IsInherited -and (Test-Full $_) -and $_.InheritanceFlags -eq $CiOi -and $_.PropagationFlags -eq 'None' }
            if (-not $ok) { Fail $p "missing required (OI)(CI)FullControl for $($required[$sid])" }
        }
    } else {
        if ($acl.AreAccessRulesProtected) { Fail $p 'protected descendant (does not inherit root policy)' }
        foreach ($sid in $required.Keys) {
            $ok = $rules | Where-Object { $_.IdentityReference.Value -eq $sid -and $_.AccessControlType -eq 'Allow' -and
                $_.IsInherited -and (Test-Full $_) }
            if (-not $ok) { Fail $p "missing inherited FullControl for $($required[$sid])" }
        }
    }
}

"Items checked: $($items.Count)"
if ($fails.Count) { $fails | ForEach-Object { "  $_" }; 'ACL CHECK: FAIL'; exit 1 }
'ACL CHECK: PASS'; exit 0
```

- 最後の行が `ACL CHECK: PASS`（exit code 0）でなければ、**Runner Acceptanceを成立させない**（`harness/REMOTE_REVIEW.md` §20）。FAILの場合は理由の行が出る（例: `unexpected principal` / `missing required` / `missing inherited` / `Deny ACE` / `protected descendant` / `owner not allowed` / `reparse point` / `enumeration failed` / `ACL unreadable`）。hardeningをやり直し、再度検証する
- 実施するタイミング: (1) hardening直後、`run.cmd`（またはservice）を初めて起動する前、(2) AT-02のjob完了後、§6.1 Step 1と同じ機会。**両方がPASS**であること
- 読み取るのはACLだけであり、treeを変更しない

#### Negative test（2026-09-23、Windows 10 Pro / PowerShell 5.1、使い捨てdirectory）

runner directoryと同じ構成（`bin`・`externals`・`_work`・`_diag`・hidden `.runner` / `.credentials`）の使い捨てdirectoryに対して、上記2 scriptをそのまま実行した。本番runner directoryのACLは変更していない（本番は読み取りだけの検証を行い、FAILになることを確認した）。

| Case | 結果 |
|---|---|
| hardening後のtree / 旧手順のtreeをhardening / 全mutationを混ぜたtreeを再hardening | PASS |
| hardening無し（既定継承） | FAIL |
| runner user欠落 / SYSTEM欠落 / Administrators欠落 | FAIL |
| runner userがread-onlyのみ（right不足） | FAIL |
| rootへ通常ユーザーのexplicit Modify | FAIL |
| child binaryへexplicit write ACE / hidden `.credentials` へexplicit ACE | FAIL |
| protected child directory | FAIL |
| 必須principalへのDeny ACE | FAIL |
| 許可外のowner | FAIL |
| 旧手順（`/inheritance:r` + `/grant`）の後に残ったstale explicit ACE | FAIL |
| 非elevatedで `/setowner` を実行（icacls exit 1307） / 存在しないrunner user | scriptが例外で停止 |

- hardening後にrunner user（の代理）が `_work` / `_diag` へ新しいfile・directoryを作っても、継承によりPASSのままであることを確認した
- 検証環境は非elevatedであったため、`-OwnerSid` にはAdministratorsではなく実行ユーザー自身を与え、runner userの代理も実行ユーザー自身とした。Administratorsへの `/setowner` は、非elevatedで失敗しscriptが停止することだけを確認した（elevated環境での成功経路は本番Acceptance時に確認する）

#### 記録するもの

Runner Acceptance記録（`harness/REMOTE_REVIEW.md` §20 Durable Acceptance Record）へ次を残す。

- runner directory path
- 専用runner user名
- `HARDENING: DONE` が出たこと
- 検証 (1)・(2) の `Items checked:` の件数と最終行（`ACL CHECK: PASS` / `FAIL`）。FAILの理由行を残す場合は、`S-1-5-21-` で始まるlocal account / groupのSIDを一般名へ置き換える

password・local accountのSID値・token・無関係な個人pathは記録しない。group名・アカウント名は一般名で足りる。

### 5.2 Interactive mode: 起動

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

### 5.3 Interactive mode: 停止

runnerを起動したterminalで `Ctrl+C` を送り、runner processを終了させる。job実行中に停止した場合、そのrunは失敗またはcancelとして残りうるため、Idle（job非実行）の状態で停止する。

期待する状態:

- GitHub の Runners 画面で **Offline** になる
- 新しいRemote Review jobが開始されない（`/review` してもjobはqueuedのまま残る）

### 5.4 Interactive mode: 再起動・logoff

Interactive runnerは、PCの再起動や `<runner-user>` のlogoffのあと **自動では復帰しない**。人間が再度サインインして `run.cmd` を起動する必要がある。

これは意図した設計であり、露出時間を人間が制御するというRunner Mode選択の根拠そのものである（`harness/REMOTE_REVIEW_SECURITY.md` §3 Runner Mode）。Remote Reviewを使うときだけrunnerを起動する。

### 5.5 Mode変更

既に一方のmodeで登録済みのrunnerを他方へ変更する場合:

1. 現在のrunner processを停止する（Interactive: `run.cmd` を停止 / Service: serviceを停止）
2. Runners画面 → 対象Runner → Remove で表示される `./config.cmd remove --token ...` を実行する
3. §5の手順で、目的のmodeとして登録し直し、§5.1のhardeningと検証をやり直す（Service modeの登録はrootへrunner groupのexplicit ACEを付けるため、以前のhardening結果を流用しない）
4. `harness/REMOTE_REVIEW.md` §20のmode固有checkを **変更後のmodeで再実施する**（旧modeのAcceptance結果を流用しない）

Node.js・Git・Codexを後からinstall / 更新した場合は、runner processを再起動する。

- Interactive mode: `run.cmd` を停止して起動し直す
- Service mode: `Restart-Service "actions.runner.*"`

## 6. 動作確認（Acceptance Tests）

`harness/REMOTE_REVIEW.md` §15 のうち、GitHub上でしか確認できない項目を実施する。AT-01〜AT-29はRunner Mode非依存である（`harness/REMOTE_REVIEW.md` §15）。

§20 Runner Acceptance Gate（共通invariant + 採用したmodeのmode固有check）が成立することを、以下のAT-02実行時に確認する。Interactive modeでは、共通invariant（専用runner user context・ACL検証・runner identity / run binding・Codex起動・read-onlyコマンド実行・`TOOL_CHECK` VERIFIED・Evidence生成・採用modeの記録）に加えて次を記録する:

- Runner Mode = Interactive
- §5.1 のACL検証（tree全体）が、hardening直後とAT-02完了後の両方で `ACL CHECK: PASS` であること
- **target installationのListener processが1つに特定され、その実ownerが `<runner-user>` であり、そのprocessがAT-02のGitHub Actions jobを実行したrunner登録・同じ稼働期間に結び付いていること（下記 §6.1。サインインしているshell userの自己申告では足りない）**
- `run.cmd` 稼働中にGitHub上でOnline / Idleであること
- `run.cmd` 起動後、対話promptでreview jobが停止しないこと（review job単位でunattendedに完走する）
- runner processを停止するとGitHub上でOfflineになり、新しいjobが開始されないこと

Service modeを検証していない場合、Service modeの項目をPASSと記録しない（`harness/REMOTE_REVIEW.md` §20）。

### 6.1 Runner Identity and Run Binding（必須。§20 共通invariant）

**`whoami` だけでprincipalを証明しない。** `whoami` はそのshellの実行者を示すだけで、`Runner.Listener` が同じprincipalで動いていることを示さない。

**process名だけでrunnerを特定しない。** `Name = 'Runner.Listener.exe'` は同一PC上の別installationのListenerにも一致する。PIDとownerだけでは、どのinstallation・どのrunner登録・どのGitHub runに対応するかを示さない。PIDは再利用されうる。runner nameは一意である保証が無い。

次の連鎖を、手作業の対応付けを挟まずに証明する。

```text
target runner directory
  → そのinstallationの bin\Runner.Listener.exe で動く唯一のprocess（PID + 作成時刻）
  → そのprocessの実owner（SID）= 専用runner user
  → 同じdirectoryの .runner（agentId / agentName）
  → GitHub: 受け入れ対象run attemptのself-hosted jobを実行したrunner_id = agentId
  → job環境: metadata.json の run.id / run.attempt / run.runner_name
  → 同じinstallationの _diag に、同じrun_idを持つWorker logがある
  → Durable Acceptance Record
```

#### 前提

- Step 1は **elevated（管理者）PowerShell** で実行する。非elevatedでは他ユーザーの `Runner.Listener` の `ExecutablePath` とownerが読めない（実機で `ExecutablePath` が空、`GetOwner` が ReturnValue 2）。その状態では曖昧さを解消できないため、scriptはFAILにする。Interactive modeでは、runner userのsessionで `run.cmd` を動かしたまま、別のAdministratorでelevated PowerShellを開く（runner userはlogoffしない）
- runnerは、AT-02の `/review` を投稿する **60秒以上前** に起動しておく
- Step 1は、AT-02のjobが完了してから **60秒以上後** に、runnerを停止・再起動する前に実行する。runnerの自己更新や再起動が挟まるとprocessが変わるためFAILになる。その場合はAT-02からやり直す
- Windowsの時刻同期が有効であること（`w32tm /query /source` が成功し、`Local CMOS Clock` / `Free-running System Clock` でない）
- `.credentials` / `.credentials_rsaparams` は読まない・記録しない。本手順が読むのは `.runner` と、`_diag` の `Worker_*.log` の中の `run_id` の値だけである

#### Step 1: local evidence（runner PC、elevated）

次を `Get-RunnerListenerEvidence.ps1` として §5.1のscriptと同様に保存し、実行する。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File Get-RunnerListenerEvidence.ps1 -RunnerDir 'C:\actions-runner\<repository>' -RunnerUser '<runner-user>' -RunId <run-id> -OutFile listener.json
```

```powershell
param(
    [Parameter(Mandatory)] [string] $RunnerDir,
    [Parameter(Mandatory)] [string] $RunnerUser,
    [Parameter(Mandatory)] [string] $RunId,
    [Parameter(Mandatory)] [string] $OutFile
)
$ErrorActionPreference = 'Stop'
$fails = New-Object System.Collections.Generic.List[string]
$rec = [ordered]@{}

# Must run elevated: otherwise other users' Runner.Listener processes hide ExecutablePath and owner.
$id = [System.Security.Principal.WindowsIdentity]::GetCurrent()
if (-not ([System.Security.Principal.WindowsPrincipal]$id).IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)) {
    $fails.Add('not running elevated (Administrator)')
}

$RunnerDir = (Resolve-Path -LiteralPath $RunnerDir).ProviderPath
$expected  = [System.IO.Path]::GetFullPath((Join-Path $RunnerDir 'bin\Runner.Listener.exe'))
$RunnerSid = (New-Object System.Security.Principal.NTAccount($env:COMPUTERNAME, $RunnerUser)).Translate(
    [System.Security.Principal.SecurityIdentifier]).Value
$rec.RunnerDir = $RunnerDir

# 1. Listener process: exactly one Runner.Listener.exe whose ExecutablePath is this installation's bin.
$all = @(Get-CimInstance Win32_Process -Filter "Name = 'Runner.Listener.exe'")
$rec.ListenerProcessesOnHost = $all.Count
foreach ($p in $all) { if (-not $p.ExecutablePath) { $fails.Add("PID $($p.ProcessId): ExecutablePath unreadable (cannot disambiguate)") } }
$m = @($all | Where-Object { $_.ExecutablePath -and
    [string]::Equals([System.IO.Path]::GetFullPath($_.ExecutablePath), $expected, [System.StringComparison]::OrdinalIgnoreCase) })
$rec.TargetListenerMatches = $m.Count
if ($m.Count -ne 1) {
    $fails.Add("target Listener matches: $($m.Count) (exactly 1 required)")
} else {
    $p = $m[0]
    $rec.ListenerPath       = $p.ExecutablePath
    $rec.ListenerPid        = $p.ProcessId
    $rec.ListenerCreatedUtc = $p.CreationDate.ToUniversalTime().ToString('o')
    $o = Invoke-CimMethod -InputObject $p -MethodName GetOwner
    $s = Invoke-CimMethod -InputObject $p -MethodName GetOwnerSid
    $rec.ListenerOwner = "$($o.Domain)\$($o.User)"
    if ($o.ReturnValue -ne 0 -or $s.ReturnValue -ne 0) { $fails.Add('process owner unreadable') }
    elseif ($s.Sid -ne $RunnerSid) { $fails.Add("owner mismatch ($($rec.ListenerOwner))") }
}

# 2. Local registration of the same installation (.runner only; never read .credentials*).
try {
    $reg = Get-Content -Raw -LiteralPath (Join-Path $RunnerDir '.runner') | ConvertFrom-Json
    $rec.RunnerAgentId   = [string]$reg.agentId
    $rec.RunnerAgentName = [string]$reg.agentName
    if ($rec.RunnerAgentId -notmatch '^\d+$' -or -not $rec.RunnerAgentName) { $fails.Add('.runner agentId/agentName missing') }
} catch { $fails.Add('.runner unreadable') }

# 3. Local trace of the accepted run in this installation's _diag (Worker log carrying run_id).
$hits = @()
foreach ($w in @(Get-ChildItem -LiteralPath (Join-Path $RunnerDir '_diag') -Filter 'Worker_*.log' -File)) {
    $kv = Select-String -LiteralPath $w.FullName -Pattern '^\s*"k": "run_id",?\s*$' -Context 0, 1 | Select-Object -First 1
    if ($kv -and $kv.Context.PostContext[0] -match '"v":\s*"(\d+)"' -and $Matches[1] -eq $RunId) {
        $hits += [pscustomobject]@{ Name = $w.Name; CreatedUtc = $w.CreationTimeUtc.ToString('o') }
    }
}
if ($hits.Count -ne 1) { $fails.Add("Worker log for run ${RunId}: $($hits.Count) found (exactly 1 required)") }
else { $rec.WorkerLog = $hits[0].Name; $rec.WorkerLogCreatedUtc = $hits[0].CreatedUtc }

# 4. Clock source (timestamps are compared with GitHub's).
$src = @(& w32tm.exe /query /source 2>$null); $rc = $LASTEXITCODE
$rec.ClockSource = if ($rc -eq 0 -and $src.Count -and $src[0].Trim()) { $src[0].Trim() } else { "<w32tm failed: exit $rc>" }
if ($rc -ne 0 -or $rec.ClockSource -match '^<|Local CMOS Clock|Free-running System Clock') { $fails.Add("clock source not verified ($($rec.ClockSource))") }

$rec.RunId       = $RunId
$rec.CapturedUtc = (Get-Date).ToUniversalTime().ToString('o')
$rec.Failures    = @($fails)
$rec.Result      = if ($fails.Count) { 'FAIL' } else { 'PASS' }
$rec | ConvertTo-Json | Set-Content -LiteralPath $OutFile -Encoding UTF8
$fails | ForEach-Object { "  $_" }
"LISTENER CHECK: $($rec.Result)"
if ($fails.Count) { exit 1 } else { exit 0 }
```

- 同じ機会に、§5.1の検証 (2)（`Test-RunnerAcl.ps1`）も実行する
- `LISTENER CHECK: PASS`（exit code 0）でなければ **§20 FAIL**

#### Step 2: GitHub側evidence（repositoryを読めるoperator環境）

```powershell
gh api repos/<owner>/<repository>/actions/runs/<run-id>/attempts/<attempt>/jobs > jobs.json
gh run download <run-id> -R <owner>/<repository> -n remote-review-evidence -D evidence
```

- `gh` は `<runner-user>` では実行しない（runner userにGitHub認証情報を置かない。`harness/REMOTE_REVIEW_SECURITY.md` §3）
- `jobs.json` の `run_id` / `run_attempt` / `status` / `started_at` / `completed_at` / `runner_id` / `runner_name` は、GitHub REST API（Workflow jobs）が文書化しているfieldである。`runner_id` は `.runner` の `agentId` と同じ値になる（2026-09-16のPilot runの実データで確認）
- `evidence/metadata.json` の `run.id` / `run.attempt` / `run.runner_name` は、self-hosted runner上のreview jobが、job環境の `GITHUB_RUN_ID` / `GITHUB_RUN_ATTEMPT` / `RUNNER_NAME` から記録した値である（既存のEvidence Schema 1.1。`harness/REMOTE_REVIEW.md` §10。変更しない）
- Artifactは14日で失効する（`harness/REMOTE_REVIEW.md` §10）。Acceptance記録を作るまでに取得する

#### Step 3: binding判定

次を `Test-RunnerRunBinding.ps1` として保存し、実行する。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File Test-RunnerRunBinding.ps1 -ListenerFile listener.json -JobsFile jobs.json -MetadataFile evidence\metadata.json -RunId <run-id> -RunAttempt <attempt>
```

```powershell
param(
    [Parameter(Mandatory)] [string] $ListenerFile,   # output of Get-RunnerListenerEvidence.ps1
    [Parameter(Mandatory)] [string] $JobsFile,       # gh api .../actions/runs/<id>/attempts/<n>/jobs
    [Parameter(Mandatory)] [string] $MetadataFile,   # metadata.json from the run's remote-review-evidence Artifact
    [Parameter(Mandatory)] [string] $RunId,
    [Parameter(Mandatory)] [string] $RunAttempt,
    [string] $JobName = 'Verify and review (self-hosted)',
    [int] $MarginSeconds = 60
)
$ErrorActionPreference = 'Stop'
$fails = New-Object System.Collections.Generic.List[string]
function T([string] $v) { if (-not $v) { return $null }; [DateTimeOffset]::Parse($v, [Globalization.CultureInfo]::InvariantCulture).UtcDateTime }

$l    = Get-Content -Raw -LiteralPath $ListenerFile | ConvertFrom-Json
$jobs = @((Get-Content -Raw -LiteralPath $JobsFile | ConvertFrom-Json).jobs)
$meta = Get-Content -Raw -LiteralPath $MetadataFile | ConvertFrom-Json

# Local side
if ($l.Result -ne 'PASS') { $fails.Add('listener evidence is not PASS') }
if ([string]$l.RunId -ne $RunId) { $fails.Add('listener evidence captured for a different run ID') }

# GitHub side: exactly one self-hosted review job in this run attempt, executed by this registration
$j = @($jobs | Where-Object { $_.name -eq $JobName })
if ($j.Count -ne 1) { $fails.Add("self-hosted job '$JobName': $($j.Count) found (exactly 1 required)") }
else {
    $j = $j[0]
    if ([string]$j.run_id -ne $RunId -or [string]$j.run_attempt -ne $RunAttempt) { $fails.Add('job run_id/run_attempt mismatch') }
    if ($j.status -ne 'completed') { $fails.Add("job status is '$($j.status)'") }
    if ($null -eq $j.runner_id -or [string]$j.runner_id -ne [string]$l.RunnerAgentId) { $fails.Add("job runner_id ($($j.runner_id)) != .runner agentId ($($l.RunnerAgentId))") }
    if ([string]$j.runner_name -ne [string]$l.RunnerAgentName) { $fails.Add('job runner_name != .runner agentName') }
}

# Job-environment side (metadata.json written inside the job from GITHUB_RUN_ID / GITHUB_RUN_ATTEMPT / RUNNER_NAME)
if ([string]$meta.run.id -ne $RunId)                 { $fails.Add('metadata run.id mismatch') }
if ([string]$meta.run.attempt -ne $RunAttempt)       { $fails.Add('metadata run.attempt mismatch') }
if ([string]$meta.run.runner_name -ne [string]$l.RunnerAgentName) { $fails.Add('metadata run.runner_name != .runner agentName') }

# Timing: the same Listener process (PID + creation time) existed before the job started and after it completed
$c = T $l.ListenerCreatedUtc; $cap = T $l.CapturedUtc; $w = T $l.WorkerLogCreatedUtc
$s = if ($j -and $j -isnot [array]) { T $j.started_at } else { $null }
$e = if ($j -and $j -isnot [array]) { T $j.completed_at } else { $null }
$mg = [TimeSpan]::FromSeconds($MarginSeconds)
if (-not ($c -and $cap -and $w -and $s -and $e)) { $fails.Add('timing evidence incomplete') }
else {
    if (-not ($c -le ($s - $mg)))  { $fails.Add("listener not started >= ${MarginSeconds}s before job start") }
    if (-not ($cap -ge ($e + $mg))) { $fails.Add("listener not observed >= ${MarginSeconds}s after job completion") }
    if (-not ($w -ge $c -and $w -ge ($s - $mg) -and $w -le $e)) { $fails.Add('Worker log not created within this listener uptime and job window') }
}

$fails | ForEach-Object { "  $_" }
if ($fails.Count) { 'BINDING CHECK: FAIL'; exit 1 }
'BINDING CHECK: PASS'; exit 0
```

`-JobName` の既定値はOTOMO LABのworkflowのself-hosted job名である。workflowのjob名が異なるProductでは、そのjob名を渡す。

#### 判定規則（1つでも満たさなければ §20 FAIL）

| 対象 | FAIL条件 |
|---|---|
| 実行環境 | elevatedでない / 他のListenerの `ExecutablePath` が読めない / 時刻同期を確認できない |
| Listener | target installationの `bin\Runner.Listener.exe` と一致するprocessが0件、または2件以上（最初の1件を選ばない） |
| owner | ownerを取得できない / owner SIDが専用runner userと一致しない |
| local registration | `.runner` が読めない / `agentId` か `agentName` が無い |
| GitHub job | 対象run attemptのself-hosted jobが1件でない / `completed` でない / `run_id` か `run_attempt` が不一致 / `runner_id` が `agentId` と不一致（nullを含む） / `runner_name` が `agentName` と不一致 |
| job環境 | `metadata.json` の `run.id` か `run.attempt` が不一致 / `run.runner_name` が `agentName` と不一致、または欠落 |
| local trace | 同じinstallationの `_diag` に、`run_id` が対象run IDのWorker logが1件でない |
| timing | 下記Timing ruleを満たさない / 判定に必要な時刻が欠落している |
| 証跡の対応 | `listener.json` が別のrun ID向けに取得されている / Step 1がPASSでない |

#### Timing rule

- process identityは **PID + 作成時刻（`CreationDate`）** で扱う。終了したprocessが同じPIDと同じ作成時刻で復活することは無いため、PID単独では同一性を扱わない
- Listenerの作成時刻 ≤ jobの `started_at` − 60秒
- Step 1の取得時刻 ≥ jobの `completed_at` + 60秒。この時点で同じPID + 作成時刻のprocessが存在していれば、そのprocessはjobの開始前から完了後まで存続していた（同じrunner稼働期間）
- Worker logの作成時刻が、Listenerの作成時刻以後、かつ `started_at` − 60秒 〜 `completed_at` の範囲にある
- 60秒は、runner PCとGitHubの時計のずれに対する安全側の余裕である。境界に近い場合はFAILになる（runnerを早めに起動し、job完了後に余裕をもって取得すればよい）
- 以前のrunner稼働期間に取得した証跡は、上記を満たさないため使えない

#### 検証済み事項（2026-09-23、runner 2.337.0 / Windows 10 Pro / PowerShell 5.1）

- `.runner` に `agentId` / `agentName` がある。2026-09-16のPilot runについて、jobs APIの `runner_id` / `runner_name`、`metadata.json` の `run.id` / `run.attempt` / `run.runner_name`、`_diag` のWorker logの `run_id` が互いに一致することを実データで確認した（Step 3 scriptがPASS。Listenerのprocess fieldは、当時のprocessが既に存在しないため合成値）
- Step 3 scriptは、agentId不一致・agentName不一致・別run ID・別attempt・metadataのrunner_name欠落・`runner_id` null・job未完了・self-hosted job 0件 / 2件・Worker log欠落 / 範囲外・Listener作成がjob開始より後 / 60秒未満前・取得がjob完了前、のすべてでFAILした
- Step 1 scriptは、同じ名前のListenerを3つ（target・別installation・targetとprefixが同じsibling directory）動かした状態でtargetのPIDだけを選んだ。0件・同一installationから2件・owner不一致・`.runner` の `agentId` 欠落・Worker log無しでFAILした。非elevatedでは、elevatedでないこと・他ユーザーのListenerのpathが読めないこと・`w32tm` が失敗することでFAILした（検証環境は非elevatedのため、Step 1のPASS経路は本番Acceptance時にelevated環境で確認する）
- Worker logの `run_id` の形式は runner 2.337.0 で確認したものである。runnerの更新で形式が変わり見つからなくなった場合、Step 1はFAILになる（fail closed）。その場合は本手順の更新が必要になる

#### 記録するもの

Durable Acceptance Record（`harness/REMOTE_REVIEW.md` §20）へ、`listener.json` の値とStep 3の結果を残す。

| 項目 | 取得元 |
|---|---|
| runner directory | `RunnerDir` |
| Runner Mode | Interactive / Service |
| 専用runner user | 設定値 |
| ACL検証 | §5.1 検証 (1)・(2) の結果 |
| local registration | `RunnerAgentId` / `RunnerAgentName` |
| Listener | `ListenerPath` / `ListenerPid` / `ListenerCreatedUtc` / `ListenerOwner` |
| GitHub run | run ID / attempt、jobの `runner_id` / `runner_name` / `started_at` / `completed_at` |
| job環境 | `metadata.json` の `run.id` / `run.attempt` / `run.runner_name` |
| local trace | `WorkerLog` / `WorkerLogCreatedUtc` |
| timing | `CapturedUtc`、Timing ruleの判定 |
| 判定 | `LISTENER CHECK` / `BINDING CHECK` / 総合 PASS / FAIL |
| 実施日 | 日付 |

- password・token・SID値・個人のhome path等は記録しない。hostnameは必要なら一般名へ置換してよい
- 記録欠落・不一致はいずれも **§20 FAIL**

### 6.2 Acceptance Test手順

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
| `Verify and review (self-hosted)` が Queued のまま | Runner offline、label不一致、PCスリープ。Interactive modeでは `run.cmd` が起動されていない・再起動/logoffで停止した（§5.4） | `<runner-user>` でサインインして `run.cmd` を起動する、Runners画面で Idle を確認、`REMOTE_REVIEW_RUNS_ON` を確認 |
| INCOMPLETE + `AUTHENTICATION_FAILURE` | Codex loginの期限切れ | `<runner-user>` で `codex login` |
| INCOMPLETE + `Codex CLI was not found` | runner processから `codex` が見えない | `REMOTE_REVIEW_CODEX_BIN` を設定、runner processを再起動（§5.5） |
| Install が INFRA_ERROR（EPERM / EBUSY） | antivirusのスキャン、前回processの残留 | 再度 `/review`、runner processを再起動（§5.5） |
| Install が INFRA_ERROR（network） | npm registryに届かない | 再度 `/review` |
| Format check が多数のfileでFAILし、Infrastructureに `CRLF` の記録がある | checkoutで改行がCRLFへ変換された | workflowの `GIT_CONFIG_*`（`core.autocrlf=false`）が残っているか確認する |
| path too long | Windows MAX_PATH | Runner directoryを短くする、`core.longpaths` |
| INCOMPLETE + `TOOL_CHECK` の記録（UNAVAILABLE / MISSING） | Codexがread-onlyコマンドを実行できなかった（例: Windows sandbox未設定で `blocked by policy`） | `codex.log` を確認し、§4 のsmoke testをRunnerユーザーでやり直す |
| INCOMPLETE + `Codex could not start its tool host` | Codex CLIの補助実行ファイル（tool host）が無い | npm版 `@openai/codex` を使う、`REMOTE_REVIEW_CODEX_BIN` を確認、§4 のsmoke testをやり直す |
| Codexが境界違反（checkout changed） | Codex CLIの挙動変化 | Runnerを停止し、Codex CLIのversionと `codex.log` を確認 |

## 8. 停止・削除

- 一時停止:
  - Interactive mode: 稼働中の `run.cmd` を停止する（§5.3）
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
- 2026-09-23 L2 Independent Review remediation Round 1（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: (1) 登録手順にRunner directoryのACL hardeningが無く、既定NTFS権限のままでは通常アカウントがrunner binary・`_work` を書き換えられた。§5.1 Runner directoryのACL hardening を新設し、`icacls /inheritance:r` + SYSTEM（`*S-1-5-18`）/ Administrators（`*S-1-5-32-544`）/ 専用runner user への `(OI)(CI)F` 付与と、`Get-Acl` による検証scriptを追加した。Service modeでrunnerが作る `GITHUB_ActionsRunner_*` groupがInteractive modeでは作られないため、継承ACEを外す際にrunner userへ明示付与しないとrunnerが `_work` / `_diag` へ書けなくなる点も明記した。コマンドは runner 2.337.0 / Windows 10 Pro 上で、hardening前後に対してFAIL / PASS双方が出ることを実機確認済み。(2) §6へ 6.1 Identity Evidence を新設し、`Get-CimInstance Win32_Process` + `Invoke-CimMethod GetOwner` による `Runner.Listener` の実 process owner確認（`whoami` では代替しない）と、run ID・runner name・採用mode・ACL検証結果を1つの記録として結び付ける Run binding 表を追加した。構文は実機の稼働processに対して `Domain\User` / `ReturnValue 0` を返すことを確認済み。**節番号の変更**: §5.1の新設に伴い、従来の §5.1 起動 / §5.2 停止 / §5.3 再起動・logoff / §5.4 Mode変更 を、それぞれ §5.2 / §5.3 / §5.4 / §5.5 へ繰り下げ、本文中の参照（§5 step 4、§7 Troubleshooting、§8 停止・削除）を更新した。2026-09-23の前エントリ本文にある「§5.1〜§5.4」の表記は当時の節番号であり、現在は§5.2〜§5.5を指す。**専用 `<runner-user>` 運用、Administrators非追加、Codex以外の認証情報を置かない要求、Runner Mode（Interactive採用）、AT-01〜AT-29は変更していない**
- 2026-09-23 L2 Independent Re-review remediation Round 2（CORE-RR-L2-001 / CORE-RR-L2-002、Blocking、docs-only）: Round 2のwhole-PR再レビューで、両Findingが未解消と判定された。(1) CORE-RR-L2-001: 旧§5.1の検証はrootだけを見て、許可集合外のidentityが無いことしか判定しておらず、必須principalの存在・Allow / Deny・必須right・descendant・ownerを確認していなかった。旧手順の `/inheritance:r` は継承ACEしか消さず、`/grant` は追加であるため、rootの既存explicit ACE（Pilot runnerではrunner groupのexplicit FullControl ACE）とdescendantのexplicit ACE・保護が残った。§5.1を、tree全体のinvariant（許可principal3つだけ・必須FullControl・Deny無し・root protected・protected descendant無し・owner制限・reparse point無し・全commandのexit code確認）、`icacls` semanticsの表、`/reset /T` → `/setowner /T` → `/grant:r` → `/inheritance:r` のhardening script、tree全体を走査する検証scriptへ置き換えた。`GITHUB_ActionsRunner_*` groupは許可principalから外した（membershipの汎用検証を定義できないため）。使い捨てdirectoryでPASS 3件・FAIL 13件・例外停止2件のtestを実施した。本番runner directoryのACLは変更していない。(2) CORE-RR-L2-002: 旧§6.1はprocess名だけでListenerを選び、PIDとownerを手作業でrun IDとrunner nameへ対応付けていた。§6.1を Runner Identity and Run Binding へ置き換え、target installationの `bin\Runner.Listener.exe` と一致するprocessがちょうど1つであること、owner SID、作成時刻、同じdirectoryの `.runner`（`agentId` / `agentName`）、jobs APIの `runner_id` / `runner_name` / `started_at` / `completed_at`、既存 `metadata.json` の `run.id` / `run.attempt` / `run.runner_name`、同じinstallationの `_diag` のWorker logの `run_id` を結び付けるscriptと、PID + 作成時刻によるTiming rule（60秒の余裕）を定義した。§6の番号付き手順へ §6.2 の見出しを付けた。§5.5へ、Mode変更時に§5.1をやり直すことを追加した。**Evidence Schema・workflow・script実装・permissions・AT-01〜AT-29・Runner Mode（Interactive採用）・専用 `<runner-user>` 運用は変更していない**
