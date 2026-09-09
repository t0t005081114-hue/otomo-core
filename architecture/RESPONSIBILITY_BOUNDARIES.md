# OTOMO CORE Responsibility Boundaries

## 1. Purpose

OTOMO COREと各Productの責務混在を防止する。

## 2. OTOMO CORE Owns

OTOMO COREが正本として管理するもの:

- 共通開発標準
- Phase Workflow
- 独立レビュー原則
- AI開発の基本責務分担
- 失敗記録フォーマット
- Knowledge昇格基準
- Product Registry
- Repository構造の共通原則
- Product横断で有効と承認されたRule / Skill / Hook / Test等

## 3. Product Owns

各Product Repositoryが正本として管理するもの:

- requirements
- 正式仕様
- Product固有の責務
- UI / UX
- DB設計
- Product固有ビジネスルール
- Product固有KPI
- Product固有技術選定
- Product固有Validation command
- Product固有Decision
- Product内で発生したFailure履歴

## 4. OTOMO CORE Does Not Own

OTOMO COREに以下を置かない。

- Product画面
- Productコード
- Product DB
- Product顧客データ
- Product固有Prompt
- Product固有コンテンツ
- Product固有Feature backlog
- Product固有Roadmap

## 5. Rule Precedence

原則:

1. Security / secret leakage / destructive-operation guardrails
2. Product requirements
3. Product正式仕様
4. Product固有Rule
5. OTOMO CORE共通Rule
6. AIエージェントの推測

Product要件とCORE共通Ruleが衝突した場合は、人間へ判断を戻す。AIが独断でどちらかを書き換えない。

## 6. Common vs Product-Specific Test

次の質問で判断する。

> 別のOTOMO Productでも、そのまま適用して問題ないか？

YES: CORE候補  
NO: Product側

例:

- 「blocking reviewが残っていればPhase完了にしない」→ CORE
- 「pytestを実行する」→ Product
- 「Waitlistのメールをマーケティング利用しない」→ OTOMO LAB
- 「外部プロセスの機密を含みうる生出力をログへ出さない」→ CORE候補
