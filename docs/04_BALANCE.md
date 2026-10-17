# 04_BALANCE — 調整値の説明

実際の値は `data/balance.json`。ここは各値の意味と調整の目安。角度は度、時間は秒、距離はピクセル。

| キー | 初期値 | 意味 | 調整の目安 |
|---|---|---|---|
| arena.center | [360, 560] | フィールド中心 | HUD と回転ボタンが重ならない位置に |
| arena.orbit_radius | 300 | パドル軌道の半径 | 画面幅の約 0.42 |
| arena.lost_margin | 24 | 軌道の外、ボール喪失までの余白 | |
| planet.radius | 210 | 惑星の半径 | 軌道との差（リング幅）が狭いほど難しい |
| planet.cell | 28 | 1マスの大きさ | radius×2÷cell が奇数になるように（15マス） |
| planet.rotate_speed_deg | 90 | 回転操作の速さ（度/秒） | 速すぎると狙いにくい |
| planet.layer_outer / layer_inner | 0.66 / 0.33 | 層の境目（半径比） | |
| paddle.width_deg | 34 | パドル幅 | 易しくするなら広く |
| paddle.max_deflect_deg | 60 | 端で打った時の最大傾き | |
| paddle.max_angular_speed_deg | 540 | パドルの最大角速度 | |
| ball.radius | 8 | ボール半径 | |
| ball.speed | 520 | 初速 | 手触りの最重要値 |
| ball.speed_up_per_hit | 0.02 | パドルで打つたびの加速率 | |
| ball.max_speed | 900 | 最高速 | |
| ball.max_step | 6 | 衝突計算のサブステップ距離 | 小さいほど正確・重い |
| ball.max_balls | 8 | 同時ボール上限 | |
| blocks.*.hp / shards | 表参照 | 耐久と破片数 | 破片数が連鎖の起こりやすさを決める |
| shard.speed | 420 | 破片の速さ | |
| shard.life | 0.9 | 破片の寿命 | 速さ×寿命＝射程（約378px） |
| shard.max_active | 3000 | 同時破片上限 | 性能に合わせて |
| chain.point_cap | 9999 | 1回の破壊の得点上限 | |
| chain.heart_spawn_depth | 6 | ハートが出る連鎖段数 | |
| chain.hitstop_depth / hitstop_ms | 8 / 40 | ヒットストップ | |
| stun.base_taps / taps_increment / max_taps | 5 / 1 / 15 | 復帰に必要な連打数 | |
| stun.timeout_sec | 3.0 | 復帰の制限時間 | |
| regen.interval | 2.0 | 再生ブロックの増殖間隔 | 溜めの駆け引きの強さ |
| regen.max_spawn_per_tick | 3 | 1回に増える最大数 | |
| run.lives | 3 | 初期残機 | |
| run.rarity_weights | 70/25/5 | 強化のレア度の重み | |
| input.drag_sensitivity_deg_per_px | 0.45 | スマホのドラッグ感度 | 設定画面で変更可 |
| fx.* | | 画面揺れ・音階 | |
