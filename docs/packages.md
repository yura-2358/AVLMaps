# AVLMaps パッケージ構成

音声・視覚・言語マルチモーダル3Dマップ、ロボットナビゲーション向け。音声/視覚/言語情報クロスモーダル統合、LLM連携ゼロショット空間ナビ実現。

## ディレクトリツリー

```
AVLMaps/
├── application/            エントリポイント層(実行スクリプト)
├── avlmaps/                コア実装パッケージ
│   ├── map/                3Dマップ表現
│   ├── robot/               ロボット制御インタフェース
│   ├── task/                ナビゲーションタスク定義
│   ├── navigator/           経路計画
│   ├── controller/          ナビゲーション制御
│   ├── dataloader/          座標変換
│   ├── lseg/                言語セグメンテーションモデル
│   ├── audioclip/           音声-視覚マルチモーダルモデル
│   └── utils/                共通ユーティリティ(19ファイル)
├── config/                  Hydra設定(yaml)
├── docs/                    ドキュメント
├── install.bash             環境構築
├── download_checkpoints.bash モデルチェックポイント取得
├── requirements.txt         主要依存
├── setup.py / pyproject.toml パッケージ設定
└── README.md
```

## 全体アーキテクチャ

層構造:

- Application層(`application/`) → Integration層(`avlmaps/map/avlmap.py`) → 各マップ型(VLMap/SoundMap/AreaMap/VisualMap) → 特徴抽出層(lseg/audioclip/utils) → グリッド3D特徴保存(H5DF)
- Navigation/Control層: Navigator + Controller + HabitatLanguageRobot + Task

デザインパターン:

- Factory: `Map.create()` → VLMap/VLMapMultiFloor 生成
- Builder: VLMapBuilder, VLMapBuilderMultiFloor
- Strategy: NavController(discrete/continuous)、Map種別
- Template Method: HabitatTask(load_task/test_actions)
- Adapter: VLMapsDataloaderHabitat(Habitat座標↔グリッド座標)

## application/ — エントリポイント

- `create_map.py`: Hydra CLI、AVLMap.create_map実行、マップ作成パイプライン(VLMap+SoundMap+AreaMap+VisualMap生成)
- `index_map.py`: インタラクティブマップ検索UI、オブジェクト/音声/エリア/画像クエリ対応
- `generate_obstacle_map.py`: 障害物マップ生成(カスタマイズ版)

## avlmaps/map/ — 3Dマップ表現

- `map.py`: 抽象基底クラス`Map`。グリッド座標系・RGB・特徴量管理・障害物生成・create/load/index_map定義。Factoryメソッド`create()`でVLMap/VLMapMultiFloor生成
- `avlmap.py`: 統合クラス`AVLMap`。VLMap・SoundMap・AreaMap・VisualMap統合管理。create_map/load_map/index_object/index_sound/index_area
- `vlmap.py`: `VLMap`(Map継承)。LSeg特徴+CLIPによるセマンティック3Dマップ。create_map(VLMapBuilder使用)/load_map/index_map(CLIPスコア)
- `vlmap_builder.py`: `VLMapBuilder`。RGB-Dフレーム逐次処理→LSeg特徴抽出→3Dバックプロジェクション→グリッド特徴集約。create_mobile_base_map()/create_camera_map()
- `vlmap_multi_floor.py`: `VLMapMultiFloor`(Map継承)。マルチフロア対応版、VLMapBuilderMultiFloorで構築
- `vlmap_builder_multi_floor.py`: `VLMapBuilderMultiFloor`。グローバル座標系/モバイルベース/カメラ座標系対応
- `vlmap_3d.py`: 3D VLMap、旧実装
- `sound_map.py`: `SoundMap`。AudioCLIPで音声特徴抽出→空間位置マッピング
- `area_map.py`: `AreaMap`。CLIP(ViT-L/14)、フレーム毎画像特徴→密集スパース表現
- `visual_map.py`: `VisualMap`。HLocLocalizerで参照フレームベース画像ローカライゼーション
- `interactive_map.py`: `InteractiveMap`。matplotlib、topdown map描画、ユーザ入力受付
- `clip_map.py`/`gtmap.py`/`gradcam_map.py`: 旧実装・未使用スタブ

クラス継承: `Map` ← `VLMap`, `VLMapMultiFloor`。`AVLMap`/`SoundMap`/`AreaMap`/`VisualMap`/`InteractiveMap`は独立クラス。

## avlmaps/robot/ — ロボット制御

- `lang_robot.py`: 抽象基底`LangRobot`。マップ/音声マップローディング、ナビゲーションAPI(move_to, turn等)。execute_actions/get_agent_pose_on_map/get_sound_pos(stub)定義
- `habitat_lang_robot.py`: `HabitatLanguageRobot`(LangRobot継承)。Habitatシミュレータ統合。setup_scene()でシム初期化+Navigator/Controller組込

## avlmaps/task/ — ナビゲーションタスク

- `habitat_task.py`: 抽象基底`HabitatTask`。ゴール位置設定、タスク実行検証、メトリクス管理
- `habitat_object_nav_task.py`: `HabitatObjectNavigationTask`(HabitatTask継承)。JSON複数オブジェクト目標読込、成功判定
- `habitat_spatial_goal_nav_task.py`: Spatial Goal Navigationタスク(スタブ)

## avlmaps/navigator/ — 経路計画

- `navigator.py`: `Navigator`。visibility graph構築→Dijkstra経路計画。build_visgraph()/plan_to(start,goal→パス)

## avlmaps/controller/ — ナビゲーション制御

- `controller.py`: 抽象基底`NavController`。ゴール→アクション変換、アクション→ポーズ予測
- `discrete_nav_controller.py`: `DiscreteNavController`(NavController継承)。離散アクション(move_forward, turn_left, turn_right)
- `continuous_nav_controller.py`: `ContinuousNavController`。連続アクション制御(旧実装、不完全)

## avlmaps/dataloader/ — 座標変換

- `habitat_dataloader.py`: `VLMapsDataloaderHabitat`。Habitat座標↔マップグリッド座標相互変換。from_habitat_tf()/to_full_map_pose()等

## avlmaps/lseg/ — 言語セグメンテーション

RGB画像→ピクセルレベル無限語彙セマンティック分割特徴抽出用モデル一式。

- `modules/lseg_module.py`: `LSegModule`、PyTorch Lightning学習モジュール
- `modules/lsegmentation_module.py`: 基底セグメンテーションモジュール
- `modules/models/lseg_net.py`: `LSegNet`エンコーダネットワーク
- `modules/models/lseg_vit.py`: Vision Transformerバックボーン
- `modules/models/lseg_blocks.py`: 構成要素(FeatureFusionBlock, depthwise_conv等)
- `additional_utils/models.py`: 画像前処理(resize/pad/crop)
- `additional_utils/encoding_models.py`: エンコーディング関連
- `checkpoints/`: LSegモデルチェックポイント格納先

## avlmaps/audioclip/ — 音声-視覚マルチモーダルモデル

音声分類・検索、音声→ナビゲーション指示用。

- `model/audioclip.py`: `AudioCLIP`。audio+image+text→共有埋め込み空間
- `model/esresnet/`: `ESResNeXtFBSP`ほか、音声エンコーダ(base.py基底、attention.py、fbsp.py)
- `model/clip/`: OpenAI CLIP実装(clip.py, model.py)
- `ignite_trainer/`: PyTorch-Ignite学習フレームワーク(_trainer.py, _interfaces.py, _visdom.py, _utils.py)
- `utils/datasets/`: ESC-50/US8Kデータセット読込(esc50.py, us8k.py)
- `utils/transforms.py`: 音声前処理(ToTensor1D, RandomCrop, RandomPadding)
- `assets/`: AudioCLIPチェックポイント格納先
- `protocols/`: README のみ
- `requirements.txt`: AudioCLIP固有依存

## avlmaps/utils/ — 共通ユーティリティ

- `mapping_utils.py`: 座標変換・マップI/O。cvt_pose_vec2tf, load_depth_npy, depth2pc, transform_pc, base_pos2grid_id_3d, project_point, load_3d_map, save_3d_map
- `clip_utils.py`: CLIPテキスト/画像処理。multiple_templates, match_text_to_imgs, get_text_feats, get_lseg_score
- `lseg_utils.py`: LSeg特徴抽出。get_lseg_feat(マルチスケールcrop-and-concatenate)
- `audio_utils.py`: 音声処理。generate_video_from_rgbs, select_audio_frames, segment_audio_with_silence, encode_audio_batch
- `audio_mapping_utils.py`: 音声マップ作成。create_audio_map_batch, create_audio_map_statistics
- `index_utils.py`: インデックス・検索補助。find_similar_category_id(OpenAI LLM使用), get_segment_islands_pos, get_lseg_score
- `habitat_utils.py`: Habitat API。make_cfg, make_sensor_spec, tf2agent_state, agent_state2tf, display_sample, get_position_floor_objects
- `navigation_utils.py`: パス計画補助。build_visgraph_with_obs_map, plan_to_pos_v2
- `localization_utils.py`: 画像ローカライゼーション。`ImageDataset`, `HLocLocalizer`(hloc統合)
- `category_assigner.py`: `CategoryAssigner`、matplotlib GUIカテゴリ割当
- `visualize_utils.py`: 可視化(Open3D/OpenCV)。visualize_rgb_map_3d, get_heatmap_from_mask_3d, convert_heatmap_to_rgb, pool_3d_label_to_2d
- `esc50_utils.py`: `ESC50Meta`、ESC-50カテゴリ↔パスマッピング
- `matterport3d_categories.py`: MP3Dセマンティックカテゴリ定義(mp3dcatリスト)
- `hm3dsem_categories.py`: HM3Dセマンティックカテゴリ(空)
- `llm_utils.py`: OpenAI LLM言語解析。parse_object_goal_instruction, parse_spatial_instruction
- `sound_nav_collect_utils.py`: 音声ナビデータ収集補助。load_meta, load_audio_video_gt_cat_and_poses, draw_audio_video_gt_on_map
- `task_utils.py`: 空ファイル
- `time_utils.py`: `Tic`クラス、処理時間計測

## config/ — Hydra設定

- `map_creation_cfg.yaml`: マップ作成メイン設定。defaults(data_paths/map_config/params/sound_data_collect_params/sound_config)
- `map_indexing_cfg.yaml`: マップ検索設定。decay_rate, image_query_cfg(camera_height, resolution)
- `data_paths/default.yaml`: データパス。habitat_scene_dir, avlmaps_data_dir, esc50_meta_path, esc50_audio_dir
- `map_config/vlmaps.yaml`: マップパラメータ。map_type, pose_info(pose_type, camera_height, base2cam_rot), cam_calib_mat, grid_size, cell_size, depth_sample_rate, customize_obstacle_map, obstacle_names
- `params/default.yaml`: ナビ/マップパラメータ。gs, cs, forward_dist, turn_angle, sound_map_config, controller_config
- `sound_config/sound_config.yaml`: ESC-50カテゴリマッピング。difficulty(level_1〜5), major_categories
- `sound_data_collect_params/sound_collect_default.yaml`: 音声データ収集パラメータ。difficulty, fps, sample_rate, silence_duration_s, silence_threshold

## ルート直下

- `install.bash`: 環境構築。pip install -r requirements.txt, conda install habitat-sim, hloc(Hierarchical-Localization)clone
- `download_checkpoints.bash`: モデルチェックポイント取得。AudioCLIP-Partial-Training.pt(wget)、LSegチェックポイント(gdown)
- `requirements.txt`: 主要依存(clip, torch-encoding, opencv, pytorch-lightning, hydra-core, librosa, sentence_transformers, openai等)
- `setup.py`: パッケージインストール設定(name: avlmaps)
- `pyproject.toml`: black/isort設定
- `.pre-commit-config.yaml`: pre-commit hook設定
- `README.md`: プロジェクト概要・チュートリアル
- `docs/setup.md`: セットアップガイド

## データフロー

マップ作成時:

- RGB/Depthフレーム → LSeg → ピクセルレベル特徴(H5DF grid_feat)。VLMapBuilder.create_mobile_base_map()内でdepth2pc→3D点群、base_pos2grid_id_3d→グリッドセル割当、特徴集約
- RGB/Depthフレーム → AudioCLIP → 音声特徴(pickle audio_data)。SoundMap.create_sound_map()
- フレーム → CLIP(ViT-L/14) → フレームレベル特徴(H5DF clip_sparse_map)。AreaMap.create_map()
- フレーム → HLoc → 参照ポーズ(COLMAP db)。VisualMap.create_and_load_map()

検索/インデックス時:

- クエリ(テキスト/音声/エリア) → find_similar_category_id(OpenAI LLM) または CLIPスコア(query_feat @ grid_feat.T)
- get_heatmap_from_mask_3d()で距離減衰ヒートマップ生成 → interactive_map.pyで可視化

座標変換チェーン: Habitatシミュレータ座標 → (habitat_lang_robot.py)Habitatグローバル座標 → (dataloader.from_habitat_tf())マップフルグリッド座標 → (dataloader.to_cropped_map_pose())クロップマップ座標(可視化用)

## 実機研究向け分類

実機実験前提。シム依存排除対象/改修対象/そのまま流用対象 切分け。上記各セクション役割は維持、分類タグのみ追加視点。

### シム専用（実機で丸ごと不要）

- `avlmaps/robot/habitat_lang_robot.py`: HabitatLanguageRobot本体、シム初期化処理ごと不要
- `avlmaps/task/habitat_task.py` / `habitat_object_nav_task.py` / `habitat_spatial_goal_nav_task.py`: シム内タスク評価・成功判定、実機評価には別枠組必要
- `avlmaps/utils/habitat_utils.py`: make_cfg等シム設定一式
- `avlmaps/utils/sound_nav_collect_utils.py`: シム内音声ナビデータ収集専用
- `avlmaps/map/interactive_map.py`: habitat_sim.Simulator/AgentState/GreedyGeodesicFollower呼出箇所(get_habitat_robot_state, load_habitat_scene_mesh等)。matplotlib topdown描画・ユーザ入力受付自体は汎用、habitat_sim依存部分のみ切離要
- `application/index_map.py`: 冒頭habitat_sim.Simulator初期化・シーンロード部分。検索UI本体(オブジェクト/音声/エリア/画像クエリ)は汎用
- `avlmaps/navigator/`, `avlmaps/controller/`: DiscreteNavController等move_forward/turn_left/turn_rightがHabitat離散行動空間前提。visibility graph+Strategyパターンの設計は流用可、実装は実機actuator向け作り直し必須

### 要改修（Habitat pose前提だが実機でも同種処理は必要→入力元差替で流用）

- `avlmaps/dataloader/habitat_dataloader.py`(`VLMapsDataloaderHabitat`): クラス名Habitat前提だが中身はロボットpose↔マップグリッド座標変換全般(from_habitat_tf, to_full_map_pose等)。実機でも必須処理→リネーム+pose入力元差替で流用可
- `avlmaps/map/vlmap_builder.py`(106-132行), `vlmap_builder_multi_floor.py`(77-142行): `habitat_base_pose`変数、`habitat2cam_rot_tf`回転定義。Habitat座標規約(Y-up)前提の回転行列→実機センサー座標規約に合わせ再定義要、変換の枠組自体(create_mobile_base_map/create_camera_map)は必須のまま
- `avlmaps/map/area_map.py`(85行): pose変換部の変数名のみhabitat、ロジック自体は汎用
- `avlmaps/utils/audio_mapping_utils.py` / `localization_utils.py`: `get_cam_pose_habitat()` — Habitat pose file形式パース専用。実機ではpose取得元(SLAM/odometry等)に応じ置換要、後続の音声マップ作成・画像ローカライゼーション処理自体は必要
- `avlmaps/utils/audio_utils.py`: 上記VLMapsDataloaderHabitat使用箇所(433, 462行)、それ以外(generate_video_from_rgbs等)は汎用

### 共通コア（シム非依存、実機研究の中心）

- `avlmaps/map/map.py`, `vlmap.py`, `vlmap_multi_floor.py`, `sound_map.py`, `area_map.py`(コア部分), `visual_map.py`
- `avlmaps/lseg/`, `avlmaps/audioclip/`(全体)
- `avlmaps/utils/clip_utils.py`, `lseg_utils.py`, `mapping_utils.py`, `index_utils.py`, `visualize_utils.py`, `category_assigner.py`, `esc50_utils.py`, `llm_utils.py`, `time_utils.py`
