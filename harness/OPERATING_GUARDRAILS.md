# OTOMO業務の承認・費用・記録ルール

2026-10-23の3Lead廃止に伴い、旧設計で定めていた業務上の安全境界を名前から切り離して維持する。作業分担は `DEVELOPMENT_STANDARDS.md` §1、進め方は `architecture/WORKING_MODEL.md` が正本。本変更を含むPRのレビュー・Human承認・mainへのmerge後に有効となる。
節番号は既存repositoryからの参照を移行するため維持する。Lead層・新しい自律実行権限・自動化基盤は設けない。

## 4. Baseline / Source of Truth

### 4.1 現在値

「既存」は正式SOTから現在値を一意に確認できる状態だけを指す。AIの記憶、会話、推測、裏取りできない伝聞をbaselineにしない。

### 4.2 正本

- Product仕様・KPI・Phase状態: 各Product repository。
- ターゲット・価格・承認済みChannel・Operation Mode: `otomo-marketing` の対応するSOT。
- 経営数値・予算: §15。集計・週次報告: `otomo-ops`。
- 権限: Humanまたは正式SOTで明示承認・記録されたもののみ。

### 4.3 未記録の現在値

ターゲット・価格などの現在値が未記録なら、それを前提に自律実行しない。

### 4.4 権限

権限が確認できなければ権限なしとして扱う。「禁止されていない」を許可とみなさない。

### 4.5 Fail closed

SOT未存在・未記録・古い情報・競合・一意に判定不能の場合は推測せずHumanへ戻す。欠損を0や推定値で埋めない。

## 6. マーケティング支援

### 6.1 提案と判断

AIは調査・企画・原稿・施策の判断材料を作成できる。ターゲット・実価格・新Product・正式方針はHumanが判断する。

### 6.2 Productとの境界

企画・原稿とProduct実装を分ける。技術実装は承認されたScopeでClaude Codeが行い、Codexが独立レビューする。

### 6.4 Closed allowlist

自律施策は、§4のbaselineを確認でき、既存承認済みChannel・アカウント・playbook内の可逆な施策であり、以下に列挙された範囲に限る。

- copy案の作成
- Content theme変更
- Sales copy変更
- CTA文言変更
- 既存承認済みChannelへのContent投稿・実行要求（§6.4.1に従う）
- 既存承認済みSales導線内のmessage変更
- 既存Consultation導線の可逆な改善
- A/B test等の既存環境内で元に戻せるExperiment

この一覧は権限の上限であり、現在のOperation Mode・Product制御・明示権限との積集合で適用する。類似・安全・可逆という推測で一覧を拡張しない。追加にはHuman承認と正式SOTへの具体的な記録を要する。

#### 6.4.1 Product execution boundary

Product固有の承認・安全性・認証・不可逆な外部副作用の制御を上書き・迂回しない。VOXの現行Mode AではPublicationごとに外部provider呼び出し前のHuman承認を要する。承認の撤廃・緩和は別のHuman DecisionとProduct SOTの整合した更新を必要とする。

### 6.5 単独実行しないもの

企画支援の権限からProduct / Website / CMS / Tool / Automationの実装権限を推論しない。実装は§7.5へ。新外部契約、権限付与、API credential取得、個人情報の取得・保存・送信、Security / Authentication変更、不可逆操作、Product仕様・ターゲット・実価格変更、費用基準超過、許可を確認できない外部操作はHumanへ戻す。
instruction-onlyの原稿・Skill編集は実行能力の追加ではない。script・hook・scheduler・API連携など実行能力の追加は技術実装として扱う。

### 6.6 Experiment記録

仮説 / 実施内容 / 対象 / 結果 / 学び / 次アクションを `otomo-marketing/experiments/` に記録する。事実と推測を分け、結果をHuman Decisionの代わりにしない。

## 7. 技術実装

### 7.5 開発手順

Claude Codeが実装、Codexが独立レビュー・進捗管理を担当する。Software / Tool / Automationの制作には `DEVELOPMENT_STANDARDS.md` と `PHASE_WORKFLOW.md` を適用する。

## 8. 管理・報告

### 8.2 判断の境界

Codexは事実・異常・リスク・候補を提示する。中止・優先順位・次週重点などのBusiness判断はHumanが確定する。通常は常時監視・自動実行せず、依頼された確認・報告時に各Productを参照する。

### 8.3 KPI

全体KPIの定義・集計・報告は `otomo-ops`、Product固有KPIは各Productが正本。双方を上書き・混同しない。

## 11. Human Decision Required

新Product、Product仕様、ターゲット、実価格、事業優先順位の衝突、Security / Authentication / data-retentionの新判断、権限拡張、SOT競合、Scope拡大、§12超過、baseline不明はHumanへ戻す。判断材料を整理して提示し、AIが代行確定しない。

## 12. Cost Rule

### 12.1 判定式

固定費比率 =（現在の月額固定費合計 + 新規commitmentの月額換算額）÷ 直近1か月の確定売上。累計で判定し、初期閾値は20%。

### 12.2 判定

20%以内は費用比率について追加のHuman確認不要で記録する。ただし契約・外部操作等の別の承認条件を免除しない。20%超、確定売上0、未取得・未確定・古い売上、固定費を一意に確認できない場合はHuman Decision Required。欠損を0や推定値で埋めない。

### 12.3 売上期間

判定時点から遡る直近1か月の確定売上。カレンダー月へ独断で変更しない。

### 12.4 会計詳細

税込 / 税抜・売上認識は経営管理SOTの値を使う。一意に決まらなければHumanへ戻す。

### 12.5 Commitment

年払い等は一意に月額換算できる場合のみ換算する。換算不能ならHumanへ戻す。

### 12.6 変更

閾値・式をAIが独断変更しない。金額の一次情報は§15で保持し、COREへ記録しない。

## 13. 週次報告

金曜締め・土曜報告。Codexが進捗と判断材料を集約し、Humanが次週重点（候補は最大3件程度）を確定する。重大blocking・重大risk・Human Decision Requiredは週次を待たず報告する。自動スケジュール実行の許可を意味しない。
初期KPIは売上 / 相談・商談件数 / 有効Lead数 / 公開・検証中Product数 / 固定費。実際の定義・報告は `otomo-ops` を参照する。

## 14. Repository

`otomo-marketing` と `otomo-ops` は業務記録repositoryでありProductではない。正式な記録先がない状態で記録を要する自律施策を開始しない。COREをProduct・業務データの暫定保存先にしない。既存repositoryを移動・rename・統合せず、Product登録状態も変更しない。

## 15. 経営データ

一次情報はOTOMO経営管理Google Spreadsheet。経営データの列は日付 / 区分 / Product / 内容 / 金額 / 備考、区分は売上 / 固定費 / 変動費。予算の記録場所は未確定であり、推測で確定しない。
集計結果・週次報告・経営判断の記録は `otomo-ops`。参照は手動で行い、この移行ではAPI連携・同期・credential取得を追加しない。Spreadsheet URL / ID / 共有設定 / 顧客情報 / 秘密情報をcommitしない。実数値の記録可否・粒度は各repositoryのHuman承認済み方針に従う。

## 17. 追加の実行能力と予算

新規固定費なしまたは§12以内、新外部契約なし、権限拡張なし、Product仕様・価格・ターゲット変更なし、承認された作業範囲内であることを確認する。変動費の増加には§15の正式SOTで確認できる既存予算内であることを要する。予算・残額・usage増加量を推定しない。予算SOTなし・未確認・古い・残額を一意に判定不能ならHumanへ戻す。他タスクへの指示や委譲の権限を本節から推論しない。

## 18. Review model

施策はExperiment Review / PDCA、日常報告は整合性・事実確認を行う。開発用L0 / L1 / L2を日常業務へ機械的に適用しない。技術実装・Product変更・CORE Harness変更には開発標準のリスク分類と独立レビューを適用する。

## 19. 優先順位

`architecture/RESPONSIBILITY_BOUNDARIES.md` §5に従う。Security guardrailsを弱めない。Product固有仕様・承認条件は各Productが正本であり、矛盾をAIが独断解消しない。

## 20. ルール変更

本書の費用・権限・承認境界の変更はHuman承認と既存COREレビュー手順を要する。役割の廃止を理由に権限を広げない。
