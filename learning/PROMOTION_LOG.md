# OTOMO CORE Promotion Log

OTOMO各Productで得たFailure・改善知識のうち、一般化・レビュー・人間承認を経てOTOMO COREへ正式昇格した内容を記録する。

Product側のFailure原本は移動・削除しない。ここでは昇格したKnowledgeと、どのCORE Harnessへ反映したかを追跡する。

---

## P-001 — Remediationの横展開確認

- Date: 2026-09-09
- Source Product: OTOMO塾
- Source Repository: `t0t005081114-hue/ai-teacher`
- Source Failure: `[Phase 3 re-review Round 3] Claim-level corroborationがVERIFIED_MULTI_SOURCEにしか適用されていなかった`
- Evidence commit: `2681d1d351ec31133ea53afdec7246afc5bfaabd`
- Status: Approved / Applied

### What happened

同じ根本原因に対するRound 2の修正が、共通関数 `verify_by_tiers` の一つの戻り値経路にだけ適用され、別の戻り値経路に同種の問題が残った。独立再レビューでBLOCKINGとして発見され、Round 3で修正された。

### Generalized lesson

同じ根本原因に対する修正を一つの分岐・戻り値・call siteへ適用した場合、その修正だけで完了と判断しない。

修正対象の共通関数・共通パターンが到達しうる関連分岐、戻り値、call site、類似データ経路を列挙し、同じ問題が残っていないことを横展開確認する。

### Enforcement

Documentationだけではなく、OTOMO CORE `harness/PHASE_WORKFLOW.md` のFAIL Loopへ必須手順として昇格する。

Remediation時は:

1. Root causeを特定する
2. 同じRoot causeが影響しうる関連経路を列挙する
3. 必要な全経路へ修正を適用する
4. 修正対象だけでなく関連経路にも回帰Testを追加する
5. Validationと独立再レビューを行う

### Promotion destination

- `harness/PHASE_WORKFLOW.md` — FAIL Loop

### Why this belongs in CORE

- 特定ProductのBusiness ruleではない
- 特定言語・Frameworkに依存しない
- 独立レビューで実際に再発したFailureから得られた
- 将来のOTOMO LAB / VOX / LOOP / COMES等でも同様に起こりうる
- Workflowと回帰Test要求によって再発防止へ変換できる

---

## P-002 — External Boundary Safety

- Date: 2026-09-09
- Source Product: OTOMO塾
- Source Repository: `t0t005081114-hue/ai-teacher`
- Source Failure:
  - `[Phase 0 remediation] Claude CLI stderrによる機密情報漏洩リスク`
  - `[Phase 2 remediation] Source fetchのURL機密漏洩・HTTPエラーの一律Retry・RSS Tier固定`
  - `[Phase 2 re-review] RSS Tier未指定の許容・Source fetch例外チェーンによる機密漏洩残存`
- Evidence commit: `75a0afa`, `c5b57e7`, `410e6e6`, `c142e26`
- Status: Approved / Applied

### Generalized lesson

外部プロセス、外部API、外部Service等の境界では、機密を含みうる生出力がlog、exception、tracebackへ流れる設計を避ける。後段のマスキングだけに依存せず、安全な診断情報と生出力を境界で分離し、必要最小限へ縮約する。

Retryはすべての失敗へ一律に適用せず、transient / permanentを分類した上で、transientな失敗だけを対象にする。

### Enforcement

外部境界での情報縮約、観測経路への生出力混入防止、Retry前の失敗分類を共通Ruleとして要求する。

### Promotion destination

- `harness/DEVELOPMENT_STANDARDS.md` — External Boundary Safety

### Why this belongs in CORE

- 外部境界を持つ複数のProductへ適用できるSecurity原則である
- 特定の外部Service、言語、libraryに依存しない
- 異なる外部入出力経路で繰り返し発見されたFailureに基づく
- 境界での分離・縮約とRetry分類という共通Ruleで再発を抑制できる

---

## P-003 — Explicit Invariants / Fail Fast

- Date: 2026-09-09
- Source Product: OTOMO塾
- Source Repository: `t0t005081114-hue/ai-teacher`
- Source Failure:
  - `[Phase 1 re-review] SQLiteマイグレーションの非原子性・legacyデータのfail-unsafe処理`
  - `[Phase 2 re-review] RSS Tier未指定の許容・Source fetch例外チェーンによる機密漏洩残存`
- Evidence commit: `bf691eb`, `c142e26`
- Status: Approved / Applied

### Generalized lesson

正しさ、信頼度、分類、安全性、Business判断に関係し、仕様が明示的な判断を要求する値へ、利用頻度だけを理由に暗黙defaultを設定しない。未指定なら境界で明示的に失敗させる。

既知のdata / config invariant違反により安全または正しく処理できない場合、警告だけで続行せず、原因を特定できる境界でfail fastする。

### Enforcement

明示的判断を要する値の必須化、境界でのinvariant検証、安全に続行できない状態の早期停止を共通Ruleとして要求する。

### Promotion destination

- `harness/DEVELOPMENT_STANDARDS.md` — Explicit Invariants / Fail Fast

### Why this belongs in CORE

- Product固有の値ではなく、設定・入力・データを扱う共通の安全原則である
- 暗黙defaultによる誤分類と、既知の不整合を抱えた処理続行を横断的に防止できる
- 特定のDB engine、data model、実装方式に依存しない
- 境界検証と早期停止という明確なEnforcementへ変換できる

---

## P-004 — Aggregation / Confidence Integrity

- Date: 2026-09-09
- Source Product: OTOMO塾
- Source Repository: `t0t005081114-hue/ai-teacher`
- Source Failure:
  - `[Phase 3 re-review] Verification Gateの過大評価`
  - `[Phase 3 re-review Round 2] Verification Gate過大評価の残存`
- Evidence commit: `13efa64`, `1567130`
- Status: Approved / Applied

### Generalized lesson

件数、source数、vote数、event数、record数等を使ってconfidence、trust、verification、importance等を格上げする場合、数えている実体のdistinct性を確認し、duplicate IDやduplicate eventを除外する。

同じgroup / cluster / themeに属することと、同じclaim / fact / decisionを裏付けることを区別する。集約単位より細かいsemantic relationが必要なら、その対応関係を明示的に確認し、件数の多さだけでsemantic confidenceを上げない。

### Enforcement

件数ベースの格上げに対し、重複排除、集約関係と意味的裏付けの区別、必要な粒度での対応確認を共通Ruleとして要求する。

### Promotion destination

- `harness/DEVELOPMENT_STANDARDS.md` — Aggregation / Confidence Integrity

### Why this belongs in CORE

- 集約結果から信頼度や重要度を判断する複数のProductへ適用できる
- 特定の判定名、field、data modelに依存しない
- 件数条件だけを強化しても解消しなかった実際のFailureに基づく
- P-001のremediation横展開とは異なり、集約とsemantic confidenceの関係を定める設計原則である
