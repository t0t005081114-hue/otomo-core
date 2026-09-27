# OTOMO Product Registry

OTOMO COREが認識するProductと、その現在の役割・Repository・状態を管理します。

Product詳細仕様はここには記載しません。

最終棚卸し: 2026-09-17

最終更新: 2026-09-27（OTOMO VOX / OTOMO LOOPの責務整合。全Productの再棚卸しではない）

## Status Guide

- `Active`: Productとして継続運用中
- `Active Development`: Repositoryが確定し、実装またはPhase作業が進行中
- `Planned`: 構想・責務はあるが、実装開始またはRepository確定前

Repositoryの存在とOTOMO CORE Adoptionは別に管理する。Repositoryが存在していても、CORE共通Harnessとの接続・移行が未完了なら、その旨を各Product項目に明記する。

## Active

### OTOMO LAB

- Product ID: `otomo-lab`
- Repository: `t0t005081114-hue/otomo-lab`
- Status: Active
- Type: Application / Experience / Product Experiment Layer
- Responsibility:
  - OTOMOの実践やProductを外部へ提示する
  - 市場との接点を作る
  - Product Experimentの入口になる
- Product Specification Owner: `otomo-lab` Repository
- OTOMO CORE Adoption:
  - Migration Step 1 complete: `CLAUDE.md` recognizes OTOMO CORE as the upper shared foundation
  - Migration Step 2 complete: common-vs-Product-specific audit recorded and OTOMO LAB-specific rules extracted to `harness/PRODUCT_RULES.md`
  - Migration Step 3 complete: standard local sibling checkout confirmed; duplicated local shared documents reduced to Product-local shims
  - Product-specific business / brand / UI / KPI / MVP scope remains owned by `otomo-lab`
  - Product-specific validation commands are owned by `otomo-lab`
  - Legacy local `sns-growth-os` is not the same Git repository as official `otomo-lab` and is outside automatic migration
  - Remote Review pilot is implemented in Product (`.github/workflows/remote-review.yml` + `scripts/remote-review/`); OTOMO CORE側仕様は2026-09-25 Human DecisionによりSuspended / Non-default Pilot（`harness/REMOTE_REVIEW.md` Suspension）。Product側の実装は削除していない

### OTOMO塾

- Product ID: `otomo-juku`
- Current Repository: `t0t005081114-hue/ai-teacher`
- Status: Active Development
- Type: Knowledge / Learning Product
- Responsibility:
  - Research
  - Verification
  - Selection
  - Teaching
  - Learning Support
- Product Specification Owner: `ai-teacher` Repository
- OTOMO CORE Adoption:
  - Existing harness was the primary source used to derive OTOMO CORE v0.1
  - Migration Step 1 complete: `CLAUDE.md` recognizes OTOMO CORE as the upper shared foundation
  - Migration Step 2 complete: overlap audit recorded and Product-specific validation extracted to `harness/VALIDATION.md`
  - Migration Step 3 complete: standard local sibling checkout confirmed; duplicated local shared documents reduced to Product-local shims
  - Product requirements / formal specification / Decision-Failure history / Validation remain owned by `ai-teacher`

NOTE: Product正式名称はOTOMO塾だが、現時点のRepository名は `ai-teacher`。Repository renameは本ファイルでは決定しない。

### OTOMO VOX

- Product ID: `otomo-vox`
- Repository: `t0t005081114-hue/otomo-vox`
- Status: Active Development
- Type: AI PR / Content Transformation & Publication Execution Product
- Responsibility:
  - AI広報
  - Source content / Content Intentを媒体別の発信内容（Platform Rendition）へ変換する
  - Human承認を強制した上で、スケジューリングと公式API経由の配信を実行する
  - 配信の運用状態（Publication / attempt / result / error）を管理する
  - 発信Packageのための公開投稿収集・構造分析・Pattern候補化の仕組み（Package Research）を持つ
  - 何を・なぜ・誰へ・どのチャネルで発信するか等の戦略判断と、Package Research結果の戦略的解釈はSHIVAが所有する（`architecture/LEAD_AGENTS.md` §6）。VOXはその実行の仕組みである
- Product Specification Owner: `otomo-vox` Repository
- Current Development State（2026-09-27）:
  - Phase 0 `Package Research Feasibility Test` 完了（PR #1で`main`へmerge）
  - PH-01 Architecture Rebaseline 完了（PR #2でarchitectureを、PR #3でCompletion Recordを`main`へmerge）。SHIVA / VOX / LOOPの責務分担（2026-09-26 Human Decision）のもとで、VOXを配信実行Productとして再定義した
  - PH-02は未開始。最初の配信Platformは未決定
- OTOMO CORE Adoption:
  - `main` の `CLAUDE.md` がOTOMO COREを上位共通基盤として明示
  - standard local sibling checkoutでCORE shared rulesを直接参照
  - Product固有Ruleは `harness/PRODUCT_RULES.md` が所有
  - Product requirements / data boundary / validation / Decision-Failure historyは `otomo-vox` が所有
  - Registry上のPlanned表記が実態と不一致だったため、2026-09-17のHuman指示による棚卸しでActive Developmentへ更新
  - 2026-09-26のHuman Decision（SHIVA / VOX / LOOPの責務分担。記録: `otomo-vox` `docs/ARCHITECTURE.md` §1.1）に基づき、2026-09-27にType / Responsibility / Current Development Stateと、OTOMO LOOPのResponsibilityを整合した

### OTOMO COMES

- Product ID: `otomo-comes`
- Repository: `t0t005081114-hue/otomo-comes`
- Status: Active Development
- Type: AI Manager / Internal Management Support Product
- Responsibility:
  - AIマネージャー
  - Task / progress / management support
  - 判断材料の集約・委譲候補・要フォロー等の支援
- Product Specification Owner: `otomo-comes` Repository
- Current Development State:
  - Product仕様・Acceptance・Accepted Decision・Open IssuesがRepository内に存在
  - AI開発Harness（`CLAUDE.md` / `AGENTS.md` / `.claude/` / Phase Workflow等）を整備済み
  - Phase 00 Development Foundationが`main`上で進行し、Next.js / TypeScript / test / CI等の基盤が存在
  - 最新のPhase 00作業ではclean installの再現性修正まで実施済み
- OTOMO CORE Adoption:
  - Repository自体とProduct-local Harnessは稼働している
  - 2026-09-17棚卸し時点では、`CLAUDE.md`等にOTOMO COREを上位共通基盤として参照する明示的Migrationは確認できない
  - CORE共通Ruleとの重複監査・責務分離・sibling checkout等の正式Adoptionは未記録
  - したがってRepository確定 / Active Developmentへの更新と、OTOMO CORE Adoption完了は別扱いとする

## Planned / Reserved

### OTOMO LOOP

- Product ID: `otomo-loop`
- Status: Planned
- Repository: Not fixed
- Responsibility:
  - 市場反応の観測・取得（配信の実行はOTOMO VOXが所有する）
  - Performanceの集約・比較
  - 市場検証
  - Learning / insight候補化
  - SHIVA / OTOMO Product（OTOMO LAB等）へのフィードバック
- 詳細な計測・収集設計は未決定（Human Decision未了）であり、本Registryでは定めない

2026-09-17棚卸し時点で、`t0t005081114-hue` 配下に `otomo-loop` Repositoryは確認できない。

## Out of Scope

以下は現時点でOTOMO COREの管理対象外とする。

- TREASURE LOG
- CRM関連Repository
- `threadpilot-ai-copilot`
- その他、明示的にOTOMO COREへ登録されていないRepository

対象追加は人間判断で行う。
