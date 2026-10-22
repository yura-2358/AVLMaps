# 退避記録（archive）

他docs（主に`strategy.md`）から外した記述の保管場所。完全削除でなく退避→後から経緯・根拠追跡可能にする。

## 運用ルール

- 対象: スコープ外（study側事項等）・前提と齟齬・実装結果と矛盾・訂正済み履歴 等で元docsから外す記述
- 原文は**改変なし**で転記（コードブロック内）。要約・修正は「退避理由」「現在の正」欄に書く
- 1エントリ = 1まとまり。記載項目: 退避日／出典（ファイル・節・退避時点行番号）／退避理由／現在の正（正しい情報の所在）／原文
- 元docs側で削除後、他docsからの参照（「strategy.md『○○』節」等）は本ファイル該当エントリへ付け替え
- 新しいエントリを末尾に追加（番号連番）

---

## A1. strategy.md「2026-09-23再構成メモ」

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md` 冒頭、7-11行
- 退避理由: 提案手法側区分・順序がベースライン記述に混入していた誤りの訂正履歴。本文側訂正済み→現行計画に不要
- 現在の正: `strategy.md`「現在地」「ベースライン実装順序」B5〜B8

```markdown
**2026-09-23再構成メモ**: 旧版は提案手法側の区分・順序がベースライン進捗記述に混入、以下の誤りあり→訂正済み。
- 「vlmap_builder動作確認でVLMap/AreaMap作成成功」→誤り。構築したのはVLMapのみ、AreaMap一度も未構築（`avlmaps/dataset/`に`area_map/`なし）
- 「SoundMap実機接続は後回し（decisions.md方針通り）」→decisions.mdに該当方針なし。「Dynamic Map後回し」は提案手法側の話で、ベースライン`SoundMap`（AudioCLIPベース）に適用される方針でない
- 「音声パイプライン（録音・DOA・CLAP特徴抽出）はベースライン・提案手法共通基盤」→誤り。ベースライン`SoundMap`はDOA・CLAP不使用。共通なのは録音のみ
- 「SoundMap実機接続＝推奨実装順序5番（study側Dynamic Map接続点）」→5番はODAS（study側）。ベースラインSoundMap実機接続は番号自体なかった→下記B6として新設
```

## A2. strategy.md「study側タスク（参照のみ、本リポジトリスコープ外）」節

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md`、92-98行
- 退避理由: 提案手法（study側）タスク。本リポジトリ＝ベースラインのスコープ外（CLAUDE.md「E. 評価準備: study側事項、本リポジトリでは扱わない」）
- 現在の正: study側リポジトリ（`colcon_ws/src/study`）で管理。根拠となる決定は`decisions.md`決定8・9

```markdown
## study側タスク（参照のみ、本リポジトリスコープ外）

旧「推奨実装順序5・6番」。AVLMaps本体（本リポジトリ）はDOA不使用、提案手法固有処理も置かない。study側との接続点としてのみ記載。

- **ODAS統合（DOA推定）単体検証**（旧5番）: odas_ros出力（音源方向）を既知角度0/90/180/270度で検証（decisions.md決定9）。前後対称位置テストはTAMAGO追加効果検証を兼ねる
- **条件2（サニティチェック用の易しい設定）で通し動作確認**（旧6番）: 提案手法との比較評価。**前提としてベースライン完成（上記B5〜B8）要**
- 評価指標・試行回数・統計検定手法、AVLMaps原論文条件サニティチェック（decisions.md決定8）、実機評価タスク枠組み（Habitat版`avlmaps/task/`代替）
```

## A3. strategy.md「順序入替（2026-09-16決定）」節

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md`「実施記録」内、120-127行
- 退避理由: 当時の理由付けが提案手法側との混同（訂正済み）。唯一有効な理由（ODAS⇔capture_node ALSA競合の緊急性低）もODAS＝study側。順序自体はB4完了で決着済み
- 現在の正: 次アクションは`strategy.md` B5〜B8。`audio_recorder_node.py`実機テストはB6前提
- 被参照: `step2.md:117`、`temporary.md:47`（付け替え済み）

```markdown
### 順序入替（2026-09-16決定）

旧順序（audio_recorder_node.py先・HSRLangRobot後）を逆転、HSRLangRobot+nav2を先行。

**2026-09-23訂正**: 当時の理由付けに混同あり。
- 「現在地はVLMap/AreaMap系完了直後」→実際はVLMapのみ完了、AreaMap未構築
- 「音声パイプライン（録音・DOA・CLAP特徴抽出）はベースライン・提案手法共通基盤、着手はSoundMap実機接続時（推奨実装順序5番、study側Dynamic Map接続点）で足りる」→ベースライン`SoundMap`はDOA・CLAP不使用、共通は録音のみ。5番はstudy側ODASでありベースラインSoundMap接続の番号でない→音声収集検証は今後B6前提としてベースラインタスク扱い
- 理由のうち有効なもの: ODAS⇔capture_node ALSA専有競合の緊急性低い。設計自体「録音は生データ保存→後段オフライン処理」（step1.md記載、ODAS file入力対応済み確認済み）→同時起動運用にならない可能性
```

## A4. strategy.md「SoundMap後回し方針」訂正注記

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md`「環境状況（判明済み）」内、191行（190行`__init__.py`芋づる式import項目のサブ項目）
- 退避理由: 訂正済み履歴。封印解除済み（同ファイル「SoundMap問題の解決・封印解除」節）
- 現在の正: `resolved.md` R12（旧`strategy.md`「SoundMap問題の解決・封印解除（2026-09-22）」節、2026-09-30移設）

```markdown
  - 2026-09-23訂正: 当時「SoundMapも後回し方針」としていたが、根拠とされた方針（「Dynamic Map後回し」）は提案手法側の話でベースライン`SoundMap`には適用されない（下記「SoundMap問題の解決・封印解除」節）
```

## A5. strategy.md ODAS関連の部分記述（B1・B6・残未決事項）

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md` 36行（B1 ch3補正項目の一文）、37行（B1末尾「ALSA専有競合」）、70行（B6前提）、269行（残未決事項ベースライン側の後半）
- 退避理由: ODAS＝提案手法側DOA推定エンジン（decisions.md決定10）。ベースライン`SoundMap`はODAS不使用・同時起動しない→ALSA専有競合はベースラインの前提条件でない
- 現在の正: ch3補正方針（案A、`apply_tamago_ch3_gain_correction()`）はB1/B6に残置。録音基盤＝`capture_node`＋`audio_recorder_node.py`（step1.md）

36行から外す一文:
```markdown
ODAS側補正（0.42倍）はODAS経路限定、
```

37行から外す語:
```markdown
・ALSA専有競合
```

70行（B6前提の1項目）:
```markdown
     - ODAS⇔capture_node ALSAデバイス専有競合、未検証。ただし録音は生データ保存→処理は後段オフライン設計→同時起動運用にならない可能性あり（下記「順序入替」節）
```

269行から外す部分:
```markdown
データ収集ノードの音声記録はODAS（study側）とは別タスク（2026-09-02調査）。ODASはマイクカード直結でROS audio topic不使用、
```

## A6. strategy.md「colcon_ws内 既存資産（AVLMaps外）」のstudy側項目

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md`「環境状況」内、255-257行・259-260行
- 退避理由: ODAS/odas_ros（DOA、study側）、yamnet_ros（決定4＝提案手法側セグメンテーション切替用）、HARK（DOAエンジン選定の不採用記録）、sobits_interfaces（ベースライン無関係）。ベースライン`SoundMap`（AudioCLIP、DOA不使用）と無関係
- 現在の正: `decisions.md`決定4・10。ベースライン関係の`audio_utils`（`capture_node`）項目は`strategy.md`残置
- 被参照: `step1.md:149`（HARK不採用の参照先、付け替え済み）

```markdown
- DOA推定エンジン: ODAS採用決定（decisions.md決定10、study側）。odas_ros、ROS2ラッパー・msgs・wav済み
- `odas_ros`: ODAS ROS2ラッパー。DOA推定の軸（study側）
- `yamnet_ros`: YAMNet ROS2ラッパー実装済み。提案手法側で無音検出→分類ベース切替時（decisions.md決定4）即転用可
```

```markdown
- `HARK_Tutorial_2025`: 不採用決定（2026-09-04）。PS Eye/TAMAGOチュートリアル実施済みだがDOA推定エンジンはODAS採用確定済み、HARKは今後も使わない方針。参考実験流用も対象外
- `sobits_interfaces`: SOBITS自作ロボット向けmsg/srv集。HSR本体制御の代替にはならない（`GetHandToTargetTF`等マニピュレータ系、`SoundDetection.msg`あり）
```

## A7. strategy.md「残未決事項 > study側（参照のみ）」

- 退避日: 2026-09-30（元docs削除済み）
- 出典: `strategy.md`、271-276行
- 退避理由: 提案手法（study側）の未決事項。本リポジトリスコープ外
- 現在の正: `decisions.md`未決事項（評価指標、決定4見直し要否）。ODAS・TAMAGO関連詳細は`step1.md`

```markdown
### study側（参照のみ）
- 評価指標・試行回数・統計検定手法（decisions.md未決事項）
- decisions.md決定4（提案手法側セグメンテーション）根拠見直し要否（決定11起因）
- HSRBマイク構成（直列4マイク＋TAMAGO追加）の前後分離性能実測（ODAS側での検証待ち）
- ODAS file入力: 分析済み（2026-09-04、詳細docs/step1.md）。raw入力側`type="file"`対応済み、ただしヘッダなしraw PCM専用（wavヘッダ非対応）、オフライン投入時は要変換
- TAMAGO DOA精度検証（既知角度0/90/180/270度）未実施。`configuration_tamago.cfg`キャリブレーション済み確認（2026-09-04）
```

## A8. strategy.md「次アクション（2026-09-30）」節

- 退避日: 2026-10-22（元docs削除済み）
- 出典: `strategy.md`「ロボット変更: HSR→sobit_edu」節直後、退避時点85-100行
- 退避理由: 2026-09-30時点の次アクション。項目1（B5）完了、項目4・5（B6前提・実装方針）消化済み、残り（B6判断・B7・B8）は「ロボット変更」節「今後の流れ（案）」・「B5〜B8 必要作業」に統合済み。ロボット変更（HSR→sobit_edu）で順序も再編→現行計画に不要
- 現在の正: `strategy.md`「現在地」「ロボット変更」節「今後の流れ（案）」、「B5〜B8 必要作業」

```markdown
### 次アクション（2026-09-30）

優先順。判断待ちと作業を並行進行。

1. ~~**B5 AreaMap構築（即着手可、ユーザー判断不要）**~~ ✅完了（2026-09-30）
   - `__init__.py`の`AreaMap`コメントアウト解除 → `avlmaps` conda envで`AreaMap.create_map(avlmaps/dataset)`実行 → `.venv`でロード確認（VLMap同一プロセスGPUメモリ含む）
   - 構造上の障害なし: `AreaMap.create_map`はVLMapと同じ`poses.txt`/`rgb/`/`cvt_pose_vec2tf`使用→現dataset流用可見込み。`index_area_2d`は`with_init_cat=False`→OpenAI依存なし
   - 完了後: `HSRLangRobot.setup_map`へAreaMapロード経路追加
2. **B6判断（ユーザー、最大ブロッカー）**
   - ~~最優先: `time_ranges`取得方法~~ → 当面「録音後の人手ラベル」採用（2026-09-30、詳細B6）
   - 次点: 入力マイク・ch／ch3補正、秒→フレーム変換バグ修正要否、音源配置条件、使用音カテゴリ
3. **B7判断（ユーザー）**: VisualMap扱い。B8着手のゲート
4. **B6前提作業（判断と並行可）**: `audio_recorder_node.py`実機起動テスト、ビット深度不整合（S24_3LE vs `signed_16`）解消方式決定 → 推奨案`plughw`＋`signed_16`（recorder改修不要、2026-09-30調査、B6「着手前調査」）
   - 2026-10-20: TAMAGO単体テストPASS、`plughw`＋`signed_16`で解消確認。capture_nodeは`latency_us:=256000`指定必須（既定64000で約1秒停止）。残: HSR接続下で`hsr_data_collector.py`同時稼働・HSR⇔PC時刻同期確認（詳細B6「実機起動テスト結果」）
5. **B6実装方針確定**: 実機用経路`create_audio_map_batch_real`はそのまま実行不可（`setup_paths`未定義・`load_real_pose`未import・`nr`未import、2026-09-30発見）→「変換層＋既存関数修正」ではなく「現行収集形式で同等処理する関数を新規作成」が現実的。雛形はHabitat用経路`create_audio_map_batch`（poses.txt処理がAreaMapと同一、2026-09-30調査、B6「着手前調査」）
6. B8（B5〜B7完了後）
```

## A9. temporary.md（2026-09-23 docs整理用一時メモ）全文

- 退避日: 2026-10-22（元ファイル`docs/temporary.md`削除済み）
- 出典: `docs/temporary.md`全体（退避時点104行）
- 退避理由: 2026-09-23未完了事項A〜E区分・ユーザー判断の元資料。各docs（CLAUDE.md・strategy.md・step1〜4・decisions.md）へ反映済み、役目終了（同ファイル「編集実施状況」節）
- 現在の正: A〜E区分・判断はCLAUDE.md「現状の実装状況」節、進捗は`strategy.md`
- 被参照: `strategy.md`（B5〜B8節末尾「docs/temporary.md」）、`step4.md`（S4-4注記「CLAUDE.md・temporary.md」）→本エントリ指す

````markdown
# ベースライン実装 未完了事項整理（2026-09-23、ドキュメント編集用一時メモ）

docs/全ファイル×コード実状態突合せ、AVLMapsベースライン実装の未完了事項洗出し結果＋ユーザー判断。各ドキュメント（strategy.md・step*.md・CLAUDE.md等）編集方針の元資料。編集完了後、削除or各ドキュメントへ移設想定。

## 全体判断（ユーザー）

- **最大問題=A（ベースライン成立阻害）**。原因: ベースライン（AVLMaps、本リポジトリ）と提案手法（study側、Static Map/Dynamic Map）の話がドキュメント上混在。「Object/Area Localizationのみ先行」「Dynamic Mapは後回し」等、提案手法側の実装順序・区分がベースライン進捗記述に持込まれ、ベースラインとして何が揃い何が欠けているか不可視化してた
- **B（既知バグ）・C（ナビ精度・前提検証）**: 現状動作範囲で問題化してない→明らかにそれが原因とわかる問題発生まで問題扱いしない
- **D（環境・運用）**: 問題だが優先度下げる
- **E（評価準備）**: study側事項。ベースライン実装観点では問題扱いしない

## A. ベースライン成立阻害の未完了（最重要）

### A-1. AreaMap未構築・未接続

- `avlmaps/map/__init__.py`で`AreaMap`importコメントアウトのまま（step3.md案a、2026-09-07絞込み時のまま）。2026-09-22`SoundMap`封印解除時も`AreaMap`は戻さず。コメント上理由「VisualMap非依存だが今回未使用」→依存上の障害なし
- `avlmaps/dataset/`には`vlmap/vlmaps.h5df`のみ。AreaMap出力（`clip_sparse_map.h5df`、`area_map.py:72`）未生成
- **ドキュメント食違い**: strategy.md「推奨実装順序 3.」は「実機データでVLMap/AreaMap作成成功」記載。だがstep3.md実行記録（実行スクリプト含む）はVLMap単体のみ。AreaMap、実際は一度も構築されてない
- `HSRLangRobot.setup_map`はVLMap（`load_scene_map`）＋dataloaderのみロード。ロボット側にAreaMapロード経路なし（`lang_robot.py`/`hsr_lang_robot.py`ともarea関連記述なし）

### A-2. 統合クラス`AVLMap` import不可

- `avlmaps/map/avlmap.py:14`の`from avlmaps.map import VLMap, SoundMap, AreaMap, VisualMap`、`__init__.py`が`AreaMap`・`VisualMap`非公開のため失敗
- 実証（2026-09-23、`/opt/pytorch/.venv`、PYTHONPATHから`/tmp/opencv_pkg_install`除外）:
  ```
  ImportError: cannot import name 'AreaMap' from 'avlmaps.map' (/home/yura/colcon_ws/src/AVLMaps/avlmaps/map/__init__.py)
  ```
- → ベースライン統合ロジック本体（`index_object`/`index_area`/`index_sound`クロスモーダル推論）、実機データで一度も動作してない
- `AVLMap.__init__`/`create_map`/`load_map`は`VisualMap`を無条件初期化・構築・ロードする構造。VisualMap（hloc/pycolmap/`third_party.SuperGluePretrainedNetwork`依存、リポジトリ外依存あり）をベースラインで扱うか否か、扱わない場合`AVLMap`側回避方法、要判断

### A-3. SoundMap実機接続層未実装

- `avlmaps/utils/audio_mapping_utils.py`の`create_audio_map_batch`（Habitatシム動画前提）・`create_audio_map_batch_real`（`rob2cam_tf.txt`等旧実機実験形式、DOA不使用・録音時カメラ位置＝音源位置の簡易設計）とも、現行データ収集ノード出力（`poses.txt`/`timestamps.txt`/`audio.wav`/`audio_timestamps.txt`）と非互換
- ~~AVLMaps方式無音検出（`segment_audio_with_silence`、音量閾値ベース）→SoundMap入力形式への変換層、未実装~~（2026-09-23決定11で無音検出不使用、下記「2026-09-23追加（ユーザー判断）」参照）
- TAMAGO ch3ゲイン補正（`apply_tamago_ch3_gain_correction`、実装済み）の呼出元も未実装
- `config/data_paths/default.yaml`の`avlmaps_data_dir`、旧ダミー値（`/home/huang/hcg/projects/avlmaps/data/vln_data/`）のまま。~~SoundMapは`avlmaps_data_dir`基準でデータ探索する設計~~（2026-09-23訂正: `SoundMap`は保持のみで未使用、実データは`create_sound_map`/`load_sound_map`引数`data_dir`基準。影響は`application/`スクリプト経由時のみ）
- **2026-09-23追加（ユーザー判断）**: AVLMaps方式＝公式コード実機用経路と定義→セグメンテーションは既知区間`time_ranges`入力方式（無音検出不使用）。decisions.md決定11
- 2026-09-23追加発見:
  - 実機用経路`create_audio_map_batch_real`、`nr.reduce_noise`呼出すが`noisereduce`未import・未install→現状NameError
  - `SoundMap.create_sound_map`は`is_real`無視、常にHabitat用`create_audio_map_batch`呼出
  - 実機用経路は`librosa.load(sr=44100)`（mono平均化＋リサンプル）。TAMAGO 8ch/16kHz入力時のマイク・ch選択未決
  - `AVLMap.__init__`は`is_real=False`固定（`avlmap.py:28`）

### A-4. 音声収集ノード実機未検証

- `audio_recorder_node.py`実機起動テスト未実施
- ODAS⇔capture_node ALSAデバイス専有競合も未検証（ただし「録音は生データ保存→処理は後段オフライン」設計→同時起動運用にならない可能性あり、archive.md A3、旧strategy.md「順序入替」節）
- ビット深度不整合: TAMAGOはS24_3LE固定（ハード変更不可）、`audio_recorder_node.py`は`signed_16`のみ対応。`plughw`経由変換で吸収 or recorder側24bit対応、未決
- 従来「SoundMap実機接続時（study側Dynamic Map本実装との接続点）に先送り」扱い。だがSoundMapはベースライン（AVLMaps原論文AudioCLIPベース）構成要素→ベースライン実装タスクとして扱う

## B. 既知バグ（判断: 問題顕在化まで扱わない）

- `lang_robot.py`の`load_sound_map`、`SoundMap.__init__`シグネチャ不一致（呼ぶとTypeError）。`HSRLangRobot.get_sound_pos`使用前に必ず踏む。※A-3着手時に顕在化見込み
- `map.py`の`get_north/south/east/west_pos`角度符号（`get_forward_pos`同種疑い）未検証
- `lang_robot.py`の`move_forward`、`move_to`戻り値return漏れ（軽微）

## C. ナビ精度・前提検証（判断: 問題顕在化まで扱わない）

- マップ収集時と現在でnav2側`map`座標系（使用地図`room1.yaml`・AMCL設定）同一性、未検証
- nav2 recovery毎回1回発生、着地ズレ0.03〜0.14m級、原因未特定（`local_costmap`sensor origin警告との関連も未調査）
- `xy_goal_tolerance=0.15m`→実移動距離が要求より最大0.15m短い
- カメラ水平オフセット（前方-0.0598m・左右+0.022m）無視で運用中、精度影響未検証

## D. 環境・運用（判断: 問題だが優先度低）

- `CYCLONEDDS_URI`プレースホルダー問題恒久修正未着手（毎回`export CYCLONEDDS_URI="file://$CYCLONEDDS_URI_PATH"`で回避中）
- `/opt/pytorch/.venv`使用時PYTHONPATH（`/tmp/opencv_pkg_install`除外）恒久対応未実施
- 正式起動スクリプト（`main()`相当）なし、scratchpadスクリプト方式で運用中

## E. 評価準備（判断: study側事項、ベースライン実装観点では問題扱いしない）

- 評価指標（誤差距離 or recall）・試行回数・統計検定手法
- AVLMaps原論文条件（合成音声）サニティチェック（decisions.md決定8、「余力があれば」）
- 実機評価タスク枠組み（Habitat版`avlmaps/task/`代替）

## ドキュメント編集時論点メモ

- strategy.md「推奨実装順序」、「3. vlmap_builder動作確認 ✅完了（VLMap/AreaMap）」「5. ODAS統合」「6. 条件2通し動作確認」等、ベースラインと提案手法（study側）タスクが同一番号列に混在。ベースライン完成条件（VLMap・AreaMap・SoundMap構築＋`AVLMap`統合推論が実機データで動作）軸に再整理要
- strategy.md・step3.md「AreaMap作成成功」記述、事実と異なる→訂正要
- CLAUDE.md「現状の実装状況」節「完了」「未完了・残課題」も上記A〜E区分・判断に合わせ更新要

## 編集実施状況

- 済（2026-09-23）: decisions.md（決定11・未決事項追加）、strategy.md（ベースライン完成条件軸B0〜B8で全面再構成、study側タスク分離、旧記述の誤り訂正注記）、step1.md・step3.md・step4.md（混同由来記述に訂正注記）
- 済（2026-09-28）: CLAUDE.md（下記相違点C-1〜C-15反映）、strategy.md a案節「マップ構築にtorch-encoding要る」誤記訂正
- 済（2026-09-28）: step1〜4相違点反映（下記「step1〜4相違点」節）、strategy.md B6・実施記録・CLAUDE.md B6・decisions.md決定11へ新発見反映
- 済（2026-09-30）: 優先順位（temporary.md→CLAUDE.md→strategy.md→step1〜4、上位を正）に沿った整合修正。CLAUDE.md（C区分2件追加・ヒートマップ過大記述訂正・時点日付）、strategy.md（`load_sound_map`扱い・`hsrb_library`状況・`raw_file_writer_node`不採用反映・依存件数・行番号・根拠不明日付削除）、step1・3・4（解消済み/参照先不在注記・行番号・0.1m切り分けテスト無効注記）
- 残: C-16（temporary.md自体のガイド記載）は一時メモのため対応せず。本ファイルは役目終了、削除or移設判断待ち

## step1〜4相違点（2026-09-28洗出し・同日反映済み、各ファイルに訂正注記追加）

- 新発見（コード）: #1 実機用経路`create_audio_map_batch_real`、秒→映像フレーム番号で音声スライス（バグ疑い）／#2 固定fps前提 vs 収集データ非等間隔（平均約2.3fps）→strategy.md B6・CLAUDE.md・decisions.md決定11へ反映
- step1: S1-1 `hsr_data_collector`実機未検証（古）／S1-2 CLAUDE.md方針の出所誤記（現decisions.md決定4・9）／S1-3 無音検出との整合記述（決定11）／S1-4 TAMAGO実機確認TODO（古）／S1-5 capture_node例がPS Eye 4ch想定／S1-6 CLAP記述（ベースラインはAudioCLIP）／S1-7 B/C/D案参照先不在
- step2: S2-1 `cam_calib_mat`未差替（古）／S2-2 次アクション`audio_recorder`（順序入替と矛盾）／S2-3 データ置き場2系統（`vlmaps_dataset`/`avlmaps_dataset`）／S2-4 CLAUDE.md座標系方針参照先不在
- step3: S3-1 後回し方針（L31・L52未注記）／S3-2 廃止節参照／S3-3 SoundMap sys.path細工必須（誤）／S3-4 SoundMapコメントアウト（現状解除済み）／S3-5 依存件数不一致（strategy.mdも）／S3-6 pandas要確認（解消済み）
- step4: S4-1 実機検証未実施（古）／S4-2 廃止節参照／S4-3 次アクションとB/C区分判断の食違い／S4-4 `main()`要確認（D区分扱い）／S4-5 Stage2結果未収録（strategy.mdのみ、注記で所在明示）／S4-6 strategy.mdに`move_forward(0.3)`本番・`get_forward_pos`修正未記載→strategy.md実施記録へ追記

## CLAUDE.md相違点（2026-09-28洗出し・同日反映済み）

- 決定11矛盾: C-1 `SoundMap`説明「無音検出は音量閾値ベースのみ」／C-2 フェアネス原則の音声セグメンテーション項（中身は提案手法側決定4、主語不明）／C-3 リポジトリ構成「無音検出等AVLMaps原論文アルゴリズム」／C-4 未完了「無音検出からの変換層」
- 事実誤り・過大: C-5 完了「コア移植（LSeg/CLIP）」（CLIP＝AreaMap側は実機未実行）／C-6 「マップ構築にtorch-encoding要る」（学習用モジュールのみ依存）／C-7 `.venv`オンライン稼働にAreaMap未検証の注記なし
- 欠落: C-8 import節に`AreaMap`コメントアウト記載なし（`AVLMap`import失敗の直接原因）／C-9 現状節にベースライン完成条件・B5〜B8・SoundMap実機経路の新発見・ビット深度不整合なし／C-10 `noisereduce`未install注記なし
- 区分・優先度: C-11 未完了先頭がC区分（nav2精度）、A区分後方or欠落／C-12 既知バグ節にB区分判断未反映／C-13 `audio_recorder`テストがB6前提と未記載
- ガイド: C-14 strategy.md再構成未反映／C-15 決定11参照なし／C-16 temporary.md記載なし（対応不要）
````
