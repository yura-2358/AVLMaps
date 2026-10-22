# ステップ3: vlmap_builder動作確認 調査結果

strategy.mdステップ3詳細化。着手前調査メモ。

**※2026-10-22注記（HSR時代記録）**: 構築手順・環境（PYTHONPATH・`__init__.py`問題・実行スクリプト）はロボット非依存で有効。生成物`vlmaps.h5df`はHSR収集データ→sobit_eduデータで再構築要。sobit_edu時は`overrides=["map_config=sobit_edu"]`（config新規要）、strategy.md「sobit_edu移行の要対応事項」節。

## 呼出し経路

`application/create_map.py`（Hydra CLI）→ `AVLMap.__init__`（VLMap/SoundMap/AreaMap/VisualMap全部初期化）→ `avlmap.create_map(data_dir)` → `VLMap.create_map` → `VLMapBuilder.create_mobile_base_map()`（vlmap_builder.py:54）。

## 未整備事項（着手前に要対応）

### 1. データ配置構造不一致

`create_map.py`は`<avlmaps_data_dir>/avlmaps_dataset/<scene_dir>/`下から`scene_id`番目のシーンを拾う設計（create_map.py:14-17）。`Map._setup_paths`は`data_dir`直下の`rgb/`・`depth/`・`poses.txt`を見る（map.py:41-50）。

今の`avlmaps/dataset/`はフラット構造（rgb/depth/poses.txt直下）→ `avlmaps_dataset/<シーン名>/`の1段ネスト要。※2026-09-28注記: `habitat_dataloader.py:156`（`main()`）は`avlmaps_data_dir/vlmaps_dataset/`を見る→コード側で置き場2系統（`avlmaps_dataset`/`vlmaps_dataset`）。`application/`・`main()`経由で使う場合は両方の整合要。`config/data_paths/default.yaml`の`avlmaps_data_dir`も元のダミーパス（`/home/huang/...`）のまま未変更（今回diffはesc50パスのみ）。

### 2. map_config切替要

`map_creation_cfg.yaml`デフォルトは`map_config: vlmaps`（Habitat向け）。`hsr.yaml`使うにはHydra CLIで`map_config=hsr`明示要る。

### 3. `hsr.yaml`の`cam_calib_mat`未解消TODO

ダミー値`[540,0,540,0,540,360,0,0,1]`のまま（hsr.yaml:19）。実機`camera_intrinsics.txt`実測値（fx=535.60 fy=535.90 cx=313.44 cy=238.39）に差替要。※2026-09-30注記: 2026-09-07差替済み（下記「実施済み」節）。

### 4. LSegチェックポイント

初回`gdown`でGoogle Driveから`demo_e200.ckpt`自動DL試行（vlmap_builder.py:240-245）。コンテナ内ネット疎通・既存配置有無未確認。

### 5. AVLMap全体初期化の副作用

`create_map.py`経由だとSoundMap/AreaMap/VisualMapも同時初期化される（avlmap.py:24-35）。当時（2026-09-07）方針上SoundMap（音声）実機接続は後回しのはずだが、`AVLMap.__init__`は無条件でSoundMap初期化走る。音声データ未整備でコンストラクタが落ちないか未確認 → VLMap単体で先に動作確認する方が安全（ユーザー承認済み、下記決定事項参照）。※2026-09-23訂正: 「SoundMap後回し方針」は提案手法側の「Dynamic Map後回し」の誤適用（下記「重大発見」節末尾注記参照）。VLMap単体先行の判断自体（初期化副作用回避）は有効。

### 6. GPU有無

LSeg推論、CPU実行だとかなり遅い。コンテナ内CUDA利用可否未確認。

## 重大発見: TFテストとの依存関係逆転

strategy.md当初計画は「TFテスト→vlmap_builder」の順（TFテストで座標変換の正しさ確認してからマップ構築、という想定）。

だが`habitat_dataloader.py`の`main()`（habitat_dataloader.py:155-158）が呼ぶ`VLMapsDataloaderHabitat.__init__`は、`map=None`のとき内部で`Map.create(map_config)` → `self.map.load_map(data_dir)`を実行する（habitat_dataloader.py:53-64）。`load_map`は`<data_dir>/vlmap/vlmaps.h5df`が存在しないと`False`返し、`assert load_success == True`で落ちる（vlmap.py:50-55）。

つまりTFテストは「vlmap_builderで一度マップ作成済み」が前提。順序、コード依存的に逆。

さらに`main()`が読む`config/test_config.yaml`自体が存在しない（`find`で確認、0件）。新規作成要。※2026-09-30注記: 2026-09-16新規作成見送り決定、`main()`不使用（docs/resolved.md R6（旧strategy.md「実施記録」節「TFテスト」）、docs/step2.md）。

→ **vlmap_builder実行が実質先決め**。strategy.md「次セッション着手事項」順序訂正済み（※2026-09-28注記: 同節は2026-09-23再構成で廃止、現在はstrategy.md B2（TFテスト）・B3（VLMap構築）・docs/resolved.md R6・R7（旧「実施記録」節））。

## 決定事項（ユーザー承認済み、2026-09-07）

- 順序: vlmap_builderを先に実行 → その後、生成済みマップに対しTFテスト実施
- 確認範囲: まずVLMap単体で動作確認（AVLMap全体は使わない）。理由: SoundMap等が音声データ未整備で`__init__`時点で落ちるリスク未検証、当時のSoundMap実機接続後回し方針とも整合（※2026-09-23訂正: 後回し方針はベースラインに適用されない、上記5.注記参照）
- config修正（データ配置変更・map_config切替・cam_calib_mat差替等）: 1件ずつ提示・承認得てから実施

## 実行方式決定・着手状況（2026-09-07）

案A（scratchpadスクリプトでVLMap直接呼出、hydra.compose使用、`avlmaps/dataset/`をそのままdata_dirに渡す）で進める方針決定。VLMap単体確認の方針（決定事項参照）と自然に合致するため。

### 実施済み

- `hsr.yaml`の`cam_calib_mat`を実測値に差替済み: `[535.600, 0, 313.444, 0, 535.901, 238.388, 0, 0, 1]`（`avlmaps/dataset/camera_intrinsics.txt`実測値、2026-09-07収集分）

### ブロッカー（訂正版、2026-09-07再調査）

**訂正**: 「Dockerコンテナ外で動作中」は誤判定だった。`/.dockerenv`存在確認済み、コンテナ内で確定。`docker`コマンドが無かったのはDocker-in-Dockerでないだけで無関係。

再調査で判明した実際の問題は2点:

**1. PYTHONPATH汚染によるtorchバージョン不整合**

ログインシェルでROS2 colcon workspace（`/opt/ros/jazzy/...`、`/home/yura/colcon_ws/install/...`）と`/opt/pytorch/.venv/lib/python3.12/site-packages`（Python3.12用torchビルド）が`PYTHONPATH`に積まれている。`avlmaps` conda環境（Python **3.8.20**）を`conda activate`してもこれがsys.path前方に残り、Python3.12用torchを誤って読みにいって`AttributeError: module 'typing' has no attribute '_SpecialGenericAlias'`で落ちる。

**対処法確認済み**: `unset PYTHONPATH`してから`conda activate avlmaps`すればtorch(2.4.1、conda env自体のもの)は正常import可能。avlmaps環境を使う作業では毎回必須。

**2. `avlmaps`環境自体の依存不足**

`requirements.txt`記載の依存が完全インストールされていない。全31件をimportテストした結果:

※2026-09-28注記: 件数表記と列挙数が不一致（OK列挙12件、未インストール列挙20件、9+15≠31）。列挙内容を正とし、件数は未再集計。strategy.md「環境状況」節は本節参照のみ（2026-10-22整理）。

- **OK**（9件、torch/torchvision含めれば11件）: `h5py`・`matplotlib`・`pytorch-lightning`・`opencv-python`（cv2）・`librosa`・`imageio`・`imageio-ffmpeg`・`networkx`・`notebook`・`gdown`・`torch`・`torchvision`
- **未インストール**（15件）: `hydra-core`（→`hydra`）・`omegaconf`・`open3d`・`clip`（openai/CLIP）・`shapely`・`grad-cam`（→`pytorch_grad_cam`）・`mpl_point_clicker`・`google-cloud-speech`・`timm`・`wandb`・`sentence_transformers`・`pyvisgraph`・`altair`・`wav2clip`・`tensorboardX`・`test-tube`（→`test_tube`）・`PyQt5`・`torch-encoding`（→`encoding`）・`ai2thor`・`rospkg`

インストール方針は後述の「重大発見: `avlmaps/map/__init__.py`一括import問題」を踏まえて決定（案a採用、下記参照）。

## 重大発見: `avlmaps/map/__init__.py`一括import問題

`avlmaps/map/__init__.py`（avlmaps/map/__init__.py:7-11）が`VLMap`だけでなく`AreaMap`・`VisualMap`・`SoundMap`・`VLMapMultiFloor`を無条件で一括importする構造。Pythonの仕様上、`from avlmaps.map.vlmap import VLMap`と書いても`avlmaps.map`パッケージ全体（`__init__.py`）が先に実行されるため、**VLMap単体のつもりでも他4クラスの依存が全部読み込まれる**。

importチェーンを静的に追った結果:

| 経路 | 追加で必要になる依存 | 備考 |
|---|---|---|
| VLMap本体（vlmap.py, map.py, vlmap_builder.py, 関連utils） | clip, cv2, torch, torchvision, omegaconf, scipy, gdown, tqdm, h5py, matplotlib, PIL, open3d, shapely, pyvisgraph, openai, pandas | pip installで完結する範囲 |
| **SoundMap**（AudioCLIP経由） | `model.clip`・`model.esresnet`という**絶対import**（avlmaps/audioclip/model/audioclip.py:6-8）。`avlmaps/audioclip`自体をsys.pathに追加しないと解決しない設計 | pip installでは解決不可、sys.path細工必須 |
| **VisualMap**（HLocLocalizer経由） | `hloc`・`pycolmap`・`third_party.SuperGluePretrainedNetwork`（avlmaps/utils/localization_utils.py:14-30） | `third_party`はAVLMapsリポジトリ内に存在しない。`hloc`は別途存在確認済みの`/home/yura/Hierarchical-Localization`を指す想定。`pycolmap`はCOLMAP Pythonバインディングで単純pip installでは入らないことが多い |
| VLMapMultiFloor（vlmap_builder_multi_floor.py含む） | VLMap本体と同じ範囲、追加なし | 問題なし |

※2026-09-22訂正（SoundMap行）: 「sys.path細工必須」は誤り。`avlmaps/audioclip/__init__.py`が`sys.path.insert`内蔵、自動解決（docs/resolved.md R12、旧strategy.md「SoundMap問題の解決・封印解除」節）。実際の不足は`wav2clip`・`pytorch-ignite`・`visdom`等pip依存のみ。

VisualMap・SoundMapの依存は、CLAUDE.md方針（Habitat/Matterport3D不使用）や当時のSoundMap実機接続後回し方針と噛み合わない重さで、今回のvlmap_builder動作確認（VLMap・AreaMap、Object/Area Localizationのみ）には本質的に不要。

※2026-09-23訂正: (1)「SoundMap実機接続後回し方針」の根拠は提案手法（study側）の「Dynamic Map後回し」でありベースライン`SoundMap`には適用されない（2026-09-22封印解除、strategy.md参照）。(2) 実際に構築したのはVLMapのみ（下記実行スクリプト参照）、AreaMapは未構築（strategy.md B5）。

### 対応方針決定（2026-09-07、案a採用）

検討した3案:
- **案a（採用）**: `avlmaps/map/__init__.py`を一時的に編集し、必要クラスのみimportするよう絞る。コード改変につき実施時は変更内容を明示
- 案b: 重量級依存（hloc, pycolmap, third_party.SuperGluePretrainedNetwork, AudioCLIP用sys.path設定）も含め全部揃える。非採用（現実的でない・CLAUDE.md方針と矛盾）
- 案c: `importlib`で`avlmaps.map.vlmap`モジュールを`__init__.py`実行バイパスして直接ロード。非採用（`sys.modules`へのダミーパッケージ注入が要る等、技巧的で保守性低い）

案a採用によりVLMap単体確認に必要な依存は「pip installで完結する範囲」（表の1行目）のみに絞られる。次にインストールする不足分は以下の想定（hloc/pycolmap/AudioCLIP関連は案a採用により対象外）:
- `hydra-core`・`omegaconf`・`open3d`・`clip`（openai/CLIP、`git+https://github.com/openai/CLIP.git`）・`shapely`・`pyvisgraph`・`openai`

`pandas`は`avlmaps/lseg/modules/models/lseg_net.py`が使用（要確認、requirements.txt未記載の可能性あり）。→ 下記「実行完了」節でinstall済み、解消。

## ✅実行完了（2026-09-14）

※2026-09-30注記: 生成物`vlmaps.h5df`のmtimeは2026-09-11 16:07。実行日は09-11の可能性、本見出し日付は記録日か誤記か未確定（docs/resolved.md R7）。

`avlmaps/map/__init__.py`編集済み（AreaMap/VisualMap/SoundMap import・`__all__`からコメントアウト、VLMapMultiFloorはVLMap本体と同依存範囲のため残置）。※現状（2026-09-28）: SoundMapは2026-09-22封印解除済み。AreaMap・VisualMapはコメントアウトのまま→`AVLMap`import不可の原因（strategy.md B5・B7・B8）。

不足パッケージインストール済み（`avlmaps`環境）: `hydra-core` `omegaconf` `open3d` `shapely` `pyvisgraph` `openai` `pandas` `clip`(`git+https://github.com/openai/CLIP.git`)。

**追加判明**（step3.md記載の想定に無かった分）: `timm`が`avlmaps/lseg/modules/models/lseg_vit.py`で必要、importエラーで判明・追加インストール済み。

下記スクリプトを実行、成功:

- LSegチェックポイント（`demo_e200.ckpt`、338MB）: 既存配置済み確認、gdown DL不要だった
- GPU利用可否: `torch.cuda.is_available()` True、RTX 3070 Laptop GPU検出
- 実処理時間: 122フレーム分LSeg推論、GPU利用で約65秒（1.7〜2.0it/s）
- 生成物: `avlmaps/dataset/vlmap/vlmaps.h5df`（約73MB）
- warningのみ（`torch.load weights_only=False`の`FutureWarning`）、エラーなし

→ vlmap_builder動作確認完了。次アクションはstrategy.md記載の通りTFテストへ。

### 実行したスクリプト（案A、実行・成功済み）

```python
import sys
from pathlib import Path

REPO_ROOT = Path("/home/yura/colcon_ws/src/AVLMaps")
sys.path.insert(0, str(REPO_ROOT))

from hydra import initialize_config_dir, compose
from avlmaps.map.vlmap import VLMap

CONFIG_DIR = str(REPO_ROOT / "config")
DATA_DIR = REPO_ROOT / "avlmaps" / "dataset"

with initialize_config_dir(version_base=None, config_dir=CONFIG_DIR):
    config = compose(config_name="map_creation_cfg.yaml", overrides=["map_config=hsr"])

print("map_config loaded:")
print(config.map_config)

vlmap = VLMap(config.map_config, data_dir=str(DATA_DIR))
vlmap.create_map(DATA_DIR)

print("Done. Map saved to:", DATA_DIR / "vlmap" / "vlmaps.h5df")
```
