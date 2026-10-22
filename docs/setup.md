# AVLMaps

[Audio Visual Language Maps for Robot Navigation](https://arxiv.org/pdf/2303.07522.pdf)

Chenguang Huang, Oier Mees, Andy Zeng, Wolfram Burgard

**AVLMAPs**（Audio Visual Language Maps）提案。音声・視覚・言語 各モダリティ横断情報格納 オープン語彙3Dマップ表現。
大規模言語モデル組合せ → 音声・視覚・言語 マルチモーダルプロンプト消費 → 相補的情報源活用しゼロショット空間目標ナビゲーション目標曖昧性解消。

![cover_lady](../media/cover_lady_cropped.png)

# クイックスタート

AVLMaps作成・ランドマークインデックス、Colabで試行可能:
[Open In Colab](https://colab.research.google.com/drive/1gdtLvg_Fbl16N3ITp5FsU9ZAG6HmspVb?usp=sharing)

## 環境セットアップ

自マシンで始める場合、リポジトリclone:
```bash
git clone https://github.com/avlmaps/AVLMaps.git
```

依存関係インストール:
```bash
$ conda create -n avlmaps python=3.8 -y  # or use virtualenv
$ conda activate avlmaps
$ conda install jupyter -y
$ cd AVLMaps
$ bash install.bash
```

## チェックポイントダウンロード

[AudioCLIP](https://github.com/AndreyGuzhov/AudioCLIP)・[LSeg](https://github.com/isl-org/lang-seg) チェックポイント、以下コマンドでダウンロード:

```bash
bash download_checkpoints.bash
```

## データセット生成

シミュレーション環境向けAVLMaps構築に、Habitatシミュレータ+Matterport3Dデータセットで10シーン分RGB-D動画 手動収集済み。
RGB-D動画生成用スクリプト・ポーズメタデータ提供。各シーンにつきポーズ付RGB動画20シーケンスも収集し、ESC-50データセットから音声挿入し音声動画作成。
以下手順でデータセット生成。

### ESC50データセットダウンロード

元ESC50音声データセット、以下コマンドでダウンロード必要。詳細は公式repo参照: [https://github.com/karolpiczak/ESC-50](https://github.com/karolpiczak/ESC-50)

```
wget https://github.com/karoldvl/ESC-50/archive/master.zip -P ~/
unzip ~/master.zip -d <target_dir>
```

展開後ESC-50データセット、`<target_dir>/ESC-50-master` 配下に配置。`config/data_paths/default.yaml` 内パス修正必要:
* `esc50_meta_path` を `<target_dir>/ESC-50-master/meta/esc50.csv` に設定
* `esc50_audio_dir` を `<target_dir>/ESC-50-master/audio` に設定

### Matterport3Dデータセットダウンロード

[Dataset Download](https://niessner.github.io/Matterport/) 確認、[利用規約](http://kaldir.vc.in.tum.de/matterport/MP_TOS.pdf)署名、担当者へHabitatシミュレータ用Matterport3Dメッシュ利用申請送付。
返信メールにダウンロード用pythonスクリプト添付。スクリプトを `~/download_mp.py` として保存。以下実行しデータダウンロード:

```bash
cd ~
# download the data at the current directory
python2 download_mp.py -o . --task habitat
# unzip the data
unzip v1/tasks/mp3d_habitat.zip
# the data_dir is mp3d_habitat/mp3d
```

`config/data_paths/default.yaml` 内パス修正:
* `habitat_scene_dir` をダウンロード済Matterport3Dデータセット `~/mp3d_habitat/mp3d` へ変更

`habitat_scene_dir` 構造例:
```
habitat_scene_dir
  |-5LpN3gDmAk7
  |   |-5LpN3gDmAk7.glb
  |   |-5LpN3gDmAk7_semantic.ply
  |   |-...
  |-gTV8FGcVJC9
  |   |-gTV8FGcVJC9.glb
  |   |-gTV8FGcVJC9_semantic.ply
  |   |-...
  |-jh4fc5c5qoQ
  |   |-jh4fc5c5qoQ.glb
  |   |-jh4fc5c5qoQ_semantic.ply
  |   |-...
  ...
```

### AVLMapsデータセット ダウンロード・生成

`config/generate_dataset.yaml` 設定:
* `defaults/data_paths` の値を `default` へ変更
* `avlmaps_data_dir` をダウンロード先へ変更
* `data_cfg.resolution.w`・`data_cfg.resolution.h` で生成rgb/depth/semantic画像解像度調整
* `rgb`・`depth`・`semantic` を `true` にすると対応データ生成、`false` で無視
* `camera_height` でロボットベース相対カメラ高さ変更

以下コマンドでダウンロード・データセット生成。生成データ 約150GBディスク容量消費。

```bash
# go to <REPO_ROOT>/dataset of this repository
python dataset/generate_dataset.py
```

データ生成後、`avlmaps_data_dir` 構造:
```
avlmaps_data_dir
├── 5LpN3gDmAk7_1
│   ├── poses.txt
│   ├── audio_video
│   │   ├── 000000
│   │   │   ├── meta.txt
│   │   │   ├── poses.txt
│   │   │   ├── output.mp4
│   │   │   ├── output_level_1.wav
│   │   │   ├── output_level_2.wav
│   │   │   ├── output_level_3.wav
│   │   │   ├── output_with_audio_level_1.mp4
│   │   │   ├── output_with_audio_level_2.mp4
│   │   │   ├── output_with_audio_level_3.mp4
│   │   │   ├── range_and_audio_meta_level_1.txt
│   │   │   ├── range_and_audio_meta_level_2.txt
│   │   │   ├── range_and_audio_meta_level_3.txt
│   │   │   ├── rgb
│   │   │   |   ├── 000000.png
│   │   │   |   ├── ...
│   │   ├── 000001
│   │   ├── ...
│   ├── depth
│   │   ├── 000000.npy
│   │   ├── ...
│   ├── rgb
│   │   ├── 000000.png
│   │   ├── ...
│   ├── semantic
│   │   ├── 000000.npy
│   │   ├── ...
├── gTV8FGcVJC9_1
│   ├── ...
├── jh4fc5c5qoQ_1
│   ├── ...
...
```

データ構造詳細は dataset [README](../dataset/README.md) 参照。

## 生成データセットでAVLMap作成

* `config/map_creation_cfg.yaml` 内 `defaults/data_paths` の値を `default` へ変更
* 上記データセット生成手順に従い `config/data_paths/default.yaml` 内 `habitat_scene_dir`・`avlmaps_data_dir` 変更
* 以下コマンドでVLMap構築:
  ```bash
  cd application
  python create_map.py
  ```

### 作成AVLMap設定

* `config/map_creation_cfg.yaml` 内 `scene_id`（0-9）変更しVLMap生成対象シーン変更
* `config/params/default.yaml` パラメータ変更しマップカスタマイズ
  * `cs`（セルサイズ、単位m）・`gs`（グリッドサイズ）変更でマップ解像度調整
* `config/vlmaps.yaml` の `pose_info` セクション変更しカメラポーズ・ベースポーズカスタマイズ
  * `pose_type`: `poses.txt` 内ポーズ種類。現状 `mobile_base`（ベースポーズ）のみ対応。`camera` 実装も可能
  * `camera_height`: ベース相対カメラ高さ。データセット生成時カメラ高さ変更した場合ここも変更
  * `base2cam_rot`: ロボットベース座標系→カメラ座標系（z前方、x右、y下）回転行列、行方向平坦化したもの
  * `base_forward_axis`・`base_left_axis`・`base_up_axis`: ロボットベース座標系定義。前方単位ベクトル[1,0,0]・左方単位ベクトル[0,1,0]・上方単位ベクトル[0,0,1]をベースフレームに射影した座標
* `config/vlmaps.yaml` 他設定
  * `cam_calib_mat`: カメラ内部パラメータ行列、平坦化したもの
  * `depth_sample_rate`: 各フレームで `h * w / depth_sample_rate` 個のランダムサンプルピクセルのみ逆投影

## AVLMapインデックス化

* `config/map_indexing_cfg.yaml` 内 `defaults/data_paths` の値を `default` へ変更
* 上記データセット生成手順に従い `config/data_paths/default.yaml` 内 `habitat_scene_dir`・`avlmaps_data_dir` 変更
* 以下コマンドで構築済VLMapインデックス化:
  ```bash
  cd application
  python index_map.py
  ```
  実行したいインデックス種類、番号入力で指定。

  ![avlmaps_index](../media/avlmaps_github_index.gif)

  **Index Object**: 物体カテゴリ名入力しヒートマップ生成。

  ![avlmaps_index_object](../media/avlmaps_github_index_object.gif)

  **Index Sound**: 挿入音源GT位置示すトップダウンマップ表示。音名入力しヒートマップ生成。

  ![avlmaps_index_sound](../media/avlmaps_github_index_sound.gif)

  **Index Area**: "kitchen"、"bathroom" 等エリア名入力しヒートマップ生成。

  ![avlmaps_index_area](../media/avlmaps_github_index_area.gif)

  **Index Image**: トップダウンマップ表示、地図上2点選択し画像位置・向き取得。選択ポーズ・該当ポーズでのクエリ画像表示後、クエリ画像用ヒートマップ生成。

  ![avlmaps_index_image](../media/avlmaps_github_index_image.gif)

### インデックス設定

`config/map_indexing_cfg.yaml` 変更:
* `decay_rate`: ヒートマップ減衰率設定。小さいほど熱遷移明瞭、広範囲カバー
* `image_query_cfg`:
  * `camera_height` でカメラ高さ設定
  * `resolution.w`・`resolution.h` で画像解像度設定
  * `save_query_image` を True にするとインタラクティブトップダウンマップ内選択クエリ画像保存

## 引用

データセット・コード有用な場合、以下引用:

```bibtex
@inproceedings{huang23avlmaps,
              title={Audio Visual Language Maps for Robot Navigation},
              author={Chenguang Huang and Oier Mees and Andy Zeng and Wolfram Burgard},
              booktitle={Proceedings of the International Symposium on Experimental Robotics (ISER)},
              year={2023},
              address = {Chiang Mai, Thailand}
          }
```

## ライセンス

MIT License

## 謝辞

以下プロジェクト著者陣に感謝。コード公開、AVLMaps研究大いに促進:

* [Hierarchical Localization](https://github.com/cvg/Hierarchical-Localization)
* [AudioCLIP](https://github.com/AndreyGuzhov/AudioCLIP)
* [ESC-50 Dataset for Environmental Sound Classification](https://github.com/karolpiczak/ESC-50)
* [Language-driven Semantic Segmentation (LSeg)](https://github.com/isl-org/lang-seg)
* [Visual Language Maps for Robot Navigation](https://github.com/vlmaps/vlmaps)
