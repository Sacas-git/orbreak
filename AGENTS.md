# AGENTS.md — ORBREAK（仮題）開発ルール

このリポジトリは、2D円形フィールドのブロック崩しゲーム「ORBREAK」（仮題）の Godot プロジェクトです。
作業前に必ず次の順で読んでください。

1. `docs/01_GDD.md` … ゲーム仕様（何を作るか）
2. `docs/02_TECH_SPEC.md` … 技術設計（どう作るか）
3. `docs/03_TASKS.md` … 実装タスク一覧（どの順で作るか）
4. `docs/04_BALANCE.md` … 調整用の数値（すべてここと `data/` に集約）
5. `docs/05_GAMEOFF_PLAN.md` … GitHub Game Off 2026 の提出計画とスコープ

## 現在の状態（2026-10-17 時点）

- 公開リポジトリ `Sacas-git/orbreak`（main ブランチ）を作成済み。中身は仕様書（このファイル、`docs/`、`data/`）と `.gitignore`（Godot 用）、`.gitattributes` のみ。
- **ゲームのコードはまだ1行もない**。Godot プロジェクト（`project.godot`）はリポジトリ直下に作る（T00）。
- **ジャム開始（2026-11-02 06:37 JST ＝ 11/1 13:37 PST）より前は、コード・シーン・アセットを作らない**。この期間に頼まれてよいのは仕様書の読み込み、質問、`docs/` の修正提案だけ。
- 開発機は Windows。ローカルのパスは `C:\Users\shogo_fujisaka\Desktop\newgame\orbreak`。人間は GitHub Desktop でコミット／push する。
- Godot の実行ファイルは PATH に無い場合がある。コマンド例の `godot` は、環境変数 `GODOT` があればそれを使い、無ければ人間に場所を聞く。

## 技術前提

- エンジン：Godot 4.x（4.3 以降の安定版）。言語は GDScript のみ（C# は使わない）。
- 2D のみ。3D ノードは使わない。
- ゲームプレイの当たり判定に Godot の物理エンジン（RigidBody2D / Area2D / CharacterBody2D）は使わない。
  `scripts/core/collision.gd` の自前の計算で行う（決定論・性能・テストのため）。
- 画面：縦画面 720×1280 基準。`display/window/stretch/mode = "canvas_items"`, `aspect = "expand"`。
- 乱数はすべて `scripts/core/seeded_rng.gd` 経由（シード再現のため）。`randi()` などのグローバル乱数を直接呼ばない。

## コーディング規約

- GDScript は静的型付けを必ず使う（`var speed: float = 0.0`、`func foo(a: int) -> void:`）。
- 識別子は英語、コメントは日本語可。
- ファイル名・ノード名は snake_case、クラス名は PascalCase（`class_name` を付ける）。
- 調整値をコードに直書きしない。`data/balance.json` から `Balance` オートロード経由で読む。
- `scripts/core/` 以下はノードに依存しない純粋なロジック（`RefCounted` 継承）にし、単体テストを書く。
- シーン間の連絡はシグナル。グローバルなイベントは `Events` オートロード（シグナルバス）を使う。
- 1フレームで生成・破棄するオブジェクトを作らない。ボール・破片は配列ベースのプールで管理する。

## テスト

- テストフレームワーク：GUT（`addons/gut`）。
- 実行：`godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://tests -gexit`
- `scripts/core/` の変更には必ずテストを追加・更新し、全テストが通る状態でタスクを終える。

## ジャム（Game Off 2026）期間中のルール

- リポジトリは公開。API キー、個人情報、署名ファイルなどを絶対にコミットしない。
- `【J】` の付いたタスクを優先し、付いていないタスクには手を付けない。
- 書き出しは Web（HTML5）を最優先。Web で動かない機能（スレッド、ネイティブ専用 API）を使わない。
- ライセンス：コードは MIT（`LICENSE` を置く）。

## 作業の進め方

- `docs/03_TASKS.md` のタスクを ID 順に1つずつ実装する。1タスク＝1コミット（または1PR）。
- 各タスクの「完了条件」をすべて満たしたら、`docs/03_TASKS.md` の該当チェックボックスを `[x]` にする。
- 仕様にない判断をした場合は `docs/DECISIONS.md` に1行で追記する（日付・内容・理由）。
- 仕様が曖昧で手触りに関わる部分は、数値を `data/balance.json` に出して仮の値で実装し、人間が調整できるようにする。
- アセット（画像・音）は当面プレースホルダー（コードで描く図形、無音または簡易SE）でよい。外部素材を勝手に追加しない。

## 禁止事項

- 「アルカノイド」「ばくれつカブトムシ」など既存作品の名称・キャラクター・画像・音をコードやアセット、文言に入れない。
- 課金・広告 SDK は M5 まで入れない。
- 依存アドオンは GUT 以外、人間の承認なしに追加しない。
