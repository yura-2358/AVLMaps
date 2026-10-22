# 解決済み問題まとめ（resolved）

**※2026-10-22注記**: R1・R2（RGB-D側）・R4・R6〜R10の実機結果はHSR上（HSR時代記録）。ロボットはsobit_eduへ変更（decisions.md決定12）、該当項目はsobit_eduで再取得要（strategy.md「現在地」）。R3・R5（ロジック）・R11・R12はロボット非依存で有効。R13〜R21は2026-10-22 strategy.md縮小時に移設（R14・R20はHSR上の結果、R15〜R17・R19はロボット非依存で有効）。本文中「docs/log.md」参照先のHSRログは消失（log.md冒頭注記）。

**本ファイル = 解決済み問題・完了タスクの記録置き場。** `strategy.md`肥大化→確認困難のため、解決済み節をここへ移設（2026-09-30）。現在進行中・未解決の課題は`strategy.md`が一次情報、ここには置かない。

## 運用ルール

- 対象: 解決・完了確認済みの問題／タスク（実施記録・検証結果・解消経緯）
- 原文を転記。再検証で判明した誤りのみ本文訂正、訂正箇所に「※2026-09-30訂正: 旧記述○○」注記併記（旧値を消さない）
- 1エントリ = 1まとまり。記載項目: 出典（移設元ファイル・節・移設時点行番号）／再検証（2026-09-30時点の現物確認結果）／本文
- 本文中の「下記『実施記録』節」「下記節」等、`strategy.md`内相対参照 → 本ファイル内の該当エントリ（R番号）を指す
- 解決済みエントリ内に残る未解決の懸念（例: R9残留懸念）は`strategy.md`・`CLAUDE.md`側にも記載あり、そちらが正
- 新エントリは末尾追加（番号連番）
- `archive.md`との違い: archive = 誤り・スコープ外等で**外した**記述、resolved = **解決済み**で本流から外した記述

---

## 2026-09-30 再検証サマリ

移設前に全エントリの現物（コード・データ・環境）確認実施。実機（HSR・nav2・TAMAGO）要の項目は再実行不可→記録ベース。

### 記述どおり確認
- R3: `hsr_data_collector.py:51-56`（rmtree＋`"w"`）、`audio_recorder_node.py:47`（`"w"`）
- R4: rgb/depth/poses/timestamps 全122一致、frame_index 0〜121連番、x 0.50→1.27→0.35（最大ステップ0.124m）、52.09秒・2.32fps・先頭間隔0.051秒、poses1行目yaw −15.06°、rgb (480,640,3)
- R5: `config/map_config/hsr.yaml`存在、cam_calib = camera_intrinsics実測値
- R7: `vlmaps.h5df` 76.9MB（≒73MiB）存在
- R8: `HSRLangRobot`構成（`lookup_transform`・`navigate_to_pose`/`spin`・`setup_map`はVLMapのみ）
- R10: `map.py:220` `new_j = j - pix * np.sin(rad)`
- R11: `.venv`で rclpy・VLMap・SoundMap・VLMapMultiFloor・dataloader・HSRLangRobot 同一プロセスimport成功、torch 2.9.0+cu128 CUDA True。パッケージ版数 h5py 3.16.0 / timm 1.0.29 / hydra-core 1.3.1 / gdown 4.6.0 一致。avlmaps env（3.8.20）でVLMap import後`encoding`モジュール未ロード、torch-encoding未install
- R12: `__init__.py`・`lang_robot.py`両方SoundMap import有効、`audioclip/__init__.py:5` sys.path.insert存在。`.venv`でSoundMap instantiate＋AudioCLIPチェックポイントロード成功（30カテゴリ）。avlmaps env側はimportのみ再確認（instantiate未再実行）。追加6件＋torchaudio 2.4.1 両環境install済み

### 記述と相違
- R7: B3完了日「2026-09-14」、`vlmaps.h5df`のmtimeは**2026-09-11 16:07**。記録ミスor時計ずれ
- R4: depth範囲「0.0〜9.2m」、全122フレーム実測**0.0〜10.432m**（妥当範囲内、数値のみ誤り）
- R11「AreaMapの`.venv`上import・ロードは未検証」: `from avlmaps.map.area_map import AreaMap`直接importは`.venv`で**成功**。ロード・構築は未検証のまま

### 再確認不可（実機要）
- R2 TAMAGO USBパススルー: 2026-09-30時点`lsusb`/`arecord -l`にTAMAGO無し。未接続か再発か判別不能
- R6 TFテスト・R9 Stage2・R10 Stage3・R8 Stage1: 記録ベース、再実行なし

### 付随発見（未解決側、strategy.md B6/B8関連）
- `config/map_indexing_cfg.yaml`が存在しない`data_paths/lab_new`参照→そのままcompose時`MissingConfigException`。`data_paths=default`上書きで回避可

---

## R1. B0 HSR ROS環境構築・接続確認

- 出典: `strategy.md`「ベースライン実装順序」0.、21-23行
- 再検証: 実機要、未再確認

0. **HSR ROS環境構築・接続確認** ✅完了
   - 公式ROS2パッケージ`hsrb_library`導入・Dockerコンテナ組込 済
   - 実機最小接続確認 済

## R2. B1 データ収集ノード（RGB-D側）実機検証・TAMAGOパススルー解消

- 出典: `strategy.md`「ベースライン実装順序」1.、29-31行
- 再検証: TAMAGOパススルーは上記サマリ参照（現時点未検出）

   - ✅実機検証済み（2026-09-07、詳細docs/log.md）: `hsr_data_collector.py`をnav2起動済み環境下で15秒実行、30フレーム保存（起動直後2フレームdrop、以降drop無し）。rgb/depth/poses.txt/timestamps.txt/camera_intrinsics.txt全て整合確認（depth値0.0〜10.486m妥当範囲、camera_intrinsics fx=535.60 fy=535.90 cx=313.44 cy=238.39）
   - ✅往復動作実データ収集（`avlmaps/dataset/`、122フレーム）: 複数回起動によるposes.txt不整合バグ発見→修正後再収集完了（下記「実施記録」節）
   - ✅解消（2026-09-05）: コンテナ内TAMAGO USBパススルー可否 → コンテナ再起動で解消確認（詳細docs/step1.md）。TAMAGOビット深度、8ch/16kHz/S24_3LE固定と実機確定

## R3. 発見・修正したバグ（出力ファイル追記モード、2026-09-07）

- 出典: `strategy.md`「実施記録」節、93-99行
- 再検証: 修正コード現存確認済み

`hsr_data_collector.py`・`audio_recorder_node.py`とも、出力ファイル（`poses.txt`/`timestamps.txt`/`audio_timestamps.txt`）を`open(path, "a")`（追記モード）で開いていた。rgb/depth画像は`cv2.imwrite`/`np.save`でフレーム番号ファイル名（`000000.png`等）に上書き保存されるため、**同一`output_dir`に対して複数回起動すると、画像は最後の起動分だけ残るがposes.txt/timestamps.txtは全起動分が積み上がり、行数不一致で対応関係が壊れる**。

実際に`avlmaps/dataset/`で発生（往復動作のため複数回起動→poses.txt 181行 vs rgb/depth/timestamps.txt 98）。`vlmap_builder.py`は`zip(rgb_paths, depth_paths, base_poses)`（vlmap_builder.py:102）で先頭からペアリングする設計のため、気づかず地図構築すると誤ったpose割当のまま進む致命的リスクだった。

**修正済み**（2026-09-07、両ファイルとも）: `poses.txt`/`timestamps.txt`/`audio_timestamps.txt`を`"w"`（上書き）モードに変更、`hsr_data_collector.py`はさらに起動時に`rgb/`・`depth/`ディレクトリを`shutil.rmtree`で削除後再作成するよう変更。これで1起動=1収集セットが保証される（複数回起動しても最後の1回分のみ残る）。

## R4. `avlmaps/dataset/`再収集 ✅完了・データ正常確認済み（2026-09-07）

- 出典: `strategy.md`「実施記録」節、89-91行
- 再検証: depth範囲のみ相違→本文訂正済み、他一致

バグ修正後コードで再収集、往復動作122フレーム。rgb/depth/poses.txt/timestamps.txt**全て122で完全一致**、frame_index重複なし（0〜121連番）。`poses.txt`軌跡はx=0.50→1.26→0.35と滑らかな往復（不連続ジャンプなし）、depth値0.0〜10.432m妥当範囲（※2026-09-30訂正: 旧記述「0.0〜9.2m」、全122フレーム実測で修正）、rgb (480,640,3)正常。

## R5. B2 dataloader改修

- 出典: `strategy.md`「ベースライン実装順序」2.、35-37・39行（38行「水平オフセット無視」運用決定はstrategy.md側に残置）
- 再検証: `hsr.yaml`現存確認済み

2. **dataloader改修** ✅完了（2026-09-05、詳細docs/step2.md）
   - 調査結果: poses.txt・座標変換ロジックとも無改修で動作（実機TF・ROS座標系がHabitat側実装と整合済み）
   - `config/map_config/hsr.yaml`新規作成済み（URDF値からbase2cam_tf算出）
   - ✅TFテスト（往復変換誤差）完了（2026-09-16）: 10/10 PASS（下記「実施記録」節）

## R6. TFテスト ✅完了（2026-09-16）

- 出典: `strategy.md`「実施記録」節、101-105行
- 再検証: 未再実行

`avlmaps/dataset/`（往復122フレーム）からランダム10フレーム、往復変換誤差検証。10/10 PASS（誤差0.02〜0.05、`cell_size`グリッド量子化起因、想定内）。座標変換ロジック バグなし確認。リポジトリ無変更。

`main()`不採用の理由: `habitat_dataloader.py`の`main()`（hydra.main、`config/test_config.yaml`前提）は`avlmaps_data_dir/vlmaps_dataset/<シーン>/`という複数シーンネスト構造前提。実データ`avlmaps/dataset/`はフラット構造（1シーン直置き）で不一致、かつ`avlmaps_data_dir`自体も旧ダミー値未修正。→ step3実行済みスクリプトと同方式（`hydra.compose`直呼び）で`main()`本体の往復変換ロジックのみ移植実行、`test_config.yaml`新規作成は見送り。詳細docs/step2.md「2026-09-16 TFテスト実施・結果」節参照。

## R7. B3 VLMap構築（vlmap_builder動作確認）

- 出典: `strategy.md`「ベースライン実装順序」3.、41-42行
- 再検証: 生成物存在確認済み（76.9MB）。完了日相違→本文訂正済み

3. **VLMap構築（vlmap_builder動作確認）** ✅完了（2026-09-11、※2026-09-30訂正: 旧記述「2026-09-14」。`vlmaps.h5df`のmtime 2026-09-11 16:07基準。step3.md「✅実行完了（2026-09-14）」は記録日の可能性、実行日は未確定）
   - 実機データ（`avlmaps/dataset/`往復122フレーム）で**VLMapのみ**構築、`avlmaps/dataset/vlmap/vlmaps.h5df`（約73MB）生成済み。GPU（RTX 3070）約65秒。詳細step3.md

## R8. B4 HSRLangRobot実装 + nav2結合テスト（Stage1〜3）

- 出典: `strategy.md`「ベースライン実装順序」4.、45-52行（53行「正式起動スクリプト未作成」はstrategy.md側に残置）
- 再検証: コード構成確認済み、実機Stageは未再実行

4. **HSRLangRobot実装 + nav2結合テスト** ✅完了（Stage1〜3、2026-09-23）
   - nav2（`sobits_nav` `nav2.launch.py`、map_server+AMCL+planner/controller）起動、`map`→`base_footprint`TF配信確認済み（2026-09-07、詳細docs/log.md）。使用地図はsobits_navデフォルト`room1.yaml`（実環境との対応は未確認のまま）
   - `avlmaps/robot/hsr_lang_robot.py`実装済み（2026-09-18）。`HSRLangRobot(LangRobot, Node)`:
     - `_set_nav_curr_pose`: `tf_buffer.lookup_transform("map","base_footprint")`→4x4行列化→`vlmaps_dataloader.from_habitat_tf()`（dataloader無改修）
     - `move_to(pos)`: 現在位置→目標位置の方向角を`grid_id2base_pos_3d`で算出→`to_habitat_tf()`で目標tf算出→`NavigateToPose`アクション送信・完了待機
     - `turn(angle_deg)`: `Spin`アクション（相対yaw）。nav2アクション名前空間はsobits_nav側`use_namespace`デフォルト`false`確認済み→`/navigate_to_pose`・`/spin`そのまま使用
     - `setup_map`: Habitat版から3d分岐除いた版。**実際にロードするのはVLMap（`load_scene_map`）＋dataloaderのみ**、AreaMap・SoundMapロード経路なし（B5・B6で追加要）
   - Stage1（`navigate_to_pose`/`spin`実goal応答）✅ / Stage2（`_set_nav_curr_pose`精度）✅ / Stage3（`move_to`単純直線移動）✅。詳細下記「実施記録」節・docs/step4.md

## R9. Stage2実施・結果（2026-09-22）

- 出典: `strategy.md`「実施記録」節、107-120行
- 再検証: 未再実行。末尾「残留懸念」は未解決（問題顕在化まで扱わない、strategy.md「現在地」・CLAUDE.md B/C区分）

実機・nav2起動済み環境（AMCL初期pose設定済み、`map`→`base_footprint`TF配信正常）で`HSRLangRobot._set_nav_curr_pose()`をscratchpadスクリプトから直接呼出。

**重要な設計確認（当初誤解していた点）**: `from_habitat_tf`（`habitat_dataloader.py`）は入力TFをそのまま絶対座標変換するのではなく、`self.init_base_tf`（`poses.txt`**1行目**＝マップ収集走行開始時のTF）を基準とした**相対変換**を行う設計（`tf = inv_init_base_tf @ base_transform @ tf_hab @ inv(base_transform)`）。→ VLMapのgrid座標系原点は「マップ収集開始位置」であり、nav2の`map`座標系原点（SLAM/AMCL地図ファイル原点）とは別物。当初、実機TF実測値（map原点基準の絶対座標）と`_set_nav_curr_pose`の逆算結果を単純比較し「0.57m級の大ズレ」に見えて焦ったが、これは比較方法の誤り（絶対座標同士の比較は無意味な設計）と判明。

**正しい検証**: `poses.txt`1行目（収集開始位置、x=0.4951, y=0.2005, yaw=-15.07°）を基準に、実機TF実測値（x=-0.028, y=0.628, yaw=-4.70°）から相対変換を手計算（`inv_init_base_tf @ now_tf`相当）:
- 期待値: x=-0.616, y=0.277, yaw=10.36°
- `_set_nav_curr_pose`実測（scratchpadスクリプト）: x=-0.600, y=0.250, yaw=10.51°
- 誤差: dx=0.016m, dy=0.027m, dyaw=0.15° → `cell_size=0.05m`グリッド量子化起因の範囲内（TFテスト時2026-09-16の誤差水準0.02〜0.05と同オーダー）

→ **Stage2 PASS**。TF取得→4x4行列化→相対座標変換→grid座標(row,col)算出の一連ロジック、実機TFに対して正確に動作確認。tf_buffer初期化直後はlookup失敗（"frame does not exist"/"unconnected trees"）、5秒程度のspin_onceウォームアップで解消（tf2_echo同様の既知挙動）。

**残留懸念（未検証のまま持ち越し、問題顕在化まで扱わない）**: VLMap座標系原点＝マップ収集時のposes.txt開始位置。`move_to`が正しい絶対位置に移動するには、**その時と今でnav2側`map`座標系（使用地図ファイル・AMCL設定）が同一である**ことが前提。使用地図`room1.yaml`と実環境の対応関係が実機未確認のまま。`move_to`結果がズレた場合、`move_to`ロジック自体のバグか地図原点不一致かの切り分け要。

## R10. Stage3実施・結果（2026-09-23）

- 出典: `strategy.md`「実施記録」節、122-128行
- 再検証: `map.py:220`修正現存確認済み、実機未再実行。末尾recovery・着地ズレ・`get_north/...`未検証は未解決（B/C区分、strategy.md・CLAUDE.md側が正）

nav2再起動→`xy_goal_tolerance=0.15m`（`/controller_server`）発見。前回不調（0.33m/0.28m goal即SUCCEEDED、docs/step4.md記載）とは別に、0.1m goalテストは目標距離がtolerance圏内だったため「動かず正常終了」と判明（バグでない）。0.5m goalで再テスト→実際に前進確認（recovery1回・着地-目標間距離0.14m、tolerance圏内でSUCCEEDED）。**nav2側不調、再起動で解消確認**。今後goal距離は0.15mより十分大きく取ること。

Stage3本番（scratchpadスクリプト、`/opt/pytorch/.venv`）: `HSRLangRobot.setup_map`→`move_forward(0.3)`→`navigate_to_pose` SUCCEEDED。grid (508,504)→(504,504)、実移動0.2m相当（要求0.3mより0.1m短い＝`xy_goal_tolerance`仕様上の制約）。座標変換パイプライン（grid座標→`habitat_tf`→`map`frame→`navigate_to_pose`）実機通し動作確認完了。

**Stage3着手時のコード修正（2026-09-22）**: `map.py`の`get_forward_pos`、col方向符号がHabitat時代の時計回り正規約のまま残存→実機で前進せず別方向回転。`map.py:220` `new_j = j - pix*sin(rad)`に符号反転、修正後進行方向計算値と現在yaw一致確認。同規約の`get_north/south/east/west_pos`は未検証（既知バグ、B区分）。recovery発生・着地ズレ0.03〜0.14m級は継続課題、原因未特定。詳細docs/step4.md。

## R11. Python環境二重化問題・a案（`/opt/pytorch/.venv`一本化）

- 出典: `strategy.md`「環境状況（判明済み）」節169行＋「Python環境: a案決定経緯」節171-179行＋「a案install実施・実機検証結果」節181-211行
- 再検証: import・パッケージ版数確認済み。AreaMap直接importは`.venv`で成功→本文訂正済み（ロードは未検証）。PYTHONPATH恒久対応は未実施のまま（CLAUDE.md「環境構築・依存関係の罠」節が運用上の正）

- 🔴→🟢 Python環境二重化問題（2026-09-22判明・解消）: `HSRLangRobot`実行にはrclpy（TF/nav2）とhydra/omegaconf/open3d/clip等（VLMap/dataloader）が同時に要るが、既存2環境（`avlmaps` conda Python3.8／システムROS2 Python3.12）はどちらも片方しか持たない。第3の候補`/opt/pytorch/.venv`（Python3.12）へ追加install（a案）で解消。詳細docs/step4.md「2026-09-22 Stage1実施・Stage2着手前ブロッカー発覚」節・下記節参照

### Python環境: a案（`/opt/pytorch/.venv`一本化）決定経緯（2026-09-22）

rclpyノード（ナビゲーション制御・音声受信）とVLMapクエリを**同一プロセス・同時稼働**させる構成を採用。直接の契機は提案手法側（study）の音声モダリティオンライン逐次更新要件だが、`HSRLangRobot`（ベースライン側、rclpy＋VLMapクエリ）も同一プロセス構成を要するため、共通基盤としての判断。以下検証実施。
- **rclpy側制約確認**: `ros-jazzy-rclpy`（apt配布）のCバインディング`_rclpy_pybind11`、Python3.12 ABI固定ビルド（ROS2 Jazzy公式サポートOS=Ubuntu24.04・同梱Python=3.12のみ）。avlmaps conda env（Python3.8.20）で`import rclpy`実行→実証: `ModuleNotFoundError: No module named 'rclpy._rclpy_pybind11'`（`.cpython-38-...so`不在）。pipでの代替配布も無し（`pip index versions rclpy`→404）
- **avlmaps側Python3.8縛りの実体**: `setup.py`は`python_requires=">=3.8"`のみで上限無し、実制約は`requirements.txt`記載`torch-encoding==1.2.1`（2020年前後のCUDA C++拡張、新Python/新torchでのビルド実績不明）
- **重要判明**: VLMap**オンラインクエリ経路**（`vlmap.py`→`vlmap_builder.py`→`lseg_net.py`→`lseg_blocks.py`→`lseg_vit.py`）は**torch-encoding非依存**と実証済み。avlmaps env（python3.8）で`from avlmaps.map.vlmap import VLMap`単体import実行→成功、encoding系エラー一切なし。torch-encoding依存は`lseg_module.py`/`lsegmentation_module.py`（LSeg**学習**用PyTorch Lightningモジュール）限定、AVLMapsは学習済みLSegチェックポイントを`gdown`でDL・推論のみに使う設計のためオンライン経路には最初から不要
- オンライン経路の実依存: `clip`(openai/CLIP、pure python+torch) `timm`(pure python) `hydra-core` `omegaconf`(pure python) `open3d`(avlmaps env内確認バージョン0.19.0、python3.12公式wheel配布済み世代) `h5py` `gdown` `opencv-python` `scipy` `pandas`。いずれもpython3.12ビルド実績良好、地雷薄いと判断
- **決定**: `/opt/pytorch/.venv`（Python3.12、rclpy・torch既存）へ上記オンライン経路依存を追加install、rclpyとVLMapクエリ系を同一プロセスで同時稼働させる方針（既存rclpy/torchは変更しない、追加installのみ）。b案（rclpy Python3.8向けソースビルド）・c案（avlmapsとrclpyのプロセス分離＋IPC）は不採用。マップ**構築**（バッチ処理。※2026-09-28訂正: 旧記述「torch-encoding要る」は誤り、`vlmap_builder.py`は`lseg_net`のみimportでtorch-encoding非依存、avlmaps envにtorch-encoding未installのまま構築成功）は引き続き`avlmaps` conda env(python3.8)オフラインのまま、オンライン稼働側（実行時クエリ＋rclpy）のみ`/opt/pytorch/.venv`に一本化
- 注意: 検証対象はVLMap系のみ。AreaMapの`.venv`上ロードは未検証（B5）。※2026-09-30訂正: 旧記述「import・ロードは未検証」。`from avlmaps.map.area_map import AreaMap`直接importは`.venv`で成功確認

### a案install実施・実機検証結果（2026-09-22）

`/opt/pytorch/.venv`へ`sudo /opt/pytorch/.venv/bin/python -m pip install`で追加install実施（既存rclpy/torchは無変更）。

**install済みパッケージ**: `hydra-core==1.3.1` `omegaconf` `gdown==4.6.0` `h5py`（バージョン無指定） `timm`（バージョン無指定） `open3d` `opencv-python` `scipy` `pandas` `git+https://github.com/openai/CLIP.git` `shapely` `pyvisgraph` `openai`

**requirements.txt記載バージョンから外した2件（Python3.12非互換のため）**:
- `h5py==3.7.0`固定→ビルド失敗（Cython生成コードがPython3.12と非互換、`CompileError`）。バージョン無指定install（3.16.0取得）で解消
- `timm==0.6.12`固定→import時`ValueError: mutable default ... for field conv_cfg is not allowed`（Python3.11以降の`dataclasses`仕様変更でmutable defaultが弾かれるようになった影響、timm側が旧仕様前提のコード）。バージョン無指定install（1.0.29取得）で解消
- 判断: フェアネス原則（CLAUDE.md）はVLMap・AreaMapコアアルゴリズム（ヒートマップ生成・LSeg/CLIP特徴抽出ロジック）の共通化が対象であり、周辺ライブラリの具体バージョンは対象外。かつstudy側もAVLMapsをimportして使う一方向依存（CLAUDE.md記載）のため、この環境のバージョン差は両手法（ベースライン・提案手法）に等しく効く→問題なしと判断

**PYTHONPATH順序問題（新規判明）**: 既存PYTHONPATH内`/tmp/opencv_pkg_install/lib/python3.12/dist-packages`（ROS関連の古いopencv、numpy1.x向けビルド）が`/opt/pytorch/.venv/lib/python3.12/site-packages`より前にあり、そちらが優先ロードされてvenv側新opencv-pythonと衝突（`numpy.core.multiarray failed to import`、numpy2.x非互換エラー）。rclpy自体は`/opt/ros/jazzy/lib/python3.12/site-packages`がPYTHONPATH経由でのみ見える（`unset PYTHONPATH`だと`rclpy`自体importできなくなる、venvのinclude-system-site-packagesは無関係）ため`unset`は不可。→ 検証時は`/tmp/opencv_pkg_install`エントリを除いたPYTHONPATHを明示指定して回避。恒久対応（`.bashrc`等でのPYTHONPATH定義修正）は未実施（優先度低）

**実証結果**: 上記回避策込みで以下、同一プロセスimport成功確認
```
rclpy OK
VLMap OK
VLMapMultiFloor OK
habitat_dataloader（VLMapsDataloaderHabitat） OK
```
→ **a案の核心（rclpy + VLMapクエリ系の同一プロセス同時稼働）、実証完了**

**新規課題発覚**: `avlmaps/robot/lang_robot.py`（`LangRobot`抽象基底、`HSRLangRobot`の親クラス）8行目`from avlmaps.map.sound_map import SoundMap`が無条件トップレベルimport。`map/__init__.py`で対応済みの芋づる式import問題と**同種の問題が`lang_robot.py`側に別途存在**、`map/__init__.py`の絞り込みでは防げない経路。実際に`from avlmaps.robot.hsr_lang_robot import HSRLangRobot`実行→`ModuleNotFoundError: No module named 'wav2clip'`で停止（SoundMap→`avlmaps/utils/audio_utils.py`→`wav2clip`という連鎖）。

**一時対応（2026-09-22）**: `lang_robot.py`8行目`SoundMap`importをコメントアウト。実施後、`/opt/pytorch/.venv`で再検証:
```
rclpy OK
HSRLangRobot OK: import成功
torch: 2.9.0+cu128 cuda: True
```
→ rclpy + HSRLangRobot（VLMap一式含む）同一プロセスimport、GPU認識込みで成功。同日中に下記節の通り依存を揃えて封印解除済み。

## R12. SoundMap問題の解決・封印解除（2026-09-22）

- 出典: `strategy.md`「SoundMap問題の解決・封印解除（2026-09-22）」節、213-228行
- 再検証: 上記サマリ参照（`.venv`instantiate成功、avlmaps envはimportのみ再確認）。`noisereduce`両環境未installのまま（B6）

`map/__init__.py`・`lang_robot.py`両方でコメントアウトしていた`SoundMap`import、依存関係を実際に揃えて封印解除。**判断根拠（ユーザー指示）**: 「Dynamic Mapは後回し」方針は**提案手法（study側）の実装順序**の話であり、AVLMapsリポジトリ側は**ベースライン実装**（AVLMaps原論文のAudioCLIPベースSoundMap）を置く場所（CLAUDE.md「リポジトリ構成」節参照）。ベースライン側では音声機能を最初から使用可能にしておく方が正しい、との判断。

**依存連鎖の実態**: `SoundMap`→`audio_utils.py`（`wav2clip`）→`AudioCLIP`（`avlmaps/audioclip/`、リポジトリ同梱）→`ignite_trainer`（`avlmaps/audioclip/ignite_trainer/`、AudioCLIP付属の独自パッケージ、外部`pytorch-ignite`要）→`visdom`（可視化ツール、`ignite_trainer`経由で連鎖要求）。

**設計面の発見（step3.md記載の懸念は実は解消済みだった）**: `avlmaps/audioclip/__init__.py`自体が`sys.path.insert(0, parent_dir)`を内蔵しており、`avlmaps.audioclip`パッケージロード時に自動で`avlmaps/audioclip`ディレクトリがsys.pathに追加される。step3.md記載の「`model.clip`/`model.esresnet`絶対import、sys.path追加しないと解決不可」という懸念は、実際にはこの`__init__.py`内蔵の仕組みで自動解決される設計だった（追加のコード改修不要）。

**追加install（両環境、既存パッケージには触れず追加のみ）**: `wav2clip` `pytorch-ignite` `visdom` `termcolor` `simplejpeg` `ftfy`
- `/opt/pytorch/.venv`（python3.12、rclpy側）: 上記追加、問題なくinstall完了
- `avlmaps` conda env（python3.8、マップ構築側）: 同様に追加（`wav2clip`はdocs/step3.md記載の未インストールリスト該当、`pytorch-ignite`等残り5件はリスト外＝AudioCLIP経由の連鎖依存）。`torchaudio-2.4.1`が`wav2clip`の依存として新規追加された点のみ注意（既存`torch`/`torchvision`バージョンとは無変更で共存）
- 未install: `noisereduce`（実機用経路`create_audio_map_batch_real`で使用、B6）

**実証結果**: 両環境で`SoundMap`単体instantiate＋`AudioCLIP`モデルロード（チェックポイント`avlmaps/audioclip/assets/AudioCLIP-Partial-Training.pt`読込含む）まで成功確認。`/opt/pytorch/.venv`側は`rclpy + map全体(VLMap/SoundMap/VLMapMultiFloor) + HSRLangRobot`同時import成功も再確認済み。

**コード変更**: `avlmaps/map/__init__.py`（`SoundMap`importコメントアウト解除、`__all__`に追加）、`avlmaps/robot/lang_robot.py`（`SoundMap`importコメントアウト解除）。`VisualMap`（hloc/pycolmap依存）は従来通りコメントアウト維持。

## R13. sobit_edu移行経緯・実機接続確認（2026-10-22）

- 出典: `strategy.md`「ロボット変更: HSR→sobit_edu」節「経緯」「実機接続確認結果」、30-59行（2026-10-22移設）
- 備考: 接続確認完了。移行で残る要対応事項（HSR差分）は`strategy.md`「sobit_edu移行の要対応事項」が正

### 経緯
- 変更理由・構成・帰結: decisions.md決定12
- 使用パッケージ: `colcon_ws/src/sobit_edu`（bringup/description/control/library）、`sobits_navigation_stack`（`sobits_nav`・`sobits_slam`）。依存（kobuki_node・urg_node・openni2_camera・depth_image_proc・dynamixel_hardware・twist_stamper・topic_tools・nav2_bringup・slam_toolbox）全て解決済み
- READMEはHumble/22.04前提、Jazzy colcon_wsでビルド済み・実機起動確認済み（下記）

### 実機接続確認結果（2026-10-22）
- 起動: `ros2 launch sobit_edu_bringup real_minimal.launch.py`（READMEの`minimal.launch.py`は不在）
- 初回起動時の問題と対処:
  - シリアルポート逆指定: `DXL_SE_PORT=/dev/ttyUSB0`・`KOBUKI_SE_PORT=/dev/ttyUSB1`で起動→Dynamixel `[TxRxResult] Incorrect status packet!`・Kobuki `no data stream`。実デバイスはKobuki=ttyUSB0、Dynamixel（BestTechnology E143）=ttyUSB1。挿し直しで番号変動するため by-id指定推奨:
    ```bash
    export DXL_SE_PORT=/dev/serial/by-id/usb-BestTechnology_E143_E143-if00-port0
    export KOBUKI_SE_PORT=/dev/serial/by-id/usb-Yujin_Robot_iClebo_Kobuki_kobuki_AL03ERLV-if00-port0
    ```
  - カメラ: 搭載はASUS Xtion（`lsusb` `1d27:0601`）、Gemini 336でない。`real_minimal.launch.py`は`head_camera_name = "gemini_336"`ハードコード→`No Device found`。Xtion指定で解消
  - URG: `Could not open network Hokuyo: 172.16.10.10:10940`→有線IF未接続が原因、解消済み
  - 注: rviz（`real.rviz`）が`/sobit_edu/head_camera/depth_registered/points`等を購読→topic listに出るがpublisherでない。トピック存在だけで疎通判断しない
- 全接続後の実測（`topic hz` 15秒）:
  - `/sobit_edu/head_camera/rgb/image_raw`: 29.8Hz
  - `/sobit_edu/head_camera/depth_registered/image_raw`: 30.0Hz
  - `/sobit_edu/scan`: 40.0Hz
  - `/sobit_edu/odom`: 49.8Hz
  - `/sobit_edu/joint_states`: 99.7Hz（Kobuki車輪＋Dynamixel 10軸）、arm/head/handコントローラ起動確認
- 画像:
  - RGB: `rgb8`、640×480、frame_id `sobit_edu/head_camera_rgb_optical_frame`
  - Depth: **`32FC1`（m単位）**、640×480、RGB位置合わせ済み（同frame_id）。`hsr_data_collector.py` `_decode_depth_meters`の32FC1経路で対応可
  - `rgb/camera_info` K: fx=fy=570.342、cx=319.5、cy=239.5、歪み全0 → **OpenNI公称デフォルト値、キャリブレーション値でない**
- TF:
  - `sobit_edu/odom`→`sobit_edu/base_footprint`: 取得OK（kobuki_nodeが名前空間prefix付与）
  - `sobit_edu/base_footprint`→`sobit_edu/head_camera_rgb_optical_frame`: xyz=(0.130, 0.023, 0.941)、URDF計算値（pan=tilt=0: 0.1297, 0.023, 0.9406）と一致
  - `map`: nav2未起動のため不在（想定どおり）

## R14. B5 AreaMap構築・ロード経路（HSRデータ、2026-09-30）

- 出典: `strategy.md`「B5〜B8 必要作業」1.（102-106行）＋「ベースライン実装順序」5.（168-177行）（2026-10-22移設）
- 備考: HSRデータでの結果。sobit_eduデータで再構築要（ロード経路・`.venv`検証は流用可）。クエリ妥当性未評価

1. **B5 AreaMap** ✅完了（2026-09-30、詳細下記「ベースライン実装順序」5）
   - [x] `avlmaps/map/__init__.py`の`AreaMap`コメントアウト解除
   - [x] `avlmaps` conda envで`clip_sparse_map.h5df`構築（CLIP ViT-L/14初回DL約890MB）
   - [x] `/opt/pytorch/.venv`側ロード確認（VLMapと同一プロセス時GPUメモリ含む）
   - [x] `HSRLangRobot`にAreaMapロード経路追加

5. **AreaMap構築・ロード経路** ✅完了（HSRデータ、2026-09-30） 🔁sobit_eduデータで再構築要（ロード経路・`.venv`検証は流用可）
   - **実施結果（2026-09-30）**:
     - `__init__.py` `AreaMap`公開（`__all__`追加）
     - 構築: `avlmaps` conda env、`AreaMap().create_map(avlmaps/dataset)`→`avlmaps/dataset/area_map/clip_sparse_map.h5df`（122×768 float32、特徴ノルム≒1.0、robot_pose_list 122×4×4）。所要90秒（ViT-L/14 DL込み）
     - `.venv`検証: VLMap（hsr config、`hydra.compose`＋`map_config=hsr`）＋AreaMap同一プロセスロード成功。GPU: VLMap+CLIP ViT-B/32後343MiB→AreaMap+ViT-L/14後1241MiB、クエリ後peak 1252MiB（8GB内余裕）
     - テキストクエリ（`with_init_cat=False`）動作: kitchen/bedroom/office/corridor/living room、スコア0.13〜0.26。妥当性（正しいエリアを指すか）は未評価
     - 全122フレームpose→`from_habitat_tf`→`to_full_map_pose`、1000×1000グリッド内（rows 486..501、cols 488..511、グリッド外0件）→`index_area_2d`の座標経路整合
     - `HSRLangRobot.setup_map`へ`load_area_map`追加（`hsr_lang_robot.py`、`self.area_map`）。rclpy node生成込みオフライン`setup_map`実行PASS
     - 注: オフライン検証時、HSR有線IF（`enx9c69d3405f36`）未接続→`CYCLONEDDS_URI`（hsrb_libraryプロファイル）指定だとnode生成`RCLError`。`CYCLONEDDS_URI` unset＋別`ROS_DOMAIN_ID`で回避
   - 着手前記録（`__init__.py`コメントアウト・ロード経路なし等、上記実施結果で全解消）→ docs/archive.md A9「A-1」・docs/resolved.md「2026-09-30 再検証サマリ」

## R15. B6 SoundMap 着手前調査・実装要素整理（2026-09-23〜09-30）

- 出典: `strategy.md`「ベースライン実装順序」6.冒頭〜「B6着手前調査」、179-238行（2026-10-22移設）
- 備考: 大半は`create_audio_map_hsr`実装（R16）で解消。未解消の判断材料（マイク・ch、音カテゴリ、音源配置、`find_similar_category_id`、`application/`用データ置き場）は`strategy.md` B6・B8に要約。実機用経路の秒×fpsバグ疑いは新規関数で非経由→修正不要扱い

6. **SoundMap実機接続** 🔶コード実装・オフライン検証済み、実データ未収集（2026-09-30）。実データ収集はsobit_eduで実施（下記「HSR⇔PC時刻同期」懸念はsobit_eduで構造上解消見込み、「ロボット変更」節）
   - セグメンテーション方式: **既知区間（`time_ranges`）入力方式**（定義・根拠はdecisions.md決定11）→`silence_threshold`等Habitat向けパラメータ調整は対象外
   - 前提（B1から移管）:
     - `audio_recorder_node.py`実機起動テスト未実施 → 2026-10-20 TAMAGO単体PASS（下記「実機起動テスト結果」）
     - ビット深度不整合: TAMAGOはS24_3LE固定（ハード変更不可）、`audio_recorder_node.py`は`signed_16`のみ対応。`plughw`経由変換で吸収 or recorder側24bit対応、未決 → 2026-10-20 `plughw`＋`signed_16`で解消確認
   - 実装要素:
     - 変換層: 現行収集出力（`poses.txt`/`timestamps.txt`/`audio.wav`/`audio_timestamps.txt`）→実機用経路入力形式（`audio_video/<seq>/audio_video.mp4`・`pose/`・`rob2cam_tf.txt`）、または実機用経路相当処理を現行形式で直接実行。いずれもコード改修要
     - `time_ranges`取得方法（2026-09-30ユーザー判断）: **現在は案b（録音後の人手ラベル）で行う**
       - `time_ranges`＝シーケンス毎の音イベント区間`[start秒, end秒]`リスト（`create_audio_map_batch_real`必須引数）。区間切出し→AudioCLIP特徴化、同区間フレームのロボット位置→音源位置
       - 案a: 音源再生ログのタイムスタンプ使用。再生スクリプトが再生開始・終了時刻（ROS時刻）を記録→`audio_timestamps.txt`と時刻突合せでwav内秒へ換算。自動・再現性高いが、再生側ログ機構＋時刻同期（再生PCとHSR/録音側のクロック一致）要
       - 案b: 録音後の人手ラベル。`audio.wav`を波形/スペクトログラム表示ツール（Audacity等）で目視・聴取し区間を記録。追加機構不要で即着手可、ただし人手コスト・ラベル揺れあり。ラベル基準（区間端の決め方）を固定して記録要
       - 採用案と無関係に、ベースラインは既知区間（oracle相当）入力＝卒論で明記要（decisions.md決定11）
       - 未対応: ラベルファイル形式・置き場（例: `<data_dir>/time_ranges.txt`）、decisions.md未決事項欄への反映
     - `noisereduce`未import・未install（実機用経路）→ decisions.md決定11「帰結・注意」。新規関数で対処済み（下記「実装結果」）
     - 入力チャネル: 実機用経路は`librosa.load(sr=44100)`（既定mono=True、全ch平均＋44.1kHzリサンプル）。TAMAGO 8ch/16kHz入力時、どのマイク・chを使うか・ch3補正（`apply_tamago_ch3_gain_correction`）適用有無、未決
     - **公式コード実機用経路のバグ疑い（2026-09-28発見）**: `create_audio_map_batch_real`（`audio_mapping_utils.py:237-242`）、`time_ranges`（秒）を`convert_time_ranges_to_frame_ranges(time_ranges, fps)`で映像フレーム番号（秒×fps）に変換→その値で`audio[start:end]`をスライス。音声は44.1kHzサンプル列→例: fps=30で1〜3秒区間指定時、切出しは30〜90サンプル（約2ms）。本来サンプル番号（秒×sample_rate）のはず。呼出元リポジトリ外で意図確定不可。ベースライン使用前に修正要否判断要（修正時は「手抜きしない」原則の範囲として記録）
     - **固定fps前提**: 実機用経路はpose対応付けも固定`fps`でフレーム番号換算（`audio_mapping_utils.py:256`）。現行収集データは非等間隔（`avlmaps/dataset/timestamps.txt`: 122フレーム/約52秒、平均約2.3fps、先頭間隔0.05秒）→変換層は`timestamps.txt`基準で時刻→フレーム対応付け要
     - データ置き場2系統: `application/create_map.py:14`は`avlmaps_data_dir/avlmaps_dataset/`、`habitat_dataloader.py:156`（`main()`）は`avlmaps_data_dir/vlmaps_dataset/`。`application/`経由時は整合要
     - `SoundMap.create_sound_map`は`is_real`無視、常にHabitat用`create_audio_map_batch`呼出。`load_sound_map(is_real=True)`は`audio_video/audio_data.pkl`を読む→実機用pkl生成経路をSoundMap側に接続要
     - 既知バグ: `lang_robot.py`の`load_sound_map`、`SoundMap.__init__`シグネチャ不一致（呼ぶとTypeError）。本ステップ着手時に顕在化見込み
     - **2026-09-30発見**:
       - 実機用経路AudioCLIPモデルパス相対`os.path.join("audioclip/assets/", ...)`（cwd依存）、かつ関数内で毎回再ロード。`SoundMap`側は`Path(__file__)`基準で正→接続時`SoundMap.aclp`引数渡しへ改修が筋
       - 音声位置座標系: 実機用経路はカメラpose（`load_real_pose`、タブ区切り4x4）＋`rob2cam_tf`でbase位置算出。`index_sound_2d`は位置を`from_habitat_tf`入力扱い→変換層は現行`poses.txt`（map座標base pose）と同一座標系の位置を出力すべき（`rob2cam_tf`不要化可）
       - 音カテゴリ固定: `sound_collect_default.yaml` `difficulty: level_3`→ESC-50 domestic＋human＋animals集合内でテキスト照合。実機再生音源はこの集合から選択要。`is_ambiguous_data: True`だが`AVLMap`は`is_ambiguous=False`固定
       - `find_similar_category_id`（`index_utils.py`）、クエリ名がカテゴリ一覧と完全一致しないとOpenAI `text-davinci-002`（旧Completion API）呼出。モデル提供終了・`openai>=1.0`で旧API削除→一覧外クエリ失敗。VLMap`init_categories`使用時も同関数経由
       - TAMAGO 16kHz→44.1kHzアップサンプル: 8kHz以上帯域なし、AudioCLIPは44.1kHz学習。マイク選択の判断材料
       - AudioCLIP入力仕様（先頭5秒のみ等）→ researchoverview.md「AudioCLIP入力」
       - `create_audio_map_batch_real`、`setup_paths`（`audio_mapping_utils.py:181`・`:300`）リポジトリ内どこにも定義なし→`nr`より先にNameError
       - 同関数、`load_real_pose`（`:191`・`:268`）未import（定義は`mapping_utils.py:658`、importは`load_pose`・`cvt_pose_vec2tf`のみ、`clip_utils`の`*`importにも含まれず）→NameError
       - `create_audio_map_batch_real`は`time_ranges`必須引数、`SoundMap.create_sound_map(data_dir)`に受渡し手段なし→`is_real`分岐追加のみでは不足、インターフェース変更要
       - 上記＋`nr`未importで実機用経路は現状実行不可。公式コード実機用経路も呼出元リポジトリ外＝公開状態で未完成
       - 音位置＝録音時ロボット位置の設計→実験で音源はロボット近傍で鳴らす前提。原論文データ生成条件（5秒区間・2m以上間隔・1シーン1カテゴリ1回、researchoverview.md）に合わせるか要判断
     - `config/data_paths/default.yaml`の`avlmaps_data_dir`旧ダミー値。`SoundMap`自体は保持のみで未使用（実データは`create_sound_map`/`load_sound_map`引数`data_dir`基準）、影響は`application/`スクリプト経由時のみ
   - **B6着手前調査（2026-09-30）**: コード読解のみ、実機・実行未検証
     - **新規関数の雛形＝Habitat用経路`create_audio_map_batch`（`audio_mapping_utils.py:26`）が適切**、実機用経路でない
       - Habitat用経路は音源位置を`poses.txt`→`cvt_pose_vec2tf(posevec)[:3,3]`で算出（`:98-107`）。AreaMap `robot_pose_list`（`area_map.py:85`）と同一処理→B5でグリッド内確認済みの座標経路と整合、`index_sound_2d`→`from_habitat_tf`へそのまま渡せる
       - `from_habitat_tf`の位置計算は回転非依存（`inv_init_base_tf @ base_transform @ tf_hab @ inv(base_transform)`、右端は回転のみ）→位置のみ`tf_hab`で可
       - `rob2cam_tf`・`pose/`・mp4不要化
       - 実機用経路から取込むのは`time_ranges`入力＋`nr.reduce_noise`のみ
     - 新規関数の差分（Habitat用経路比）:
       1. 音声入力: mp4抽出→`audio.wav`を`lb.load(sr=44100)`（mono化＋リサンプル）
       2. セグメント: 無音検出→`time_ranges`（wav内秒）、切出しはサンプル番号（秒×sr）→実機用経路の秒×fpsスライスバグ疑い箇所は踏まない
       3. `nr.reduce_noise`適用（公式実機用経路準拠）
       4. フレーム対応: 固定fps→wav内秒を`audio_timestamps.txt`でROS時刻化→`timestamps.txt`で該当フレーム範囲
       5. 出力: `<data_dir>/audio_video/audio_data.pkl`（`load_sound_map(is_real=True)`読込パスと一致）
       - エンコード（`encode_audio_batch`・`*32768.0`・先頭5秒=220500サンプル）は既存流用可
     - **録音側**:
       - capture_node `signed_24`は不適の疑い: ALSAマッピング`SND_PCM_FORMAT_S24_LE`（4byte container、`audio_utils/src/PcmDevices/AlsaPcmDevice.cpp:171`）、TAMAGO `S24_3LE`（3byte）と別物。フレーム側`Signed24`は3byte/サンプル扱い→`hw:`直結は拒否、`plughw:`でもサイズ不整合疑い（実機未確認）
       - 推奨案: `plughw:CARD=TAMAGO03,DEV=0`＋`format:=signed_16`→ALSA変換、`audio_recorder_node.py`改修不要。ESC-50も16bit→精度損失ほぼなし。実機起動テスト時確認
       - `audio_recorder_node.py`は`setup.py`にentry_pointsなし→`ros2 run`不可、python直接実行（既存運用同様）
       - `noisereduce` 3.0.3、`.venv`から取得可確認（`pip download`のみ、未install）
     - 判断追加事項: `time_ranges`ラベルファイル形式・置き場（例: `<data_dir>/time_ranges.txt`、1行`start end`）。新規関数で置換なら実機用経路秒×fpsバグ修正不要
     - AudioCLIP入力はmono・44.1kHz→マイク・ch判断は「TAMAGO 8chをどうmono化するか（単ch or 全ch平均＝`librosa`既定、平均時ch3補正先行要）or 別マイク」
     - 再生音源: `dataset/ESC-50-master/audio`手元にあり、`level_3`（domestic＋human＋animals 30カテゴリ）から選択可
     - 改修対象ファイル（指示待ち）:
       - `audio_mapping_utils.py`: 新規関数＋`noisereduce` import
       - `sound_map.py`: `create_sound_map`に`is_real`分岐＋`time_ranges`受渡し、新規関数へ`self.aclp`渡し
       - `lang_robot.py:36` `load_sound_map`シグネチャ修正 or `hsr_lang_robot.py`でオーバーライド＋`setup_map`へSoundMapロード経路追加
       - `.venv`へ`noisereduce` install
       - 後回し可: `find_similar_category_id`旧OpenAI依存（クエリ名をカテゴリ名完全一致で回避可）

## R16. B6 SoundMap 実装結果・オフライン検証（2026-09-30）

- 出典: `strategy.md`「ベースライン実装順序」6.「実装結果」、239-263行（2026-10-22移設）
- 備考: コード実装・合成データ検証完了。実データ収集は`strategy.md` B6

   - **実装結果（2026-09-30）**:
     - `audio_mapping_utils.py`: `create_audio_map_hsr`＋補助`load_time_ranges`/`load_stamps`/`wav_sec_to_ros_sec`追加
       - 入力: `<data_dir>/audio.wav`・`audio_timestamps.txt`・`timestamps.txt`・`poses.txt`、`time_ranges`（wav内秒）
       - 処理: `soundfile`読込（native sr・多ch）→（`apply_ch3_correction`時ch3補正）→`channel`指定ch or 全ch平均→44.1kHzリサンプル→`nr.reduce_noise`（`reduce_noise=True`既定）→サンプル番号で切出し→`encode_audio_batch`
       - 時刻対応: capture_nodeのstampは**フレーム末尾サンプル時刻**（`start_time + 累積サンプル数（当該フレーム込み）`、`capture_node.cpp:70-76`）、recorderの`sample_offset`はフレーム前累積→フレームk末尾=`sample_offset[k+1]`としてwav内秒→ROS時刻換算。capture_node側timestamp reset（±500ms逸脱時）もフレーム毎アンカーで吸収
       - 区間内フレーム0件時: 区間中心の最近傍フレームで代替（ログ出力）
       - 出力: `<data_dir>/audio_video/audio_data.pkl`
       - `noisereduce`は関数内import→`noisereduce`未installのenv（`avlmaps` conda env等）でも`avlmaps.map`芋づるimport非破壊
     - `sound_map.py`: `create_sound_map(data_dir, time_ranges=None, channel=None, apply_ch3_correction=False, reduce_noise=True)`。`is_real`時のみ新引数使用、`time_ranges`省略時`<data_dir>/time_ranges.txt`（1行`start_sec end_sec`、`#`以降コメント）読込。Habitat経路は無変更
     - `hsr_lang_robot.py`: `load_sound_map`オーバーライド（`SoundMap(dir, sound_config, sound_data_collect_params, is_ambiguous=False, is_real=True)`）。`setup_map`は`audio_video/audio_data.pkl`存在時のみSoundMapロード、無ければwarnログのみ（現datasetで`setup_map`を壊さないため）。configに`sound_config`・`sound_data_collect_params`要（`map_indexing_cfg`compose時は含まれる）
     - 既定値（channel=None全ch平均・ch3補正なし・ノイズ除去あり）は公式実機用経路の挙動準拠。マイク・ch判断確定後に呼出側で指定
     - 環境: `/opt/pytorch/.venv`へ`noisereduce` 3.0.3・`coverage` 7.16.2追加install（sudo、venvはroot所有）
       - `coverage`理由: `.venv`の`numba` 0.67がシステム`/usr/lib/python3/dist-packages`の`coverage` 7.4.4（`coverage.types.Tracer`なし）を拾い`AttributeError: module 'coverage.types' has no attribute 'Tracer'`→`librosa`音声関数（`lb.load`/`lb.resample`等）が`.venv`で全滅していた（既存`get_pos_with_audio`も同様、未顕在化だった）
       - 注: ログインシェル`PYTHONPATH`に`.venv` site-packages含む→同PYTHONPATH下のシステムpython（ROS2ツール）も`coverage` 7.16.2を見る。`PYTHONPATH`なしなら7.4.4のまま
       - 注: sudoなし`pip install`はvenv書込不可で`~/.local`（ユーザーsite）へフォールバックする→システムpythonにも効くため不可。一度誤って入った分は撤去済み
     - **オフライン検証PASS**（合成データ、scratchpad）: 実`poses.txt`/`timestamps.txt`＋ESC-50（dog・clock alarm・crying baby、各5秒）を8ch/16kHz/16bit wavへ配置＋`audio_recorder_node`形式`audio_timestamps.txt`（1024サンプル/フレーム、stamp=フレーム末尾）
       - 3区間とも位置一致（frames 23..33／63..74／102..112、期待フレームと完全一致）
       - `HSRLangRobot.setup_map`→SoundMapロード3件、`get_sound_pos('dog')`11位置
       - テキストクエリ: dog・crying babyは正区間。**clock alarmは誤り（dog区間）**
     - **16kHz帯域制限の影響を実測**: 同じESC-50 3クリップでtop1分類比較
       - 44.1kHz原音直接: dog / clock alarm / crying baby（全正）
       - 16kHz往復のみ（ノイズ・ノイズ除去なし）: dog / **vacuum cleaner** / crying baby
       - 本パイプライン: 同上
       - →誤りは実装でなく16kHz化（8kHz以上欠落）起因。高域主体カテゴリ（clock alarm等）で劣化→入力マイク判断（TAMAGO 16kHz vs 44.1kHz以上マイク）の重要材料。1例のみ、定量評価未実施
     - 作業順（案）: ①実機`capture_node`（`plughw`＋`signed_16`、8ch/16kHz）＋`audio_recorder_node.py`起動、`hsr_data_collector.py`と同時稼働確認 ②マイク・ch／音源配置／音カテゴリ／ラベル形式判断 ③音源鳴らしつつ収集→`time_ranges`人手ラベル ④新規関数＋SoundMap改修→`audio_data.pkl`生成→テキストクエリ確認

## R17. B6 録音経路 実機起動テスト（TAMAGO単体、2026-10-20）

- 出典: `strategy.md`「ベースライン実装順序」6.「実機起動テスト結果」、264-290行（2026-10-22移設）
- 備考: TAMAGO単体PASS。「HSR⇔PC時刻同期」懸念はsobit_edu（同一PC接続）で構造上解消見込み→代わりにXtion `use_device_time`起因の画像⇔音声stampずれ実測（`strategy.md`）。HSR同時稼働は不要化、sobit_edu同時稼働確認は`strategy.md`

   - **実機起動テスト結果（2026-10-20）**: TAMAGO接続・HSR非接続（有線IFなし）。録音経路TAMAGO→`capture_node`→`audio_recorder_node.py`→`create_audio_map_hsr`読込関数まで。コード改修なし、出力はscratchpad
     - 結論: `plughw:CARD=TAMAGO03,DEV=0`＋`signed_16`で動作、recorder改修不要。ただし**capture_nodeに`latency_us:=256000`指定必須**
     - 起動コマンド（検証済み）→ docs/step1.md「音声連続記録ノード実装」節
     - ALSA単体（`arecord`）:
       - `hw:`＋S16_LE拒否（S24_3LE固定再確認）→ docs/step1.md「ネイティブHWパラメータ確定」
       - `plughw:`＋S16_LE 8ch/16kHz、3秒・20秒とも成功。ファイルサイズ理論値（秒×16000×8ch×2byte＋44）一致
       - ch別RMS: 全ch信号あり、飽和なし、DC約-0.6
     - **capture_node既定`latency_us`で即停止**:
       - 既定（`latency_us`=64000）→約1秒（17フレーム）で`AlsaException: Read from audio interface failed (5: Input/output error)`、exit 255
       - 原因: `frame_sample_count=1024`＝64ms、バッファ64ms＝1周期分のみ→オーバーラン常発。`arecord --period-size=1024 --buffer-time=64000`で同条件「オーバーラン!!!」再現
       - `5`はerrnoでなく`snd_pcm_readi`短読みフレーム数の疑い: `AlsaPcmDevice::read`（`AlsaPcmDevice.cpp`）はEPIPE時`snd_pcm_recover`→再readi、戻り値`!= periodSize`で例外、戻り値をそのまま`snd_strerror`へ渡す→正値5がEIO文言表示（コード読解ベース）
       - `latency_us:=256000`（4周期）で60秒エラーなし
     - 出力検証（60秒録音）:
       - wav: 8ch / 16kHz / PCM_16、959488サンプル（59.968秒）
       - `audio_timestamps.txt`: 937行、`sample_offset`全刻み1024（欠落なし）、最終offset＋1024＝wavサンプル数一致
       - stamp間隔: 全行0.0640秒（±1ms逸脱0件）、stamp幅59.904秒＝サンプル数換算と一致。`ros2 topic hz /audio_out` 15.625Hz（16000/1024一致）
       - 全ゼロサンプル0件
       - 注: stampはcapture_node計算値（開始時刻＋累積サンプル数/rate）→実測到着時刻でない。長時間録音時のデバイスクロック⇔システム時計ドリフトは本テストで不可視（±500ms超でcapture_node側reset）
     - `create_audio_map_hsr`側接続: 実`audio_timestamps.txt`を`load_stamps`・`wav_sec_to_ros_sec`へ入力、wav内0／0.064／10／59.9秒→ROS時刻、期待値（先頭stamp−0.064＋秒）と全一致
     - **新規判明**:
       - **HSR⇔PC時刻同期要確認**: `timestamps.txt`＝RGB `header.stamp`（HSR本体時計）、capture_node stamp＝`m_clock.now()`基準（コンテナ実行PC時計）。`create_audio_map_hsr`は両者を直接突合せ→時計ずれ分だけ音源位置が別フレームへ割当。約2.3fps→1フレーム約0.4秒、数百ms級ずれで影響。HSR接続時`date +%s.%N`同時比較等で確認要
       - ch3感度異常 再現せず（ch3比1.0〜1.06）→ docs/step1.md「ch3感度異常、再現確認」節注記。音源鳴らして再確認要、ch3補正判断前に必須
       - 循環import: `avlmaps.utils.audio_mapping_utils`を最初にimport→`ImportError: cannot import name 'get_level_categories' from partially initialized module 'avlmaps.utils.audio_utils' (most likely due to a circular import)`。`import avlmaps.map`先行で回避。`HSRLangRobot`経路は`avlmaps.map`先行→影響なし。単体スクリプト作成時注意
       - recorder Ctrl+C停止時`RCLError: failed to shutdown: rcl_shutdown already called on the given context`（rclpy shutdown二重呼出）。`destroy_node`先行→wav・timestampsは正常クローズ、実害なし
       - recorder購読開始前フレームは記録されない（capture_node `queue_size`=1）。wav・timestampsは同一フレームから開始→整合性保持。録音はrecorder起動後に音源再生開始
     - 環境: `CYCLONEDDS_URI` unset＋`ROS_DOMAIN_ID=77`（HSR有線IFなし）、PYTHONPATHから`/tmp/opencv_pkg_install`除外、`.venv`実行
     - 未実施（HSR接続要）: `hsr_data_collector.py`同時稼働（取りこぼし・負荷・時刻範囲重複）、HSR⇔PC時刻同期

## R18. 環境状況（判明済み）・CYCLONEDDS_URI問題（HSR特有、解決扱い）

- 出典: `strategy.md`「環境状況（判明済み）」節、341-356行（2026-10-22移設）
- 備考: 2026-10-22ユーザー判断: `CYCLONEDDS_URI`プレースホルダー・hsrb_libraryプロファイル・HSR有線IF関連はHSR特有→解決扱い。他項目は各参照先が正

## 環境状況（判明済み）

※2026-10-22注記: `hsrb_library`・`CYCLONEDDS_URI`（hsrb_libraryプロファイル）・HSR有線IF関連はHSR時代記録。sobit_eduは`sobit_edu`パッケージ群＋`sobits_navigation_stack`使用、DDS設定のsobit_edu運用方針は未整理（2026-10-22実機確認はCLAUDE.md「実機に関する備考」節）。

- ROS2 Jazzy / Python 3.12 / Dockerコンテナ内（CLAUDE.md前提ルール記載）
- sobit_edu（2026-10-22〜）: `colcon_ws/src/sobit_edu`（bringup/description/control/library）・`sobits_navigation_stack`（`sobits_nav`・`sobits_slam`）、Jazzyビルド済み・実機起動確認済み（「ロボット変更」節）
- HSR: `hsrb_library`導入・実機接続 済（B0）→ docs/resolved.md R1
- 🔶新規判明（2026-09-07）: コンテナ環境変数`CYCLONEDDS_URI`がプレースホルダーのまま（`/path/to/cyclonedds_profile.xml`）でDDS疎通不可だった。正しいパスは別変数`CYCLONEDDS_URI_PATH`にあり、`export CYCLONEDDS_URI="file://$CYCLONEDDS_URI_PATH"`で疎通確認できた（作業セッション限定の一時回避、恒久修正未着手、優先度低）
- 🔶`avlmaps` conda env PYTHONPATH汚染（毎回`unset PYTHONPATH`要）→ docs/step3.md「ブロッカー」節1
- 🔶`avlmaps` env依存不足（requirements.txt）→ docs/step3.md「ブロッカー」節2
- 🔶`avlmaps/map/__init__.py`芋づる式import問題・案a → docs/step3.md「重大発見: `avlmaps/map/__init__.py`一括import問題」節。現状: VLMap・SoundMap・VLMapMultiFloor・AreaMap公開、VisualMapのみコメントアウト（B7）
- 🟢 Python環境二重化問題（2026-09-22解消）: `/opt/pytorch/.venv`へ追加install（a案）でrclpy＋VLMap/SoundMap/HSRLangRobot同一プロセス稼働。決定経緯・install実施結果・SoundMap封印解除 → docs/resolved.md R11・R12。運用上の注意（PYTHONPATHから`/tmp/opencv_pkg_install`除外、恒久対応未実施・優先度低）はCLAUDE.md「環境構築・依存関係の罠」節

### colcon_ws内 既存資産（AVLMaps外）

- `audio_utils`: 録音基盤は`capture_node`のみ使用（ベースライン・提案手法共通）、`raw_file_writer_node`不採用 → docs/step1.md「音声連続記録ノード実装」節

## R19. 移植方針の前提整理（Panda非公開・nav2置換・モジュール分類）

- 出典: `strategy.md`「前提訂正: Panda向けコードは非公開」「各モジュール移植難易度」節、314-339行（2026-10-22移設）
- 備考: 方針確定済み・`HSRLangRobot`実装済み（R8）

## 前提訂正: Panda向けコードは非公開

AVLMaps公開コードにPanda/Franka関連実装なし（`avlmaps/`配下grep 0件）。
`avlmaps/robot/`中身:
- `lang_robot.py`: 抽象基底`LangRobot`。全メソッド`NotImplementedError`
- `habitat_lang_robot.py`: Habitatシム専用実装のみ

論文Section IV-E実機実験（Ridgeback+Panda+RealSense D435）用コード非公開。
→ 作業は「Panda部分置換」ではなく「LangRobot抽象基底に対しHSR向け実装ゼロから作成」。

## 各モジュール移植難易度（コード読了確認済み）

### 不採用決定: navigator/discrete_nav_controller → nav2に置換（2026-09-02決定）
- `navigator/navigator.py`（visibility graph+Dijkstra）、`controller/discrete_nav_controller.py`（move_forward/turn_left/turn_right離散コマンド生成）ともHabitat要素なし・流用可能だったが、**採用しない**
- 理由: 両モジュールはHabitatシムに実ナビゲーションスタックが存在しないための自前実装。実機ではnav2が同役割（グローバルパスプランニング＋障害物回避）を標準的に担う → 自前実装は不要かつ劣る選択
- nav2採用はフェアネス原則（CLAUDE.md）に抵触しない: 縛りは`VLMap`・`AreaMap`のアルゴリズムのコア部分＝ヒートマップ生成・LSeg/CLIP特徴抽出（将来SAM3等へアップグレード時は両手法同時適用）。ナビゲーション実行方式は対象外の統合ロジック層 → AVLMapsベースライン・提案手法どちらもnav2使えば公平性保たれる
- 注意: nav2のoccupancy costmap（走行安全・自己位置用）とAVLMapsのセマンティックグリッド（`vlmap_builder.py`生成、物体/エリア意味情報用）は別物、混同しない

### 軽微改修（クラス名・変数名がHabitat前提なだけ、ロジック自体汎用）
- 対象（`habitat_dataloader.py`・`vlmap_builder.py`等）→ docs/packages.md「実機研究向け分類 > 要改修」
- 調査結果（座標変換式無改修・`base_transform`はmap_config新規で対処）→ docs/step2.md

### 新規実装必須
※2026-10-22注記: sobit_eduでも同構成（`HSRLangRobot`をパラメータ指定で流用、`base_footprint`→`sobit_edu/base_footprint`）。
- `HSRLangRobot`設計（nav2`navigate_to_pose`/`spin`置換、pose取得元`tf2`）→ docs/step4.md「実装状況」節・docs/resolved.md R8
- `map`フレーム供給（nav2起動必須）→ docs/step1.md「nav2起動によるmap frame確保」節

## R20. ベースライン実装順序 B0〜B4 要約（2026-10-22時点）

- 出典: `strategy.md`「ベースライン実装順序」0.〜4.、143-166行（2026-10-22移設）
- 備考: HSR上で完了。sobit_eduで再実施（`strategy.md`「現在地」）。正式起動スクリプト未作成は`strategy.md`残置

## ベースライン実装順序

※2026-10-22注記: 本節の✅・実機結果はHSR上（HSR時代記録）。sobit_eduでの再取得要否は「現在地」節。各項目の🔁＝sobit_eduで再実施要。

0. **HSR ROS環境構築・接続確認** ✅完了（詳細docs/resolved.md R1） 🔁sobit_edu: bringup・センサ・TF確認済み（2026-10-22、「ロボット変更」節）、nav2未起動

1. **データ収集ノード** 🔁sobit_eduで再収集要（topic・base_frameパラメータ指定のみ、コード改修不要見込み） 🔶`hsr_data_collector.py`実機検証済み（HSR）・`audio_recorder_node.py`はTAMAGO単体で実機検証済み（2026-10-20、HSR同時稼働未検証、B6「実機起動テスト結果」）
   - RGB-D + base TF + 音声、同期記録するROSノード作成
   - poses.txt相当、TFタイムスタンプ同期で生成
   - 実装完了: `hsr_data_collector.py`（RGB-D+TF同期、timestamps.txt込み）、`audio_recorder_node.py`（連続wav+timestamps、案A方式）。詳細docs/step1.md
   - ✅RGB-D側実機検証・往復122フレーム収集（`avlmaps/dataset/`）・追記モードバグ修正・TAMAGOパススルー解消 → docs/resolved.md R2〜R4
   - 🔶TAMAGO ch3感度異常・補正方針（案A、`apply_tamago_ch3_gain_correction()`実装済み・呼出元未実装）→ docs/step1.md「TAMAGO実機検証結果」節。適用有無はB6判断事項
   - **音声側の未検証事項はB6の前提としてB6に集約**（`audio_recorder_node.py`実機起動テスト・ビット深度不整合）

2. **dataloader改修** ✅完了（2026-09-05、詳細docs/step2.md・docs/resolved.md R5・R6（TFテスト10/10 PASS））
   - 水平オフセット無視運用決定（HSR）→ docs/step2.md「残課題 → 判断確定」節
   - 🔁sobit_edu: ロジック流用可。`config/map_config/sobit_edu.yaml`新規要（未作成）。水平オフセット前方+0.1297m・左右+0.023m（HSR比前方約2倍）→無視運用の妥当性再判断要

3. **VLMap構築（vlmap_builder動作確認）** ✅完了（HSRデータ、詳細docs/resolved.md R7・step3.md） 🔁sobit_eduデータで再構築要
   - AreaMapは本ステップで未構築（B5へ）

4. **HSRLangRobot実装 + nav2結合テスト** ✅完了（HSR、Stage1〜3、2026-09-23、詳細docs/resolved.md R8〜R10・docs/step4.md） 🔁sobit_edu（差動二輪）でStage1〜3再検証要。`base_frame:=sobit_edu/base_footprint`、nav2は`robot_name:=sobit_edu`＋`params_file`指定
   - 実装構成・使用地図（`room1.yaml`）→ docs/resolved.md R8・docs/step4.md。`setup_map`のAreaMap・SoundMapロード経路はB5・B6で追加済み
   - `main()`相当の正式起動スクリプト未作成（scratchpadスクリプト方式で運用中）

## R21. 解決済み未決事項・実施記録ポインタ

- 出典: `strategy.md`「残未決事項」取消線項目（361・363・364・371行）＋「実施記録」節（305-312行）（2026-10-22移設）
- 備考: 未解決懸念（地図原点・recovery・角度規約）は`strategy.md`「問題顕在化まで扱わない」が正

- ~~`time_ranges`取得方法~~ → 当面 録音後の人手ラベル（案b）で決定（2026-09-30、案a 再生ログ方式は候補として保持、B6参照）
- ~~`audio_recorder_node.py`ビット深度対応方式（`plughw`変換 or recorder側24bit対応、B6）~~ → `plughw`＋`signed_16`で解消確認（2026-10-20、B6「実機起動テスト結果」）
- ~~HSR⇔PC時刻同期の確認・必要時補正方法（B6、2026-10-20判明）~~ → sobit_edu変更で構造上解消見込み。代わりにXtion `use_device_time`起因の画像⇔音声stampずれ実測（2026-10-22）
- ~~音声記録方式~~ → 案A（連続wav1本＋タイムスタンプログ）決定・実装済み、docs/step1.md「音声記録方式決定」節

## 実施記録

→ docs/resolved.md へ移設（2026-09-30）。R3 追記モードバグ／R4 dataset再収集／R6 TFテスト／R9 Stage2／R10 Stage3。

移設エントリ内の未解決懸念（本ファイルで継続管理、問題顕在化まで扱わない）。※2026-10-22注記: いずれもHSRでの観測。sobit_eduでは地図新規作成（収集・ナビ同一地図）で原点同一性懸念は解消可、recovery・着地ズレはsobit_edu再検証時に再観測要:
- VLMap座標系原点（poses.txt開始位置）とnav2`map`座標系の同一性前提、`room1.yaml`と実環境の対応未確認（R9）
- recovery発生・着地ズレ0.03〜0.14m級、原因未特定。`xy_goal_tolerance=0.15m`→goal距離は0.15mより十分大きく取る（R10）
- `get_north/south/east/west_pos`角度規約未検証（R10、既知バグB区分）
