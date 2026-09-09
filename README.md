# OTOMO CORE

OTOMO COREは、OTOMO各Productの開発・接続・学習・改善を支える共通基盤です。

OTOMO CORE自体はProductではありません。各Productの機能やビジネス仕様を実装する場所ではなく、Product横断で共通利用する開発ルール、実行Workflow、失敗学習、責務境界を管理します。

## Purpose

1. 各Productの開発方法を共通化する
2. 実装と独立レビューを分離する
3. 重要な失敗を再発防止につなげる
4. Productで得た知見を他Productでも利用可能にする
5. 共通ルールとProduct固有仕様を分離する
6. OTOMO全体の構造を一か所から把握できるようにする

## Current Products

現在の主対象:

- OTOMO LAB
- OTOMO塾

今後追加候補:

- OTOMO VOX
- OTOMO LOOP
- OTOMO COMES

対象Productの正式な状態は `architecture/PRODUCT_REGISTRY.md` を参照してください。

## Source of Truth

GitHubは、仕様・コード・開発ルール・Issue・PR・commit・レビュー結果・重要判断・失敗履歴を保持する開発上のSource of Truthです。

Obsidianは、Product横断で再利用する知識・学び・一般化された知見を保持します。

> GitHub = What happened / Obsidian = What we learned

## Core Documents

### Architecture

- `architecture/PRODUCT_REGISTRY.md`: OTOMO Product一覧と現在の位置づけ
- `architecture/RESPONSIBILITY_BOUNDARIES.md`: OTOMO COREと各Productの責務境界

### Harness

- `harness/DEVELOPMENT_STANDARDS.md`: 全Product共通の開発標準
- `harness/PHASE_WORKFLOW.md`: Phase単位の実装・検証・レビュー・完了手順

### Learning

- `learning/FAILURE_SCHEMA.md`: Product側で重要な失敗を記録する共通形式
- `learning/KNOWLEDGE_PROMOTION.md`: Product内の知識をOTOMO COREへ昇格させる条件と手順

## Core Principle

OTOMO CORE自身を開発目的にしない。

実際のProductで必要になった仕組みだけを共通化します。先に抽象化しない。先に中央集権化しない。先に自動化しない。

Productで実証された仕組みを、必要に応じてCOREへ昇格させます。
