# colcon build 不整合問題

## 症状

`colcon build` 実行 → hsrb_parts_description 失敗、他5パッケージ Aborted。

```
Traceback (most recent call last):
  File "/opt/ros/jazzy/share/ament_cmake_core/cmake/core/package_xml_2_cmake.py", line 22, in <module>
    from catkin_pkg.package import parse_package_string
ModuleNotFoundError: No module named 'catkin_pkg'
CMake Error at /opt/ros/jazzy/share/ament_cmake_core/cmake/core/ament_package_xml.cmake:95 (message):
  execute_process(/home/yura/miniforge3/bin/python3
  /opt/ros/jazzy/share/ament_cmake_core/cmake/core/package_xml_2_cmake.py
  /home/yura/colcon_ws/src/hsrb_common/hsrb_parts_description/package.xml
  /home/yura/colcon_ws/build/hsrb_parts_description/ament_cmake_core/package.cmake)
```

Summary: 1 failed(hsrb_parts_description) / 5 aborted(audio_utils_msgs, odas_ros_msgs, avlmaps, study, sobits_interfaces)

## 原因

conda `base` env が active 状態（`CONDA_DEFAULT_ENV=base`, python3.14.6, miniforge3 が PATH 先頭）で colcon build 実行。

- cmake が Python3_EXECUTABLE に PATH 先頭 python3（miniforge3 base）採用
- base conda env に `catkin_pkg` 未インストール（system `/usr/bin/python3` には apt 由来で有）
- hsrb_parts_description 等 ROS2 標準パッケージの ament_cmake package.xml パース処理が catkin_pkg 必須 → 失敗

確認コマンド:
```
which python3        # /home/yura/miniforge3/bin/python3
python3 --version     # Python 3.14.6
python3 -c "import catkin_pkg"   # ModuleNotFoundError
/usr/bin/python3 -c "import catkin_pkg"  # エラーなし
conda env list         # base(active), avlmaps
```

本質: **ROS2ビルド系（system python + catkin_pkg等 apt依存） と AVLMaps実行系（conda avlmaps env + torch等重量依存）が同一シェルでPATH競合**。base conda を avlmaps env に切替えても同種問題再発（conda python 側に ROS2 ament 系 python 依存が無いため）。install.bash 自体は conda env 内実行前提で書かれており、ROS2 ワークスペースビルドとの分離が考慮されていない。

## 解決案

1. **ビルド時のみ conda 抜く（推奨）**
   `conda deactivate` してから `colcon build`。ROS2標準パッケージのビルドは system python + apt の ROS依存で完結。AVLMaps側（ament_python パッケージなら setup.py 程度）もビルド自体は軽量、重量ライブラリは実行時 import のみ。実行時のみ `conda activate avlmaps` に運用分離。
   - 恒久化: `~/.bashrc` の conda init 部分に `conda config --set auto_activate_base false` 追加 → シェル起動時 base 自動 activate 停止。

2. **colcon に Python3 実行パス明示指定**
   `colcon build --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3`
   conda active状態でも cmake に system python 強制。ただし全パッケージ共通引数のため、AVLMaps側 CMakeLists/setup.py が conda python の重量パッケージ import を要求する場合は別途ビルド失敗の可能性あり（要検証）。

3. **base conda env に ROS ビルド依存追加**
   `pip install catkin_pkg empy lark-parser` 等を active env に追加インストール。
   デメリット: ROS2側依存増加のたび同種エラー再発リスク。対症療法であり根本解決でない。python3.14(base) 自体 AVLMaps 要求(python3.12)とも不一致で別途破綻要因残る。

4. **AVLMaps系・ROS標準系のビルド系統分離**
   avlmaps・study 等 AVLMaps 固有パッケージは conda avlmaps env 内で `colcon build --packages-select avlmaps study`、hsrb系 ROS標準パッケージは system env で別途 `colcon build --packages-select hsrb_parts_description ...` 実行。COLCON_IGNORE 等でパッケージ発見スコープ分離も選択肢。恒久策として最も明示的だが運用手順増える。

**推奨順**: 1（+ auto_activate_base false）を軸、必要なら2で個別パッケージ python 明示。3 は応急処置のみ。4 はビルド頻度・運用次第で検討。

## 解決結果（案1採用・2026-09-01）

`conda deactivate` のみでは CMakeCache に前回(condaのminiforge3 python3)実行パスが残存し再発（`which python3` は system 側に戻っても、CMake側は再configureされず旧パス参照）。

対処: 失敗5パッケージ + hsrb_parts_description の build/install キャッシュ削除 → `conda deactivate` 状態で colcon build 再実行 → 12パッケージ全完了。

```
source ~/miniforge3/etc/profile.d/conda.sh && conda deactivate
rm -rf build/hsrb_parts_description build/audio_utils_msgs build/sobits_interfaces build/odas_ros_msgs build/avlmaps build/study \
       install/hsrb_parts_description install/audio_utils_msgs install/sobits_interfaces install/odas_ros_msgs install/avlmaps install/study
colcon build
```

Summary: 12 packages finished（stderr出力3件: avlmaps, study, yamnet_ros — `byte-compiling is disabled, skipping` warningのみ、ビルド失敗でない）

## 恒久化対応（2026-09-01）

`conda config --set auto_activate_base false` 実行 → `~/.condarc` に `auto_activate: false` 反映済み。次回シェル起動時 base conda 自動activateされない（system python がデフォルト）。AVLMaps実行時は `conda activate avlmaps` を明示実行する運用に変更。

## 未決事項

- CMakeCache汚染を避けるため、以後 conda active状態でのビルド実行を避ける運用ルール周知が必要（誤って conda activate 状態でビルドした場合、対象パッケージの build/install キャッシュ削除→再ビルドが必要になる点、チーム内共有推奨）。

---

# 派生問題（2026-09-01・別種3件）

同日、上記解決後の再ビルドで別種エラー2件 + 運用上の注意点1件が判明。conda由来問題との切り分け記録として残す。

## 症状A: audio_utils_msgs symlinkエラー

```
failed to create symbolic link '.../ament_cmake_python/audio_utils_msgs/audio_utils_msgs' because existing path cannot be removed: Is a directory
```

### 原因

`build/audio_utils_msgs/ament_cmake_python/audio_utils_msgs/audio_utils_msgs` が実体ディレクトリ（egg-info等含む中身入り）として残存 → symlink作成処理が「既存パス削除→symlink張る」の削除ステップで失敗（中身入りディレクトリは安全側で無条件rmされない仕様）。

過去ビルドでcopy install相当の処理が一度実体生成 → 後でsymlink installに切替した際の衝突と推定。conda/catkin_pkg問題とは無関係、build内キャッシュ汚染由来のみ。symlink機構自体は正常（同ビルドで他パッケージは成功）。

### 解決

```
rm -rf build/audio_utils_msgs install/audio_utils_msgs
colcon build
```

該当パッケージのbuild/installキャッシュ削除で解消。再発条件はconda環境切替・インストール方式混在等でbuildディレクトリに残骸が残った場合のみ。同一環境・同一方式で統一していれば基本再発しない。

## 症状B: avlmaps setup.py README.md FileNotFoundError

```
File "/home/yura/colcon_ws/build/avlmaps/setup.py", line 8, in <module>
    with open(str(root / "README.md"), "r", encoding="utf-8") as f:
FileNotFoundError: [Errno 2] No such file or directory: '/home/yura/colcon_ws/build/avlmaps/README.md'
```

### 原因

症状Aとは無関係の別種・構造的バグ。[setup.py](../setup.py)が`Path(__file__).parent / "README.md"`でREADME読込 → `__file__`はビルド時`build/avlmaps/setup.py`（ament_cmake_pythonがコピーしたもの）を指す。ament_cmake_pythonはPythonソース+setup.pyのみbuild側にコピーし、README.mdはコピー対象外 → 毎回FileNotFoundError。symlink/copyどちらのインストール方式でも再発する。**キャッシュ削除では直らない**。

### 解決（コード修正・ユーザー承認済）

setup.pyのREADME読込をtry/exceptで囲み、読込失敗時は空文字にフォールバック。README.md有無に関わらずビルド通る。

```python
try:
    with open(str(root / "README.md"), "r", encoding="utf-8") as f:
        readme = f.read()
except FileNotFoundError:
    readme = ""
```

## 補足: ツール実行シェルでは auto_activate_base 設定が無効

上記「恒久化対応」（`conda config --set auto_activate_base false`）は対話シェル（`~/.bashrc`のconda init経由）には効くが、Claude Code等が起動する非対話・都度新規シェルには反映されないケースを確認（各コマンド実行のたび`CONDA_DEFAULT_ENV=base`で起動）。この種のツールでcolcon buildする際は毎回明示的に

```
source ~/miniforge3/etc/profile.d/conda.sh && conda deactivate
```

を先頭で実行する必要あり。放置するとconda由来のcatkin_pkg問題（本ファイル冒頭の症状）が再発する。
