# ステップ4: HSRLangRobot実装 + nav2結合テスト 進捗メモ

strategy.mdステップ4詳細化。実装済み・実機検証未実施分の記録。

**※2026-10-22注記（HSR時代記録）**: `HSRLangRobot`設計・`get_forward_pos`符号修正・`xy_goal_tolerance`知見はロボット非依存で有効。Stage1〜3結果・recovery/着地ズレ・`room1.yaml`使用はHSR上。sobit_edu（差動二輪）でStage1〜3再検証要: `base_frame:=sobit_edu/base_footprint`、nav2は`robot_name:=sobit_edu`＋`params_file:=.../sobit_edu/navigation_config.yaml`、地図は`sobits_slam`で新規作成。strategy.md「sobit_edu移行の要対応事項」節。

**現状（2026-09-28注記）**: Stage1〜3実機検証完了（strategy.md B4）。本ファイル前半（2026-09-18時点）の「未実施」記述は当時のもの。Stage2結果の詳細はdocs/resolved.md R9（旧strategy.md「実施記録」節「Stage2実施・結果」、2026-09-30移設）のみに記載（本ファイル未収録）。

## 実装状況（2026-09-18時点）

`avlmaps/robot/hsr_lang_robot.py`新規実装済み。**Python構文チェック（`py_compile`）のみ実施・PASS、実機動作確認は未実施。**（→2026-09-22〜23 Stage1〜3実施・PASS、下記各節）

### クラス構成

`HSRLangRobot(LangRobot, Node)`: `LangRobot`抽象基底（`avlmaps/robot/lang_robot.py`）と`rclpy.node.Node`の多重継承。`Node.__init__`・`LangRobot.__init__`を`__init__`内で個別に呼び分け。

### 各メソッドの設計

- **`_set_nav_curr_pose`**: `tf_buffer.lookup_transform("map", "base_footprint", Time())`で実機TF取得 → quat込み4x4同次変換行列化（`scipy.spatial.transform.Rotation`使用）→ `vlmaps_dataloader.from_habitat_tf()`にそのまま渡す。dataloader側（`habitat_dataloader.py`）は無改修、strategy.md記載の「軽微改修」方針通り、`from_habitat_tf`は実機TF由来の4x4行列を渡すだけで動く設計を踏襲。

- **`move_to(pos)`**: nav2 `navigate_to_pose`アクション1発呼出に置換（不採用済みのvisibility graph navigator/discrete controllerは使わない、resolved.md R19参照）。
  1. `_set_nav_curr_pose()`で現在位置取得
  2. 現在位置→目標位置(row,col)の方向を`grid_id2base_pos_3d`でx,y座標に変換し`arctan2`で算出（目標到達時の向き。nav2側が経路計画・障害物回避を担うため、waypoint単位の角度計画は不要）
  3. `vlmaps_dataloader.from_full_map_pose(target_row, target_col, theta_deg)` → `to_habitat_tf()`で目標tf算出（`from_habitat_tf`の逆演算、`habitat_dataloader.py`既存ロジックそのまま）
  4. `NavigateToPose.Goal()`にPoseStamped設定、`ActionClient`で送信・完了待機

- **`turn(angle_deg)`**: nav2 `Spin`アクション（相対yaw、度→ラジアン変換）。`behavior_server`に`spin`プラグイン（`nav2_behaviors::Spin`）標準搭載確認済み（sobits_nav `hsrb_robot/navigation_config.yaml`）。

- **`_send_action_goal`**: `navigate_to_pose`/`spin`共通のgoal送信・結果待ちヘルパー。`wait_for_server`→`send_goal_async`→`rclpy.spin_until_future_complete`で同期化→`GoalStatus.STATUS_SUCCEEDED`判定。

- **`setup_map`**: `HabitatLanguageRobot.setup_map`から3d分岐（`init_categories`/`global_pc`）を除いた版。VLMap/AreaMap（2d、Object/Area Localization）のみ対象のため不要と判断。※2026-09-23訂正: 実際にロードするのはVLMap（`load_scene_map`）＋dataloaderのみ、AreaMap・SoundMapロード経路なし（strategy.md B5・B6）。

### nav2アクション名前空間

sobits_nav側`nav2.launch.py`の`use_namespace`デフォルト`false`・`namespace`デフォルト空文字を確認済み（`nav2.launch.py:150-157`）→ 名前空間なし、`/navigate_to_pose`・`/spin`そのまま使用可能と判断。

## 未実施（実機必須のため）→ 全3点完了（2026-09-23、Stage1〜3）

- nav2（`sobits_nav`）起動状態での`navigate_to_pose`/`spin`実goal応答確認
- `_set_nav_curr_pose`が返すpose精度検証。TFテスト（docs/strategy.md「次セッション着手事項」節（※2026-09-28注記: 同節廃止、現在はdocs/resolved.md R6（旧strategy.md「実施記録」節「TFテスト」））、2026-09-16完了）は`avlmaps/dataset/`収集済みposes.txtでの往復変換ロジック検証のみで、実TF値と実際のmap座標が正しく対応するかは別途要確認
- 座標変換パイプライン通し動作確認、strategy.md記載の「まず単純直線移動で確認」（`navigate_to_pose`単発呼出）

## 未着手・要判断

- `main()`相当の起動スクリプト未作成。`habitat_lang_robot.py`の`main()`はhydra.main＋`test_config.yaml`前提だが、`test_config.yaml`自体はTFテスト時点（docs/step2.md）で「データ構造不一致のため新規作成見送り」と決定済み。同じ理由でHSRLangRobot側も`main()`は作らず、実機テスト時はstep3/TFテストと同様scratchpadスクリプト方式（`hydra.compose`直呼び）で対応する想定 → 要ユーザー確認（※2026-09-28注記: Stage2・3はscratchpad方式で実施済み。正式起動スクリプト無しはD区分＝問題だが優先度低、として扱う（2026-09-23ユーザー判断、CLAUDE.md・archive.md A9（旧temporary.md）））
- `move_to`の目標角度（現在位置→目標位置ベクトルの向き）が実際のnav2挙動として妥当か、実機での見え方次第で再検討の余地あり

## 2026-09-22 Stage1実施・Stage2着手前ブロッカー発覚

実機・nav2起動済み環境で、本ファイル「未実施」節3点を順次検証開始。Stage分け: Stage1=nav2 action server疎通確認（コード非依存）、Stage2=`_set_nav_curr_pose`精度検証、Stage3=`move_to`直線移動。

### Stage1（nav2 action server疎通確認）✅完了

事前準備、`CYCLONEDDS_URI`プレースホルダー問題再発（strategy.md記載の既知問題）→ `export CYCLONEDDS_URI="file://$CYCLONEDDS_URI_PATH"`で解消、毎回要る一時回避。

**`map`フレーム欠落発覚**: `tf2_echo map base_footprint`実行→"frame does not exist"。`odom→base_footprint`は正常配信（値ほぼ原点・静止中）確認 → **AMCL初期pose未設定が原因と判明**（map→odom未配信、particle filter未初期化状態）。RVizで「2D Pose Estimate」設定後、`map→base_footprint`正常配信確認。現在位置 x=-0.072, y=0.358, yaw=-33.48°。

参考: docs/log.md記載`local_costmap`警告（sensor origin z=0.98がbounds上限0.78m超、raytrace不可）確認済み。2Dナビゲーション自体への直接影響薄いと判断、記録のみ・保留。

`ros2 action send_goal /navigate_to_pose` CLI直呼びで前方0.5m移動goal投入（x=0.428, y=0.358、orientation維持）:
- goal accepted、`error_code: 0`、`Goal finished with status: SUCCEEDED`
- ただし`number_of_recoveries: 1`発生、最終位置(0.346, 0.477)で目標(0.428, 0.358)からy方向+0.12mずれ（直線でなく斜行）
- 原因未特定（上記local_costmap警告との関連含め要調査、Stage3以降で再確認要）

→ nav2 action server自体の疎通・基本応答は確認済み。挙動精度・recovery原因は保留課題として残る。

### Stage2着手前ブロッカー: Python環境二重化問題（新規判明）

`HSRLangRobot`実行には**rclpy（TF/nav2）とhydra/omegaconf/open3d/clip等（VLMap/dataloader）が同時に要る**が、どちらの既存環境も片方しか持たないと判明。strategy.md/step4.md旧記載になし、今回初めて表面化。

確認結果:
- `avlmaps` conda env（`/home/yura/miniforge3/envs/avlmaps`、Python3.8.20）: hydra等OK、`rclpy` `ModuleNotFoundError`
- システム`/usr/bin/python3`（3.12.3、ROS2 Jazzy）: `rclpy` OK、`hydra` `ModuleNotFoundError`
- `/opt/pytorch/.venv`（Python3.12、strategy.md既知のPYTHONPATH汚染要因の出処）: **`rclpy` OK・`torch`(2.9.0+cu128) OKと判明**、ただし`hydra`/`omegaconf`/`open3d`/`clip`は未インストール

→ 対応案: avlmaps側（Python3.8、rclpyビルド不可能に近い）に寄せるのではなく、`/opt/pytorch/.venv`に不足パッケージ（`hydra-core` `omegaconf` `open3d` `shapely` `pyvisgraph` `openai` `pandas` `clip`(openai/CLIP)、step3.md記載の不足15件相当）を追加インストールする方針が現実的。既存`rclpy`/`torch`には触れない、追加のみ。

**ユーザー判断（2026-09-22）**: 影響大きい変更のため一旦保留、検証してから実施と決定。**Stage2・Stage3は未着手のまま次セッション持越し**。

## 2026-09-22 Stage3着手・座標変換バグ発見修正・nav2 planner不調発覚

環境ブロッカー解消済み（a案install完了、SoundMap封印解除済み）前提Stage3着手。保留課題（Stage1 recovery原因・地図原点一致未検証）放置、仮動作確認として続行。

### 事前確認: AMCL状態・map→base_footprint TF

前回セッション終了時AMCL初期pose未設定（`map`フレーム欠落）。ユーザー側確認済み→`tf2_echo map base_footprint`配信確認、x=-0.029 y=0.628 yaw=-2.6°、Stage2実測値と同オーダーで整合。

### get_forward_pos()符号バグ発見・修正

scratchpadスクリプト（`HSRLangRobot.move_forward(0.3m)`実行）でStage3検証開始。1回目、navigate_to_poseはSUCCEEDED返すが実機前進せず別方向回転のみ（ユーザー目視確認）。

数値検証: `get_forward_pos`（[map.py:210](../avlmaps/map/map.py#L210)）計算の進行方向、現在yaw（`base_rot_mat2theta`のROS標準規約、x前方・反時計回り正）と逆符号。原因: `new_j = j + pix*sin(rad)`のcol方向符号がHabitat時代の時計回り正規約のまま残存。`get_delta_angle_to`コメント（"upward zero, turn right positive"）から逆算確認。

**修正**（[map.py:220](../avlmaps/map/map.py#L220)）: `new_j = j - pix*sin(rad)`に符号反転。修正後、進行方向計算値が現在yawとほぼ一致（theta_deg=-1.77° vs 現在yaw=-1.77°）、方向計算バグ解消確認。

**未検証の同種疑い残**: `get_north_pos`/`get_south_pos`/`get_east_pos`/`get_west_pos`等、同じ`curr_angle_deg`規約使う他関数群（[map.py:455-490](../avlmaps/map/map.py#L455-L490)）今回未検証。VLMap/AreaMapオブジェクトナビゲーション（`move_north`等、`lang_robot.py`経由）で使用、本実験着手前に要検証。※2026-09-28注記: 2026-09-23ユーザー判断でB区分（既知バグ）＝明らかにこれが原因とわかる問題発生まで扱わない。

### nav2 planner不調発覚（座標変換ロジックとは別問題）

符号修正後も実機前進せず。実測map座標TF（x=-0.024, y=0.681, yaw=-17.0°）と送信目標tf回転成分逆算絶対yaw（-17.05°）ほぼ一致→**ロボット目標向きに回転しただけ、xy位置不変**なのに`navigate_to_pose`はSUCCEEDED返す。

`/cmd_vel`にTwistメッセージなし、`/cmd_vel_nav`にほぼゼロ速度（linear.x≈-0.007, angular.z≈0.003）1回のみ。`/rosout`確認→`bt_navigator`ログ「Begin navigating from current location (-0.02, 0.68) to (0.28, 0.56)」正常、`controller_server`ログ「Received a goal, begin computing control effort.」直後「Reached the goal!」。

CLI直呼び（`ros2 action send_goal /navigate_to_pose --feedback`）同座標再送、feedback確認→**`distance_remaining`開始時点から0.0〜0.02m**（距離0.33m目標なのに）。current_poseは正しく実位置反映。**global planner、目標への経路をそもそも生成できていない**と判明。

**近距離0.1m goalでも同様切り分けテスト実施**（現在位置(-0.024,0.681)から前方0.1m、x=0.0718,y=0.6524）→同じく`distance_remaining=0.0`で即SUCCEEDED。**目標座標固有の問題（地図上障害物・未探索領域）でなく、planner_server側の一般的不調**と確定。Stage1（前回セッション、前方0.5m移動）は一応動いた実績あり→その後の状態変化（costmap不調・AMCL収束状態・時間経過等）疑われるが未特定。※2026-09-30注記: 0.1m切り分けテストは`xy_goal_tolerance=0.15m`圏内（2026-09-23判明、下記節・resolved.md R10）→「一般的不調と確定」の根拠として無効。0.33m/0.28m goal不調自体は有効、nav2再起動で解消。

**結論**: `hsr_lang_robot.py`/`map.py`側座標変換ロジックは今回不動作の原因でない（符号バグ修正後は容疑から外れた）。原因はnav2システム側（costmap/planner/AMCL）。

**ユーザー判断（2026-09-22）**: 本日はここで区切り、nav2側切り分け（global_costmap状態確認・AMCL再初期化・nav2スタック再起動等）次セッション持ち越し。

### 次アクション（次セッション）

- nav2 planner不調の原因特定: `/global_costmap/costmap`確認、AMCL粒子収束状態確認、または`sobits_nav`再起動からの再検証
- `get_north/south/east/west_pos`等、`get_forward_pos`と同じ角度規約使う他関数の符号検証（本実験で使うオブジェクトナビゲーション系、放置するとStage3と同じ不整合抱えたまま進む恐れ）
- nav2不調解消後、Stage3（`move_forward`実機前進確認）再実行

## 2026-09-23 nav2再起動後再検証・原因確定

nav2再起動実施。事前確認: `map`→`base_footprint`TF正常配信（x≈0.001, y≈0.002, yaw≈-0.8°、AMCL初期pose設定済み維持）、`global_costmap`・`planner_server`ともlifecycle state `active`確認。

前方0.1m goal送信（絶対座標x=0.1, y=0.002）→`distance_remaining`0.1057固定、navigation_time**60ms**でSUCCEEDED、current_pose不変。一見前回不調と同症状に見えたが、調査で原因判明。

**発見**: `/controller_server`の`general_goal_checker.xy_goal_tolerance = 0.15m`。0.1m goalの直線距離0.1057m < tolerance 0.15m → 現在位置が最初からゴール圏内、動かず即到達判定は**正常挙動**。前回不調（本ファイル上記「nav2 planner不調発覚」節、0.33m/0.28m goalで即SUCCEEDED）とは別現象、バグでなくテスト距離設計の問題だったと判明。

前方0.5m goal（絶対座標x=0.5, y=0.001）で再テスト→実際に前進確認:
- 着地x=0.362, y=0.029（目標x=0.5, y=0.001）
- recovery 1回発生、navigation_time=12.35秒
- 着地-目標間Euclidean距離≈0.14m → tolerance(0.15m)圏内でSUCCEEDED、妥当

**結論**: nav2側planner/controller不調、再起動で解消確認。**今後goal距離はxy_goal_tolerance(0.15m)より十分大きく取ること**（0.15m未満のgoalは「動かず正常終了」と「不調で動かない」が見分けられないため、テストに使えない）。

**継続課題（未解消）**:
- recovery 1回発生（前回Stage1・今回とも共通発生、原因未特定）
- 着地精度ズレ（今回y方向+0.028m級、前回Stage1もy方向+0.12mズレ記録あり）、原因未特定

### 次アクション（次セッション）

- recovery発生原因調査
- 着地精度ズレ原因調査
- Stage3本番確認（`move_forward`経由の通し動作、tolerance知見反映）再実行 → 下記節で完了

※2026-09-28注記: recovery・着地ズレは2026-09-23ユーザー判断でC区分＝問題顕在化まで扱わない。

## 2026-09-23 Stage3本番確認・座標変換パイプライン通し動作確認完了

上記セッション継続。AMCL再起動後`map`フレーム再欠落発覚（`/amcl_pose`12秒待機でも0Hz、`tf2_echo`も"frame does not exist"）。**hsrb_mode/CYCLONEDDS_URI問題とは無関係と切り分け済み**: `ROS_DOMAIN_ID`・`CYCLONEDDS_URI`正常設定確認、`/tf`139Hz安定配信でDDS通信自体は健全、`/amcl`lifecycle=`active`（プロセス生存もpose未publishなだけ）→ 単純にAMCL初期pose未設定（RViz「2D Pose Estimate」未実施/リセットされた状態）と確定。

ユーザー側RVizで2D Pose Estimate再設定→`map`→`base_footprint`再配信確認（x=0.047, y=0.108, yaw=-29.6°、安定）。

### Stage3本番スクリプト実行

scratchpadスクリプト（`hydra.compose`直呼び、`HSRLangRobot`インスタンス化→`setup_map`→tf_buffer 5秒warmup→`_set_nav_curr_pose`→`move_forward(0.3)`→`_set_nav_curr_pose`）実行。実行環境: `/opt/pytorch/.venv`、`PYTHONPATH`から`/tmp/opencv_pkg_install`除外（strategy.md既知回避策）、`CYCLONEDDS_URI`export済み。

**結果**:
- 実行前grid座標: (row=508, col=504), angle=-14.63°
- `move_forward(0.3m)`→`navigate_to_pose`: **SUCCEEDED**（`recorded_actions_list=['navigate_to_pose']`）
- 実行後grid座標: (row=504, col=504), angle=-12.73°
- 実移動量: row方向-4cell（`cell_size=0.05m`→**0.2m相当**、要求0.3mより0.1m短い）

**要求距離と実測の差(0.1m)**: 前回発見の`xy_goal_tolerance=0.15m`知見と整合。目標から0.15m以内で「到達」判定されるため、実移動距離は要求距離より最大tolerance分短くなる仕様上の制約（前回0.5m goalテストで着地距離0.14m残っていた記録と同じ現象）。

**軽微なコード上の発見（未修正、指示待ち）**: [lang_robot.py:198-201](../avlmaps/robot/lang_robot.py#L198-L201) `move_forward`が`self.move_to(pos)`の戻り値を`return`していない→呼出元の`actions`返り値は常に`None`。ただし`move_to`内部で`recorded_actions_list`には正しく`'navigate_to_pose'`が追記されるため、実行結果の記録自体には影響なし。

**Stage3結論**: 座標変換パイプライン（grid座標→`habitat_tf`→`map`frame→`navigate_to_pose`）、実機で通し動作確認**完了**。strategy.md記載の「まず単純直線移動で確認」達成。

### 次アクション（次セッション）

※2026-09-28注記: 下記のうちrecovery・着地ズレ（C区分）、`move_forward`return漏れ・`get_north`等符号検証（B区分）は2026-09-23ユーザー判断で問題顕在化まで扱わない。ベースライン次アクションはstrategy.md B5〜B8。

- recovery発生原因調査（未着手のまま持ち越し）
- 着地精度ズレ原因調査（未着手のまま持ち越し）
- `move_forward`のreturn漏れ修正判断（軽微、指示待ち）
- `get_north/south/east/west_pos`等の符号検証（未着手、本ファイル上記2026-09-22節記載の残課題）
- ~~VLMap/AreaMap（Object/Area Localization）本実験への着手判断~~ → 2026-09-23再整理: ベースライン残タスクはstrategy.md B5（AreaMap構築）・B6（SoundMap実機接続）・B7（VisualMap判断）・B8（`AVLMap`統合推論）。ODAS統合はstudy側タスクでAVLMaps本体スコープ外
