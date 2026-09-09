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
