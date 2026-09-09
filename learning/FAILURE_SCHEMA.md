# OTOMO Failure Schema

## 1. Purpose

重要な失敗を、修正して忘れるのではなく再利用可能な学習データとして残す。

Failureの実データは各Product Repositoryが保持する。OTOMO COREは記録形式のみ管理する。

## 2. Record When

以下のいずれかに該当する場合は記録候補。

- blocking bug
- セキュリティ問題
- データ破壊・整合性問題
- 回帰
- 設計ミス
- AIが繰り返し起こしそうな誤り
- 独立レビューで初めて発見された重大問題
- 実装方針を変更する原因となった問題
- 他Productでも起こりうる問題

軽微なtypoなどは原則記録しない。

## 3. Required Format

### [Date / Phase] Title

- What happened:
- Root cause:
- Impact:
- Fix:
- Prevention:
- Evidence:
  - commit:
  - Issue:
  - PR:
- Reproduction:
- Knowledge promotion candidate: Yes / No
- Generalized lesson:
- Possible enforcement:
  - Rule
  - Test
  - Hook
  - Skill
  - Workflow
  - Checklist
  - None

## 4. Principle

Failure logは日報ではない。

目的は「次回、同じ失敗をどう防ぐか」である。

単なる結果ではなくRoot causeとPreventionを残す。

## 5. Product Ownership

例:

- OTOMO塾で発生したFailure → OTOMO塾Repository
- OTOMO LABで発生したFailure → OTOMO LAB Repository
- 共通化された教訓 → OTOMO CORE
