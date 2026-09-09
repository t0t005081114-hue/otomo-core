# OTOMO Product Registry

OTOMO COREが認識するProductと、その現在の役割・Repository・状態を管理します。

Product詳細仕様はここには記載しません。

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
  - Existing local `docs/DEVELOPMENT_STANDARDS.md` / `docs/PHASE_WORKFLOW.md` remain as migration fallback
  - Product-specific business / brand / UI / KPI / MVP scope remains owned by `otomo-lab`
  - Step 3 pending: reduce duplicated local harness documents after stable local CORE access and Product validation strategy are confirmed

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
  - Existing local `docs/DEVELOPMENT_STANDARDS.md` / `docs/PHASE_WORKFLOW.md` remain as migration fallback
  - Step 3 pending: reduce local duplicated harness documents after stable local CORE access is confirmed

NOTE: Product正式名称はOTOMO塾だが、現時点のRepository名は `ai-teacher`。Repository renameは本ファイルでは決定しない。

## Planned / Reserved

### OTOMO VOX

- Product ID: `otomo-vox`
- Status: Planned
- Repository: Not fixed
- Responsibility:
  - AI広報
  - Source contentを媒体別の発信内容へ変換する

### OTOMO LOOP

- Product ID: `otomo-loop`
- Status: Planned
- Repository: Not fixed
- Responsibility:
  - SNS自動運用
  - 市場反応取得
  - 市場検証
  - OTOMO LABへのフィードバック

### OTOMO COMES

- Product ID: `otomo-comes`
- Status: Planned
- Repository: Not fixed
- Responsibility:
  - AIマネージャー
  - Task / progress / motivation management support

## Out of Scope

以下は現時点でOTOMO COREの管理対象外とする。

- TREASURE LOG
- CRM関連Repository
- `threadpilot-ai-copilot`
- その他、明示的にOTOMO COREへ登録されていないRepository

対象追加は人間判断で行う。
