# 02_TECH_SPEC — 技術設計書

Godot 4.x / GDScript / 2D のみ

## 1. ディレクトリ構成

```
res://
├─ project.godot
├─ AGENTS.md
├─ docs/                     … 仕様書一式
├─ data/
│  ├─ balance.json           … 調整値（04_BALANCE.md 参照）
│  ├─ upgrades.json          … 強化定義
│  └─ planets/
│     ├─ s1_01.json …        … ステージ定義
├─ autoload/
│  ├─ events.gd              … シグナルバス（Events）
│  ├─ balance.gd             … balance.json の読み込み（Balance）
│  ├─ game_state.gd          … ラン状態・スコア・残機（GameState）
│  └─ save_manager.gd        … セーブ／ロード（SaveManager）※M4
├─ scripts/
│  ├─ core/                  … ノード非依存の純ロジック（テスト対象）
│  │  ├─ seeded_rng.gd
│  │  ├─ block_grid.gd
│  │  ├─ collision.gd
│  │  ├─ chain_score.gd
│  │  ├─ chain_tracker.gd
│  │  ├─ shard_pool.gd
│  │  ├─ planet_loader.gd
│  │  └─ upgrade_db.gd
│  ├─ game/                  … ノードにアタッチするスクリプト
│  │  ├─ arena.gd            … 1ステージの進行・更新ループの中心
│  │  ├─ planet_view.gd      … ブロック描画（MultiMesh）と回転
│  │  ├─ paddle.gd
│  │  ├─ ball_manager.gd
│  │  ├─ shard_view.gd       … 破片描画（MultiMesh）
│  │  ├─ stun_controller.gd
│  │  ├─ input_router.gd     … タッチ／マウス／キーを抽象化
│  │  ├─ fx.gd               … 画面揺れ・ヒットストップ・ポップ表示
│  │  └─ run_manager.gd      … ステージ遷移・強化選択
│  └─ ui/
│     ├─ hud.gd
│     ├─ upgrade_select.gd
│     ├─ title.gd
│     └─ result.gd
├─ scenes/
│  ├─ main.tscn              … ルート。画面遷移を管理
│  ├─ title.tscn
│  ├─ game.tscn              … Arena＋HUD
│  ├─ upgrade_select.tscn
│  └─ result.tscn
├─ tests/                    … GUT テスト（test_*.gd）
└─ addons/gut/
```

## 2. 座標系と単位

- 基準解像度 720×1280。単位はピクセル、角度は内部ではラジアン（JSON は度で書き、読み込み時に変換）。
- **ワールド座標**：画面座標。フィールド中心 `C = balance.arena.center`。
- **惑星ローカル座標**：惑星中心を原点、惑星の回転角 `planet_angle` を打ち消した座標。
  - `to_local(p) = (p - C).rotated(-planet_angle)`
  - `to_world(q) = q.rotated(planet_angle) + C`
- 角度 0 は右方向、反時計回りが正（Godot の y 下向きに注意：`Vector2.from_angle()` をそのまま使い、画面上では時計回りに見える。見た目の向きは統一さえすれば問題ない）。

## 3. 更新ループ

`arena.gd` の `_physics_process(delta)` を唯一のゲームループとする（固定 60Hz）。他ノードは自分で `_process` によるゲームロジックを持たない（描画更新は可）。

1フレームの処理順：

1. `input_router` から入力を取得（パドル目標角、回転入力、発射、連打）
2. 時間スケール適用（ヒットストップ中は `dt = 0`、スロー中は `dt *= 0.5`）
3. 惑星の回転更新（操作＋自転）
4. パドル更新（スタン中は動かない）
5. ボール更新（サブステップ、衝突、喪失判定）
6. 破片更新（移動、ブロック衝突、パドル衝突、寿命）
7. 破壊キューの処理（得点、破片生成、爆薬の誘爆、ハート出現判定）
8. 再生ブロックの増殖
9. スタン更新（タイムアウト判定）
10. クリア／失敗判定
11. 描画データの更新（MultiMesh へ反映）

破壊は即時ではなく **破壊キュー** に積み、手順7でまとめて処理する（同フレーム内の連鎖が順序に依存しないようにするため）。キュー処理中に生まれた破片は次フレームから動く。

## 4. 主要クラス

### 4-1. SeededRng（core/seeded_rng.gd）

```gdscript
class_name SeededRng extends RefCounted
var _rng := RandomNumberGenerator.new()
func _init(seed_value: int) -> void: _rng.seed = seed_value
func randf() -> float
func randf_range(a: float, b: float) -> float
func randi_range(a: int, b: int) -> int
func pick_weighted(weights: Array[float]) -> int   # 重みの添字を返す
```

### 4-2. BlockGrid（core/block_grid.gd）

惑星のブロック配置と状態を持つ。描画とは分離。

```gdscript
class_name BlockGrid extends RefCounted
const EMPTY := 0
enum Type { NONE, NORMAL, HARD, BOMB, CORE, REGEN, MIRROR, HEART }

var size: int            # 一辺のマス数（奇数）
var cell: float          # 1マスの大きさ(px)
var planet_radius: float
var types: PackedInt32Array   # size*size
var hp: PackedInt32Array      # size*size
var valid: PackedByteArray    # 惑星半径内のマスなら1

func idx(ix: int, iy: int) -> int
func cell_center_local(ix: int, iy: int) -> Vector2   # ((ix-(size-1)/2)*cell, (iy-(size-1)/2)*cell)
func local_to_cell(q: Vector2) -> Vector2i            # 範囲外は (-1,-1)
func is_solid(ix: int, iy: int) -> bool
func damage(ix: int, iy: int, amount: int) -> bool    # 破壊されたら true（MIRROR は常に false）
func layer_of(ix: int, iy: int) -> int                # 0=表層 1=マントル 2=コア（中心距離の比率で判定）
func empty_neighbors4(ix: int, iy: int) -> Array[Vector2i]
func count_remaining_cores() -> int
func count_destructible() -> int
```

- 層の判定：中心からの距離 / planet_radius が `> 0.66` 表層、`> 0.33` マントル、それ以下コア。

### 4-3. Collision（core/collision.gd）

静的関数のみ。

```gdscript
class_name Collision
# 円と軸平行矩形。当たっていれば {hit=true, normal=Vector2, depth=float}
static func circle_vs_aabb(c: Vector2, r: float, rect_center: Vector2, half: float) -> Dictionary
static func reflect(v: Vector2, n: Vector2) -> Vector2          # v - 2*v.dot(n)*n
static func angle_diff(a: float, b: float) -> float              # -PI..PI に正規化した差
static func paddle_bounce_dir(hit_offset: float, inward: Vector2, max_deflect: float) -> Vector2
```

### 4-4. ボールと惑星の衝突

ボールは `pos`（ワールド）, `vel`（ワールド）, `radius` を持つ。

各サブステップで：

1. `q = to_local(pos)`, `vl = vel.rotated(-planet_angle)`
2. `q` の周囲 3×3 マス（ボール半径＞セルの半分なら 5×5）の solid ブロックについて `circle_vs_aabb`。
3. 最も深い衝突を1つ採用し、`vl = reflect(vl, normal)`、`q += normal * depth` で押し出す。
4. そのブロックを破壊キューへ（ダメージ1、`depth=-1` を意味する「ボール由来」で積む）。貫通強化があり耐久1なら反射せず進む。
5. `pos = to_world(q)`, `vel = vl.rotated(planet_angle)`

サブステップ数：`ceil(speed*dt / balance.ball.max_step)`（max_step=6px）。

### 4-5. ボールとパドル

- `d = pos - C`, `dist = d.length()`
- `dist >= orbit_radius - ball.radius` かつ `vel.dot(d) > 0`（外向き）のとき判定：
  - `a = d.angle()`、`off = angle_diff(a, paddle_angle)`
  - `abs(off) <= paddle_half_width + ball.radius / orbit_radius` なら命中
  - `hit_offset = clamp(off / paddle_half_width, -1, 1)`
  - 新しい向き：内向き `-d.normalized()` を `hit_offset * max_deflect` 回転
  - 速度に `speed_up_per_hit` を掛け、上限で切る
  - 位置を `orbit_radius - ball.radius - 0.5` に戻す
- 命中しなかった場合、`dist > orbit_radius + lost_margin` で喪失。

### 4-6. 惑星外周のふち

惑星には壁を設けない。ブロックが無くなった部分はボールが中心を素通りして反対側へ抜ける（反対側のパドル軌道まで飛ぶ）。これは仕様。

### 4-7. ShardPool（core/shard_pool.gd）

構造体配列（SoA）で破片を管理。

```gdscript
class_name ShardPool extends RefCounted
var capacity: int
var count: int = 0
var pos: PackedVector2Array    # ワールド座標
var vel: PackedVector2Array
var life: PackedFloat32Array   # 残り寿命（秒）
var depth: PackedInt32Array    # 連鎖段数
var chain_id: PackedInt32Array
func spawn(p: Vector2, v: Vector2, life_sec: float, d: int, cid: int) -> bool   # 上限なら false
func kill(i: int) -> void      # 末尾と入れ替えて count-=1（swap-remove）
```

- 上限 `balance.shard.max_active`（3000）。超えた分は生成しない（得点はブロック破壊時点で入るので問題ない）。
- 更新：`pos += vel*dt`, `life -= dt`。`life<=0` で kill。
- ブロック判定：`local_to_cell(to_local(pos))` が solid なら命中 → 破壊キューへ（段数 depth）→ 破片は kill（鏡なら反射して生存）。点判定でよい（破片は小さい）。
- 破片速度 × dt がセルの半分を超える場合はサブステップする。
- パドル判定：`dist >= orbit_radius - 2` で角度がパドル内ならスタン発生、破片 kill。軌道を越えたら kill。

### 4-8. 破壊キューと連鎖

```gdscript
# 破壊キューの要素
{ "cell": Vector2i, "damage": int, "depth": int, "chain_id": int }
# depth = -1 はボール由来（得点1、出る破片の段数は0）
```

処理：

1. `BlockGrid.damage()` が true（破壊）なら得点：`ChainScore.points(depth, bonus_depth, cap)`
   - ボール由来（depth=-1）→ 1点
   - 破片由来（depth=d）→ `min(2^(d+1+bonus_depth), cap)`
2. 破片を生成：ブロック種別の破片数 n（＋強化）を、ランダム初期角 θ0 から 360/n 度ずつ等間隔に放射。速度 `shard.speed`、寿命 `shard.life`。段数 = depth+1（ボール由来なら0）。
3. 爆薬なら隣接8マスへ damage 1, depth 同じでキューに追加（同フレーム内で処理する。無限ループ防止のため同一セルは1フレーム1回まで）。
4. ハートなら StunController.reset_required()。
5. ChainTracker に段数を報告。

### 4-9. ChainScore / ChainTracker

```gdscript
class_name ChainScore
static func points(depth: int, bonus_depth: int, cap: int) -> int
# depth=-1 → 1、それ以外 → min(1 << (depth + 1 + bonus_depth), cap)。シフトのオーバーフローに注意（depth+1+bonus が 30 超なら cap を返す）
```

```gdscript
class_name ChainTracker extends RefCounted
# ボールがブロックを壊すたびに新しい chain_id を発行
func new_chain() -> int
func on_shard_spawned(cid: int) -> void
func on_shard_removed(cid: int) -> void
func report_depth(cid: int, depth: int) -> void
func max_depth(cid: int) -> int
# 生きている破片数が0になったら Events.chain_finished(cid, max_depth, total_points) を発火
```

### 4-10. StunController

- 状態：`stunned: bool`, `taps: int`, `required: int`, `timer: float`, `stun_count: int`
- `stun()`：stunned=true, taps=0, required = min(5 + stun_count, 15), timer=0, stun_count+=1
- `tap()`：taps+=1、`taps>=required` で復帰
- 毎フレーム timer+=dt、`timer >= 3.0` で失敗 → `Events.stun_failed`
- `reset_required()`：stun_count=0（次回 required が 5 に戻る）

### 4-11. 再生ブロック

- 再生ブロックごとにではなく、惑星全体で `regen.interval` 秒ごとに1回処理：
  全 REGEN ブロックからランダムに最大 `regen.max_spawn_per_tick` 個を選び、それぞれ空き隣接マス（valid かつ EMPTY）に REGEN を1つ置く。

## 5. 描画

- 惑星：`PlanetView`（Node2D、位置＝C、rotation＝planet_angle）の子に `MultiMeshInstance2D`。
  - QuadMesh（cell-2 px 四方）、`use_colors = true`。
  - インスタンス数＝ size*size。空マスは色の alpha=0 にする。
  - BlockGrid の変更セルだけ更新する（dirty リスト）。
- 破片：`MultiMeshInstance2D`（ワールド座標、回転しない）。`visible_instance_count = pool.count`。毎フレーム count 分の transform と色（段数で色相を変える）を書き込む。
- ボール：数が少ないので Node2D の `_draw()` で円を描いてよい。
- パドル・軌道：`_draw()` で `draw_arc()`。
- 発光：加算合成（CanvasItemMaterial の blend_mode = ADD）のぼかし円スプライトを重ねる。

## 6. 入力（InputRouter）

出力する値：

```gdscript
var paddle_target_angle: float   # 絶対角（PC・絶対モード）
var paddle_delta_angle: float    # 相対移動量（スマホ相対モード）
var rotate_input: float          # -1, 0, +1
var launch_pressed: bool
var tap_count_this_frame: int    # スタン連打用（タッチ開始、クリック、スペース押下の回数）
```

- InputMap アクション：`rotate_left`(Q)、`rotate_right`(E)、`launch`(Space, マウス左)、`pause`(Esc)。
- タッチ：`InputEventScreenDrag` の `relative.x * drag_sensitivity` を角度移動量に。画面下の回転ボタン領域でのタッチはドラッグとして扱わない。
- パドルは目標角へ `max_angular_speed` を上限に近づく。

## 7. シグナル（Events オートロード）

```gdscript
signal block_destroyed(cell: Vector2i, type: int, depth: int, points: int, world_pos: Vector2)
signal chain_finished(chain_id: int, max_depth: int, total_points: int)
signal chain_depth_reached(chain_id: int, depth: int)
signal ball_lost(remaining_balls: int)
signal life_lost(remaining_lives: int)
signal paddle_stunned(required_taps: int)
signal paddle_recovered()
signal stun_failed()
signal stage_cleared(stage_index: int)
signal run_finished(success: bool, score: int)
signal score_changed(score: int)
```

## 8. データ形式

### 8-1. ステージ定義 `data/planets/*.json`

```json
{
  "id": "s1_01",
  "sector": 1,
  "is_boss": false,
  "modifiers": ["spin"],
  "spin_speed_deg": 12,
  "layout": [
    "......nnn......",
    "....nnnnnnn....",
    "...nnnhhhnnn...",
    "..nnhhhhhhhnn..",
    ".nnhhbbbbbhhnn.",
    ".nnhbbcccbbhnn.",
    "nnhhbcccccbhhnn",
    "nnhhbcccccbhhnn",
    "nnhhbcccccbhhnn",
    ".nnhbbcccbbhnn.",
    ".nnhhbbbbbhhnn.",
    "..nnhhhhhhhnn..",
    "...nnnhhhnnn...",
    "....nnnnnnn....",
    "......nnn......"
  ]
}
```

- 凡例：`.` 空、`n` normal、`h` hard（耐久2）、`H` hard（耐久3）、`b` bomb、`c` core、`r` regen、`m` mirror、`+` heart
- layout は `size` 行 × `size` 文字。惑星半径外のマスに文字があっても無視する。
- `layout` の代わりに `"generator": {"type": "rings", ...}` を書けば手続き生成（M4）。

### 8-2. 強化定義 `data/upgrades.json`

```json
[
  { "id": "shard_plus", "name": "破片増量", "rarity": "common", "max_stack": 5,
    "desc": "全ブロックの破片数 +1",
    "effects": [ { "stat": "shard_bonus_count", "op": "add", "value": 1 } ] }
]
```

- 強化の効果はすべて `RunStats`（GameState が持つ辞書）への加算・乗算で表す。特殊な挙動（分裂など）は stat のフラグ・確率値をゲーム側が参照する。
- `op`：`add` / `mul` / `set`。

### 8-3. RunStats の初期値

```gdscript
{
  "lives": 3, "balls_per_stage": 1,
  "paddle_width_bonus_deg": 0.0,
  "shard_bonus_count": 0, "shard_life_mul": 1.0,
  "chain_bonus_depth": 0, "point_cap": 9999,
  "pierce_count": 0, "split_chance": 0.0,
  "shard_immune_per_stage": 0, "slow_chain": false
}
```

## 9. セーブ（M4）

- `user://save.json`：ハイスコア（エンドレス）、スターダスト、永続強化、設定。
- 書き込みは一時ファイル → リネームで破損を防ぐ。

## 10. 性能目標

- 中価格帯 Android 実機で 60fps。破片 3,000 個同時でも 16ms 以内。
- 破片の更新はループ内で Packed 配列に直接アクセスし、Dictionary や Object を生成しない。
- 計測用に F3 でデバッグ表示（FPS、破片数、ボール数、最大段数）。

## 11. テスト方針（GUT）

最低限、次をテストする。

| 対象 | テスト内容 |
|---|---|
| ChainScore | depth=-1 → 1、0 → 2、3 → 16、上限で 9999、bonus_depth の効果、大きい depth でオーバーフローしない |
| Collision.circle_vs_aabb | 正面・角・非接触・めり込み深さ |
| Collision.angle_diff | 境界（PI 付近）の正規化 |
| Collision.paddle_bounce_dir | 中心で真内向き、端で ±最大角 |
| BlockGrid | layout 読み込み、valid 判定、damage と破壊、MIRROR は壊れない、層判定 |
| ShardPool | spawn 上限、kill の swap-remove で整合性維持 |
| StunController | 必要連打数の増加と上限、リセット、タイムアウト |
| 再生 | 半径外や埋まったマスに増えない |
| SeededRng | 同シードで同じ結果 |
