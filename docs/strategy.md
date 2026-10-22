# 実機移植 実装計画（sobit_edu）

AVLMapsベースライン（本リポジトリ）実機移植の**未完了タスク・未決事項のみ**置く。提案手法（study側）タスク・実装順序は持ち込まない。

- 完了済み記録 → resolved.md（R番号）。外した記述 → archive.md。設計判断 → decisions.md
- 2026-10-22 ロボットHSR→sobit_edu（decisions.md決定12）。HSR上の完了結果はsobit_eduで再取得要

## 現在地（2026-10-22）

完成条件: VLMap・AreaMap・SoundMap構築＋`AVLMap`統合推論（`index_object`/`index_area`/`index_sound`）がsobit_edu実機データで動作（VisualMapはB7で判断）。

- ✅ ロボット非依存で完了・流用可: B2 dataloader（R5・R6）、Python環境（R11・R12）、AreaMapロード経路（R14）、B6コード実装・オフライン検証・TAMAGO録音経路（R15〜R17）
- ✅ sobit_edu bringup・センサ・TF確認（R13）
- 🔁 HSRで完了・sobit_eduで再実施: B1 RGB-D収集（R2〜R4）/ B3 VLMap（R7）/ B4 nav2結合Stage1〜3（R8〜R10）/ B5 AreaMap構築（R14）
- 🔶 B6 SoundMap実データ収集: 未
- ⬜ B7 VisualMap判断 / B8 `AVLMap`統合推論: 未着手。統合推論は実機データで一度も未動作

## 今後の流れ（案）

1. nav2起動（`robot_name:=sobit_edu`・`params_file`指定）。地図未作成なら先に`sobits_slam`でSLAM→地図保存
   - **規約（2026-10-22）: 全収集セッション（撮影・録音）、同一保存地図＋AMCL下で実施。SLAM中収集禁止**。理由: collector、撮影時点`map→base_footprint`をそのまま`poses.txt`保存→SLAM中`map`frame暫定値（loop closureで地図側のみ遡及補正）、保存poseと最終地図ずれる。規約下なら撮影・録音を別セッション取得してもデータ混用可（VLMapグリッド`poses[0]`基準、SoundMap/AreaMap `map`frame生pose）
2. `map`→`sobit_edu/base_footprint` TF確認
3. `cam_calib_mat`方針判断（公称値 or キャリブレーション）
4. `hsr_data_collector.py`＋`capture_node`＋`audio_recorder_node.py`同時稼働確認、画像stamp⇔音声stampずれ実測
5. `config/map_config/sobit_edu.yaml`作成
6. RGB-D再収集→VLMap（B3）・AreaMap（B5）再構築、全poseグリッド内確認
7. `HSRLangRobot` nav2結合再検証（B4 Stage1〜3相当、差動二輪）
8. B6実データ収集→`time_ranges`ラベル→SoundMap構築・クエリ確認
9. B7・B8

並行: 下記「未決事項」のユーザー判断。

## sobit_edu移行の要対応事項

起動手順・ポート・カメラ設定・実測値 → R13。

- 名前空間: 全ノード`/sobit_edu`、`frame_prefix=sobit_edu/`→`base_frame`は`sobit_edu/base_footprint`
- `hsr_data_collector.py`（パラメータ指定のみ、改修不要）: `rgb_topic:=/sobit_edu/head_camera/rgb/image_raw` `depth_topic:=/sobit_edu/head_camera/depth_registered/image_raw` `camera_info_topic:=/sobit_edu/head_camera/rgb/camera_info` `base_frame:=sobit_edu/base_footprint`（`map_frame`は`map`のまま）。Depthは32FC1（m）→既存経路で対応可
- `HSRLangRobot`: `base_frame:=sobit_edu/base_footprint`。nav2 actionはルート名前空間→そのまま
- `sobit_edu.yaml`新規:
  - `camera_height` 0.9406（HSR 0.967172）
  - 水平オフセット 前方+0.1297m・左右+0.023m（HSR比前方約2倍）→無視運用の妥当性再判断
  - `base2cam_rot`等軸設定: HSR同様の扱いで要確認
  - `cam_calib_mat`: `vlmap_builder.py`はconfig値使用→更新必須。`camera_info`はOpenNI公称値（fx=fy=570.342、cx=319.5、cy=239.5）、キャリブレーション値でない
  - マッピング走行時 head pan=tilt=0固定（`pose_list.yaml` `initial_pose`）
- nav2: `sobits_nav/param/sobit_edu/navigation_config.yaml`既存。`nav2.launch.py`は`robot_name_val = "hsrb_robot"`ハードコード→起動引数`robot_name:=sobit_edu`＋`params_file`指定 or 書換（別リポジトリ）
- 地図: `sobits_slam`で新規作成。収集・ナビ同一地図化で原点同一性懸念も解消可
- 時刻同期: TAMAGO・Xtion同一PC接続→HSR⇔PC時計ずれは構造上解消見込み。ただしxtion.launch.py `use_device_time: True`→画像stamp（デバイス時計）⇔音声stamp（`m_clock.now()`）ずれ実測要
- 既存`avlmaps/dataset/`（VLMap・AreaMap・`camera_intrinsics.txt`）はHSR収集→再収集・再構築
- `get_sim_cam_mat`（FOV90°前提、`vlmap_builder.py:126`）は元コード由来・HSR時から同状況

## 未完了タスク

### B6 SoundMap実データ収集

- 前提: sobit_edu下で同時稼働確認、画像⇔音声stampずれ実測（上記）
- 作業: 音源再生しつつ収集→`time_ranges`人手ラベル（`<data_dir>/time_ranges.txt`、1行`start_sec end_sec`）→`SoundMap.create_sound_map`→`audio_data.pkl`→クエリ確認
- 判断材料:
  - マイク・ch: `create_audio_map_hsr`既定は全ch平均・ch3補正なし。TAMAGO 16kHz→8kHz以上欠落、clock alarm誤分類を実測（R16、1例のみ）
  - ch3補正: 判断前に音源鳴らして感度再確認要（step1.md）
  - 音カテゴリ: `level_3`（ESC-50 domestic＋human＋animals 30カテゴリ）内から選択、`dataset/ESC-50-master/audio`手元
  - 音源配置: 音位置＝録音時ロボット位置→ロボット近傍で鳴らす前提。原論文条件（5秒・2m以上間隔・1カテゴリ1回、researchoverview.md）に合わせるか
- 注意: 録音はrecorder起動後に音源再生開始。単体スクリプトは`import avlmaps.map`先行（循環import、R17）
- 残課題: `find_similar_category_id`旧OpenAI API依存→一覧外クエリ名失敗（カテゴリ名完全一致で回避可）

### B7 VisualMap扱い

- VisualMap（hloc/pycolmap/`third_party.SuperGluePretrainedNetwork`、リポジトリ外依存）は`__init__.py`でコメントアウト中
- `AVLMap.__init__`/`create_map`/`load_map`がVisualMap無条件使用→扱わない場合VisualMap任意化要

### B8 `AVLMap`統合推論

- `avlmap.py:14`のimport、`VisualMap`非公開で失敗（B7次第）
- `SoundMap(..., is_real=False)`固定（`avlmap.py:28`）→パラメータ化
- クロスモーダル統合、公開コードに不在（researchoverview.md「クロスモーダル統合」）→論文仕様再実装 or 単独モダリティ比較。decisions.md決定3の比較元実体
- Soundヒートマップ集約: 論文max・コード加算（`avlmap.py:129`）相違
- `index_sound_2d`境界チェックなし（`avlmap.py:120-122`）→グリッド外でIndexError or 負index wrap
- ヒートマップ最大位置→nav2移動経路、`HSRLangRobot`になし
- 既存`application/`実機使用不可（`import habitat_sim`、`data_paths: lab_new`不在、`avlmaps_dataset/<scene>/`構造前提、`avlmaps_data_dir`旧ダミー値）→構築・クエリスクリプト新規要
- 依存: B5・B6・B7

### その他（優先度低）

- 正式起動スクリプト（`main()`相当）なし、scratchpadスクリプト運用

## 未決事項（ユーザー判断）

- B6: 入力マイク・ch、ch3補正有無、音源配置条件、使用音カテゴリ、`time_ranges`ラベル形式確定（案a 再生ログ方式は候補保持、R15）
- B7: VisualMap扱い
- B8: 統合ロジック定義（フェアネス原則直結）、Sound集約（max vs 加算）
- B6・B8: 一覧外クエリ名の照合方法
- sobit_edu: `cam_calib_mat`、水平オフセット無視運用、`nav2.launch.py` `robot_name`対処、HSR名残存ファイル改名要否

## 問題顕在化まで扱わない（2026-09-23ユーザー判断）

- 既知バグ3件（CLAUDE.md「既知バグ」節）
- ナビ精度: recovery・着地ズレ0.03〜0.14m級、`xy_goal_tolerance=0.15m`→goal距離は0.15mより十分大きく取る（R10）。HSR観測、sobit_eduで再観測要
- VLMap座標系原点とnav2`map`座標系の同一性（R9）: sobit_eduは収集・ナビ同一地図で解消可→「今後の流れ（案）」手順1規約で解消扱い（規約遵守前提）
