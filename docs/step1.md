# ステップ1: データ収集ノード 調査結果

strategy.mdステップ1詳細化。実装前調査メモ。

**※2026-10-22注記（HSR時代記録）**: 本ファイルのtopic名・起動例・実機結果はHSR上のもの。ロボットはsobit_eduへ変更（decisions.md決定12）。sobit_eduでの`hsr_data_collector.py`起動パラメータ（`/sobit_edu/head_camera/...`、`base_frame:=sobit_edu/base_footprint`）・Depth形式（32FC1）等の差分はstrategy.md「sobit_edu移行の要対応事項」節。出力フォーマット・音声記録方式（案A）・TAMAGO検証結果はロボット非依存で有効。TF許容50msの根拠（HSR base速度）はKobukiで再確認要。本文中「docs/log.md」参照先のHSRログは消失（log.md冒頭注記）。

## 出力フォーマット（既存コード解析済み、変更不可）

`map.py`（`rgb_dir.glob("*.png")` sorted、`depth_dir.glob("*.npy")` sorted、`poses.txt`単一ファイル）準拠必須。

```
scene_dir/
  rgb/000000.png, 000001.png, ...     RGB画像
  depth/000000.npy, 000001.npy, ...   深度画像（npy）
  poses.txt                            1行1フレーム
```

**poses.txt 1行フォーマット**（`cvt_pose_vec2tf`確認済み、`avlmaps/utils/mapping_utils.py:18`）:
```
px py pz qx qy qz qw
```
= `map`→`base_footprint` TFの並進+クォータニオン。

**フレーム順序制約**: rgb連番・depth連番・poses.txt行順、3つとも完全一致必須。`vlmap_builder.py:102`で`zip(rgb_paths, depth_paths, base_poses)`、ファイル名一致でなくzip順依存 → 採番ズレ即バグ。

## ノード仕様

1. RGB-D（カメラ）+ TF（`map`→`base_footprint`）同期購読
2. 各フレーム: RGB→png保存、depth→npy保存（メートル単位）、TF→poses.txt追記
3. 連番インデックス採番、3データソース間ズレ防止（TFタイムスタンプとRGB-Dタイムスタンプのマッチング要）

## 実装前3点、調査結果

| 項目 | 結論 |
|---|---|
| depth単位 | メートル（ユーザー確定済み） |
| カメラ内部パラメータ渡し方 | **hsrb_library対象外**。中身は車輪action server + joint action serverのみ（`hsrb_wheel_action_server.cpp`, `hsrb_joint_action_server.cpp`）、README/launch/package.xml grepでカメラ関連0件。→ カメラドライバ側`camera_info`トピック（`sensor_msgs/CameraInfo`）から取得。ドライバパッケージ名は実機接続後`ros2 topic list`で特定要（未確定） |
| TF許容ズレ | **50ms以内**推奨、超過フレーム破棄。根拠: HSR base移動速度目安0.2〜0.3m/s（マッピング時は低速移動前提）→0.3m/s×0.05s=1.5cm程度、VLMaps典型cell_size(0.05m)と同オーダーに収まる。実装は「RGB-D timestamp基準、対応TFをlookupTransform、ズレ>50msならフレーム破棄」方式 |

## 実装済み

`avlmaps/robot/hsr_data_collector.py`（`HSRDataCollectorNode`）

- topic名 全部ROS2パラメータ化（rgb/depth/camera_info）
- RGB+Depth `message_filters.ApproximateTimeSynchronizer`同期
- TFタイムスタンプ-画像タイムスタンプ差>50ms（`tf_tolerance_sec`パラメータ、可変）でフレーム破棄
- depth encoding自動判定（16UC1→mm/1000でメートル変換、32FC1→そのまま）
- camera_info初回受信時`camera_intrinsics.txt`保存（K行列）
- frame_count駆動連番、rgb/depth/poses.txt順序一致保証
- `timestamps.txt`実装済（2026-09-04）: `frame_idx sec nanosec`（rgb_msg.header.stamp基準）、poses.txt同一タイミングでflush・close

### 起動方法

```bash
python3 avlmaps/robot/hsr_data_collector.py --ros-args \
  -p rgb_topic:=/head_rgbd_sensor/rgb/image_rect_color \
  -p depth_topic:=/head_rgbd_sensor/depth_registered/image_rect_raw \
  -p camera_info_topic:=/head_rgbd_sensor/rgb/camera_info \
  -p map_frame:=map \
  -p base_frame:=base_footprint \
  -p tf_tolerance_sec:=0.05 \
  -p sync_slop_sec:=0.02 \
  -p output_dir:=/path/to/scene_dir
```

デフォルト値、実機`ros2 topic list`確認済み（docs/log.md記録分）に合わせ修正済み（`/hsrb/`プレフィックス無し）。

## nav2起動によるmap frame確保（実機接続確認済み）

`ros2 launch sobits_nav nav2.launch.py`起動確認:
- 起動直後のTF木（`ros2 run tf2_tools view_frames`）: ルートは`odom`、`map`フレーム不在（`odom`→`base_footprint`のみ、43.8Hz配信）
- nav2起動後: map_server（既存地図148×115@0.05m/pix、`dining_table`/`bed_room`/`exit`等セマンティックwaypoint入り）+ amcl起動、初期姿勢(0,0,0)セット
- `ros2 run tf2_ros tf2_echo map base_footprint`で`map`→`base_footprint`配信確認（amcl起動直後数秒はTFバッファ空で一時的に失敗、数秒待てば解消）
- → データ収集ノード稼働には**nav2を先に起動必須**（`map_frame`パラメータのデフォルト値`map`はnav2前提で正しい）

## 残未決事項

- ~~カメラドライバのトピック名~~ → 実機確認済み（docs/log.md）。rgb/depth/camera_info全て`/head_rgbd_sensor/...`配下、コードデフォルト値反映済み
- ~~`map`/`base_footprint`のフレーム名自体~~ → 実機確認済み。nav2起動でmap frame供給されること確認済み（上記セクション）
- 音声記録が未実装（strategy.mdステップ1原文「RGB-D + base TF + 音声」の音声部分、`hsr_data_collector.py`に未反映）→ 調査結果、下記セクション参照
- `hsr_data_collector.py`本体、実機での起動テスト・実データ収集は未実施（コードレビューのみ）。実機起動テスト着手試行（2026-09-05、TAMAGO接続中・HSR本体未接続の状態）: `hsrb_mode`（`hsrb_library/mode_ctr.sh`、169.254.6.x帯を`ip neigh`+ping探索しHSR発見時`CYCLONEDDS_URI`/`ROS_DOMAIN_ID`設定）実行→`HSRB not found`。HSR本体がネットワーク未接続のため`ros2 topic list`等ROS2コマンド自体が失敗（`CYCLONEDDS_URI`が`.bashrc`デフォルトのプレースホルダ`/path/to/cyclonedds_profile.xml`のままでrmw初期化エラー）。HSR実機接続後、`hsrb_mode`実行→`ros2 topic list`でカメラ/TFトピック確認→本ノード起動、の順で再試行要。※2026-09-28注記: 解消済み。2026-09-07実機起動・30フレーム収集検証済み、同日往復122フレーム実データ収集済み（strategy.md B1・docs/resolved.md R2〜R4、docs/log.md）

## 音声記録、ODAS統合とは別タスク（調査済み、2026-09-02）

ODASはマイクカード直結（`odas_ros/.../config/odaslive/pseye.cfg`確認: `interface: {type="soundcard"; card=2; device=0;}`）。ROSトピック経由生audio受取不可 → `hsr_data_collector.py`（rclpyノード、RGB-D/TFをROSトピック購読）とは別プロセス・別経路。

生波形保存の部品は`audio_utils`パッケージ側に既存: `capture_node.cpp`（ALSA/PulseAudio→ROS `audio_utils_msgs/AudioFrame`トピック配信）、`raw_file_writer_node.py`（`AudioFrame`をファイル書出）。ODASとは無関係の独立パッケージ。

別タスクとする理由:
- マイクデバイス専有競合の懸念。ODAS（ALSA直結）と`audio_utils capture_node`同時起動時、排他ロックで片方起動不可の可能性（実機未検証）
- CLAUDE.md検証方針（無音検出方式切替・DOA既知角度単体検証）に必要（※2026-09-28注記: 出所は現在decisions.md決定4・決定9、いずれも提案手法側。ベースライン`SoundMap`は既知区間`time_ranges`入力方式・DOA不使用、決定11。生波形保存はベースラインでも`time_ranges`区間切出し・AudioCLIP特徴抽出に必要）。ODASのSST出力（`OdasSstArrayStamped`: id/xyz/activityのみ）だけ保存だと、後からアルゴリズム差替・再評価不可 → 生波形が要る
- 既存パイプライン設計と整合。RGB-D/TFはposes.txt形式でオフライン`vlmap_builder`が読む構成 → 音声も収集時は生記録、無音検出/DOA/CLAP特徴抽出は後段オフライン適用が筋

## 音声記録方式決定（2026-09-02）

**案A: 連続wav1本＋タイムスタンプログ**に決定。

判断根拠: 現段階は「AVLMapsを実機かつ提案手法の実験環境を模して実装」する基盤構築フェーズ止まり（DOA推定はstudy側Dynamic Mapタスク、ベースラインSoundMapの音声処理はstrategy.md B6で別途。※2026-09-23訂正: 旧記述「AVLMaps原論文方式の無音検出本実装は推奨実装順序5番」は誤り、5番はstudy側ODAS。ベースラインのセグメンテーションは無音検出でなく既知区間`time_ranges`入力方式、decisions.md決定11）。実装最小、発話分断回避（chunk分割案は音声連続性を損なうため不採用）、ODAS file入力側の互換性も見込める。AVLMaps原論文の無音検出（音量閾値ベース）が音声をセグメント単位で扱う設計とも「録音」と「セグメント化（後段オフライン処理）」を分離できる点で整合。※2026-09-28注記: ベースラインのセグメント化は無音検出でなく既知区間`time_ranges`入力（決定11）。「録音とセグメント化の分離」自体はそのまま成立（`audio_timestamps.txt`で`time_ranges`↔録音サンプル位置を対応付け）。

TF対応の設計課題（要対応）: poses.txtは`px py pz qx qy qz qw`のみで**フォーマット変更不可**（タイムスタンプ列なし）。案Aで連続録音した音声から検出した発話時刻→対応TFを引くには、フレームのタイムスタンプを別途永続化する必要あり。poses.txt自体は変更せず、`timestamps.txt`等別ファイルでframe_index↔stamp対応を追加保存する方式で対応（`hsr_data_collector.py`実装済み、2026-09-04）。

未着手TODO:
- ~~`timestamps.txt`（frame_index↔stamp対応）の追加実装~~ → 実装済み（2026-09-04）
- ~~音声連続記録ノード実装~~ → 実装済み（2026-09-04、詳細下記セクション）
- ~~PS Eye/TAMAGO実機動作確認（`arecord -l`カード番号特定、複数デバイス同時サブスクライブ可否）~~ → TAMAGOは2026-09-05実機確認済み（下記「TAMAGO実機検証結果」節）。複数デバイス同時サブスクライブ可否（ALSA専有競合）は未検証、strategy.md B6前提へ移管
- ~~ODASのfile入力対応可否~~ → 分析済み（2026-09-04、詳細下記セクション）、raw入力側対応確認済み

## ODASのfile入力対応可否、分析結果（2026-09-04）

`odas_ros`同梱ODAS本体ソース（`src_hops.c`）確認: `interface_file`ケース実装あり、`raw:`セクション`type="file"`対応済み。公式サンプルcfg（`azimut_cma.cfg`/`azimut_oma.cfg`/`xmos.cfg`/`beam.cfg`）でもraw入力側`type="file"; path="xxx.raw";`の実例確認 → 「raw入力側での対応は今回未確認」の疑問、解消。

ただし制約:
- 実装は`fopen`+`fread`のみ、**wavヘッダのパース処理なし** → ヘッダなしraw PCM専用
- `audio_recorder_node.py`が書く`audio.wav`（wave module生成、RIFFヘッダ付き）をそのまま食わせると先頭44byte分ズレる → オフライン投入にはヘッダ除去変換（ffmpeg/sox等）の前処理必須
- `fS`/`nBits`/`nChannels`自動検出なし、cfg決め打ち → 録音時フォーマットと一致要（次セクション参照）

## 音声連続記録ノード実装（2026-09-04）

`avlmaps/robot/audio_recorder_node.py`（`AudioRecorderNode`）

方針変更: TODO原文「capture_node+raw_file_writer_node活用」→ `raw_file_writer_node`不採用、新規ノード自作。理由: `raw_file_writer_node`はAudioFrame.data生バイト追記のみ、タイムスタンプ破棄・wavヘッダなし → 案A（連続wav1本＋タイムスタンプログ）未達。

- `capture_node`のAudioFrameトピック購読（`audio_topic`パラメータ、`audio_in`デフォルト、remap前提）
- `output_dir/audio.wav`: wave module経由、連続PCM書込
- `output_dir/audio_timestamps.txt`: `sample_offset sec nanosec`（フレーム毎、累積サンプル数＋capture_node算出のheader.stamp）→ hsr_data_collector.pyのtimestamps.txtと同型、発話区間（サンプル範囲）→対応TF/画像フレーム時刻の突合に使用
- format対応: `signed_16`のみ（wave moduleはsigned PCM専用、float/double非対応のため意図的に限定。他format受信時は起動時エラー）
- shutdown時wav/timestampsファイルclose（hsr_data_collector.pyと同パターン）

起動方法:
```bash
ros2 run audio_utils capture_node --ros-args \
  -p format:=signed_16 -p channel_count:=4 -p sampling_frequency:=16000

python3 avlmaps/robot/audio_recorder_node.py --ros-args \
  -r audio_in:=audio_out \
  -p output_dir:=/path/to/scene_dir
```

※2026-09-28注記: 上記`capture_node`例（4ch・`signed_16`・16kHz）はPS Eye想定。TAMAGO（8ch/16kHz/S24_3LE固定）で使う場合の設定（`plughw`経由16bit変換 or recorder側24bit対応）、およびベースライン`SoundMap`入力にどのマイクを使うかは未決（strategy.md B6）。※2026-10-20注記: TAMAGO設定は実機確認済み→`plughw`経由16bit変換で確定（recorder改修不要）。マイク選択は未決のまま。TAMAGO用起動コマンド（検証済み）:

```bash
ros2 run audio_utils capture_node --ros-args \
  -p backend:=alsa -p device:=plughw:CARD=TAMAGO03,DEV=0 \
  -p format:=signed_16 -p channel_count:=8 -p sampling_frequency:=16000 \
  -p frame_sample_count:=1024 -p latency_us:=256000

/opt/pytorch/.venv/bin/python avlmaps/robot/audio_recorder_node.py --ros-args \
  -r audio_in:=audio_out -p output_dir:=<dir>
```

recorderは`setup.py`にentry_pointsなし→`ros2 run`不可、`.venv`のpython直接実行（2026-10-22 strategy.md B6から移設）。`latency_us:=256000`必須（既定64000＝1周期分のバッファでオーバーラン→約1秒で`AlsaException`停止）。詳細resolved.md R17。

未検証のまま残る課題（コードレビューのみ、実機未確認）:
- ODASとcapture_node同時起動時のALSAデバイス専有競合（上記「別タスクとする理由」節の懸念、未解消）。pseye.cfgは`type="soundcard"`直結、capture_nodeも`backend=alsa`で同一デバイス直結の場合、同時open失敗の可能性。回避策候補（dsnoop共有・PulseAudio backend統一・capture_node一本化してODASはROS経由受信）はいずれも実機未検証
- 実機での起動テスト・実データ収集自体未実施 ※2026-10-20注記: 起動テストはTAMAGO単体（HSR非接続）でPASS、60秒録音でwav・`audio_timestamps.txt`整合確認。実データ収集・`hsr_data_collector.py`同時稼働・HSR⇔PC時刻同期は未実施（resolved.md R17）。ODAS同時起動の専有競合は未検証のまま

## TAMAGO実機動作確認、分析結果（2026-09-04）

`odas_ros`内`configuration_tamago.cfg`、**キャリブレーション済み**確認（コード読了のみ、実機未検証）:
- ch0=0度・CCW回転、8ch/16kHz/24bit。`/proc/asound/cards`確認済みコメントあり（card 2, TAMAGO03、`plughw:CARD=TAMAGO03,DEV=0`直結想定）
- `docs/calibration.md`（0度点・90度点2箇所塞ぎ実測→回転方向判定→8ch座標算出手順書）記載のTODOコメント、cfg側で既に解消済み → 配置キャリブレーション自体は完了扱い

残作業はCLAUDE.md方針のDOA精度検証（既知角度0/90/180/270度）そのもの（※2026-09-28注記: 出所は現在decisions.md決定9、study側タスク。HARK不採用はarchive.md A6（旧strategy.md「colcon_ws内 既存資産」節）。準備段階の懸念2点:

1. **ビット深度不整合の芽**: `configuration_tamago.cfg`は24bit（`nBits=24`）前提。`audio_recorder_node.py`は`signed_16`のみ対応（16bit限定、他format起動時エラー）。TAMAGO→ODAS直結なら24bitのまま問題ないが、`capture_node`経由でHSRデータ収集と同一経路に載せる場合は16bit化必要 → cfgの`nBits`も合わせて変更要 ※2026-10-20注記: capture_node経路の16bit化は`plughw`＋`signed_16`で解消確認（strategy.md B6）。録音wav（16bit）をODASへオフライン投入する場合のcfg `nBits`変更要否はstudy側で判断
2. Dockerコンテナ内Linux実行前提（CLAUDE.md）→ TAMAGOのコンテナへの直接USBパススルー可否、別途要確認（HARK_Tutorialの`virtual_TAMAGO_device_generator.sh`はWSL2+Windows専用GStreamerブリッジで別経路のため非該当。HARK自体、CLAUDE.md/strategy.md記載通り不採用決定・以降参照しない）※2026-09-30注記: 2026-09-05解消（コンテナ再起動で認識、下記「TAMAGO実機検証結果」節）

## TAMAGO実機検証結果（2026-09-05）

### USBパススルー、解消確認

コンテナ再起動前: `lsusb`・`/proc/asound/cards`・`snd_usb_audio`ロード済み確認（ホスト側は認識済み）だが`/dev/snd/`にcard2用ノード（controlC2等）なし → `arecord -l`にTAMAGO表示されず。原因推定: TAMAGO接続タイミングがコンテナ起動後のため、`/dev/snd`の静的device mountに未反映（よくあるDocker USBオーディオの罠）。

コンテナ再起動後: `controlC2`・`pcmC2D0c`出現、`arecord -l`にTAMAGO-03（カード2）表示確認。**解消**。恒久対策は不要、TAMAGO接続後にコンテナ再起動する運用で足りる（実機実験当日の手順に反映要）。

### ネイティブHWパラメータ確定

`arecord -D hw:CARD=TAMAGO03,DEV=0 --dump-hw-params`実測: **8ch / 16kHz / S24_3LE固定**、他フォーマット全拒否（`plughw`経由なら変換可）。`configuration_tamago.cfg`の8ch/16kHz/24bit前提と一致確認。ビット深度不整合の懸念（cfg24bit vs capture_node16bit限定）は「TAMAGO側は24bit固定でハード変更不可」の事実が確定、対応要否は下記ch3節と合わせ判断。※2026-10-20注記: `plughw:`＋S16_LE 8ch/16kHzで`arecord`・capture_nodeとも録音成功→`plughw`変換で対応（`hw:`＋S16_LEは`Available formats: S24_3LE`で拒否、再確認）

### ch3感度異常、再現確認

`hw:CARD=TAMAGO03,DEV=0 -f S24_3LE -c 8 -r 16000`で3秒試験録音→ch別RMS解析:

```
ch0=0.000512  ch1=0.000548  ch2=0.000532  ch3=0.001022（約2倍）
ch4=0.000527  ch5=0.000490  ch6=0.000529  ch7=0.000517
```

odas_ros/docs/solution.md記載の既知「TAMAGO ch3感度異常疑い」（ゲイン異常、2.37倍相当、DCオフセット説否定済み）と同傾向、この個体・この環境で再現確認。

※2026-10-20注記: `plughw`＋16bit録音（環境音のみ、3秒・60秒）ではch3比1.0〜1.06、約2倍の異常**再現せず**。条件差（本節: `hw:`24bit直結）あり、原因未特定。音源鳴らして再確認要、ch3補正適用判断（strategy.md B6）前に必須

### 新規懸念: ch3ゲイン補正の適用範囲

odas_ros側補正（`src_hops.c`内`if (iChannel==3) sample *= 0.42f;`）は**ODAS処理経路限定**。`audio_recorder_node.py`（`capture_node`経由の生ALSA録音、ODAS非経由）には未適用 → HSRデータ収集で保存する生波形にch3異常がそのまま残る。DOA推定（ODAS側）は補正済みで問題ないが、生波形からのオフライン再解析（CLAP特徴抽出等）でch3異常が悪影響する可能性、対応要否は解決策検討時に判断。※2026-09-28注記: CLAPは提案手法側。ベースライン側で影響しうるのはAudioCLIP特徴抽出（実機用経路は`librosa.load`で全ch平均→ch3異常が混入しうる）、適用有無はstrategy.md B6で判断。

### 対応方針決定・実装（2026-09-05）

案A（生録音は無補正のまま保存、後段オフライン処理側で補正）に決定、`audio_recorder_node.py`は変更せず。理由・比較検討した他案（B: recorder側リアルタイム補正、C: ODAS raw出力経路経由、D: 対応しない）はstrategy.md該当箇所参照（※2026-09-30注記: strategy.mdにB/C/D案の比較記述は現存せず、旧版にも無し。strategy.md B1は案A方針のみ記載、採用理由の記録なし）。

`avlmaps/utils/audio_utils.py`に`apply_tamago_ch3_gain_correction(audio, ch3_index=3, gain=0.42)`実装。入力は`(n_samples, n_channels)`のndarray、ch3列にのみ係数を掛けて返す。`TAMAGO_CH3_GAIN_CORRECTION = 0.42`をモジュール定数化（odas_ros側校正値と共通）。

検証: 実機録音データ（`tamago_test.wav`、8ch/24bit/16kHz、上記ch3感度異常再現節で使用したもの）に適用。

```
ch0: before=0.000512 after=0.000512（無関係chは不変）
ch3: before=0.001022 after=0.000429（他ch水準0.0005前後に整合）
```

注意: `audio_utils.py`自体はtorch/wav2clip等重依存を伴うファイル（Habitatシム向け既存音声パイプライン集約先）のため、この環境では単体import不可（`wav2clip`未インストール）。動作検証は同一ロジックを独立実行して実施、関数自体はファイルに実装済み。実際のHSRデータ収集パイプラインからの呼び出し箇所は未実装。ベースラインSoundMap（AudioCLIPベース・DOA不使用）側での適用有無はstrategy.md B6で判断（入力マイク・チャネル選択と併せて未決）。DOA推定・CLAP特徴抽出はstudy側Dynamic Mapタスク（旧記述「strategy.md推奨実装順序5番で接続」、2026-09-23訂正: 5番はstudy側ODASでベースライン接続点でない）。
