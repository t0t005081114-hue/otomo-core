# OTOMOの作業分担

## 1. 2026-10-23 Human Accepted Decision

人間の承認に基づき、3Lead構造を廃止し、Claude Codeが実装、Codexが独立レビューと開発進捗管理、人間が重要判断を担当する。役割の正本は `harness/DEVELOPMENT_STANDARDS.md` §1。

## 2. 作業の流れ

1. 人間が目的・仕様・優先順位を決める。Codexは判断材料と作業案を整理する。
2. Claude Codeが承認された範囲を実装・修正し、テスト・自己レビュー・Review Handoffを行う。
3. Codexが変更リスクと必要なレビュー範囲を確認し、独立レビュー・必要な検証を行う。
4. Claude Codeが指摘を修正し、Codexが再確認する。既存の再レビュー停止条件に従う。
5. Codexが完了条件・残件・次の作業を証拠とともに整理し、重要判断と最終承認を人間へ戻す。

## 3. 進捗管理

Codexは対象Productの仕様、Phase記録、Issue、PR、commit、テスト結果、レビュー結果を一次情報として確認する。
報告は「現在のPhase・対象SHA / 完了したこと / 未完了・blocking / 次の作業 / 人間の判断が必要なこと」を基本とする。
未確認・推測・古い情報は明示し、実装担当の完了報告だけで完了を確定しない。
進捗は対象Productの既存記録先へ集約し、COREへProduct固有backlogを複製しない。
独立レビュー担当はrepository fileを変更しない。進捗文書の更新も必要ならClaude Codeへ変更内容を引き継ぐ。
常時監視・自動実行・別タスクへの無断指示をこの役割から暗黙に許可しない。

## 4. 境界と移行

人間が仕様・優先順位・Scope・価格・予算・新Product・重要な外部操作・最終承認を判断する。
Lead廃止を理由にAIへ新しい営業・投稿・支出権限を与えない。確認できない権限や予算を推測しない。
既存の投稿allowlist・費用20%基準・経営データ・週次報告の安全境界は `harness/OPERATING_GUARDRAILS.md` に維持する。
Product固有の承認・安全性・認証・不可逆操作の制御を維持する。VOXの外部配信承認もProductの正本に従う。
ルールの優先順位は `RESPONSIBILITY_BOUNDARIES.md` §5に従う。

`otomo-marketing` / `otomo-ops` は既存の業務記録repositoryとして扱い、削除・移動しない。
日常の報告は事実・整合性、施策記録は結果と学びを確認する。Software / Tool / Automationの制作とCORE Harness変更には既存の開発・リスク分類・レビュー手順を適用する。
各repository内に残る旧Lead参照は個別移行が必要であり、COREの変更だけで移行完了としない。旧名称を権限の根拠にしない。

## 5. 変更の状態

本ファイルは2026-10-23のHuman承認を具体化した現行ルールである。この変更を含むPRの独立レビュー・承認・mainへのmergeをもって、approved CORE baseline上で有効になる。merge前のbranchは正式なCORE baselineとして扱わない。以後の変更も既存の独立レビューと承認・merge手順に従う。
旧設計と判断経緯は `history/TRINITY_V1.md` に保存する。
