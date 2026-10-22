# 研究背景・関連研究

## 研究の目的

視覚情報はカメラの視野角に限定されるが、音声情報は全方向から同時取得できる。既存のセマンティックマッピング手法（VLMaps、AVLMaps）は静的な環境を前提としており、物体や音源の位置が変化すると誤った推論を引き起こす。本研究では、AVLMapsをさらに拡張し、音声モダリティのみを動的に扱うことで、静的な画像情報と動的な音声情報による半動的なセマンティックマップを構築し、位置推定精度の向上を図る。

## 提案手法のアーキテクチャ

- **Static Map**: 画像情報による2モジュール構成。
  - Object Localization: オープンボキャブラリセグメンテーション（SAM3）で物体点群を取得し世界座標系に記録。ヒートマップは物体からのユークリッド距離に応じて線形減衰。
  - Area Localization: CLIPの画像エンコーダで視覚的特徴量を計算し、カメラ姿勢と共に取得。ヒートマップはCLIP出力をsoftmax正規化した信頼度を用い、xy平面上の距離に応じて線形減衰。
- **Dynamic Map**: 音声情報を逐次的に扱う。リングバッファで一定時間を録音し、YAMNet等で音声検出、無音検出でセグメント境界を判定（後述の通りAVLMaps方式は音量閾値ベース）。セグメントごとにCLAP等の音声・言語事前学習モデルでエンコード。音源定位角度にはvon Mises分布を仮定し、信頼度をsoftmax正規化した上で角度に応じた減衰率を設ける。
- **統合**: StaticとDynamicの情報を、ベイズ推定によるグラフィカルモデルで統合する（AVLMapsのクロスモーダル推論を明示的な確率モデルに置き換えたもの）。
- **メモリ削除（忘却）**: 音響情報は音源停止後に新たな観測が得られなくなる過渡的情報であるため、時間経過による確率減衰モデル（2値消去/線形減衰/エビングハウス忘却曲線のいずれか）を導入し、閾値を下回った情報を削除してメモリ飽和を防ぐ。

## 実験計画（4節より）

- **(A) 動的セマンティックマップの有効性検証**: AVLMapsをベースラインに3条件で比較。
  - 条件1: Static Map作成時は音声なし、Dynamic Mapの音声のみで位置推定。
  - 条件2: マッピング時と同じ場所で音源を鳴らして推定（AVLMapsが有利と予測）。
  - 条件3: マッピング時と異なる場所で音源を鳴らして推定（提案手法が有利と予測）。
- **(B) 時間減衰モデルの有効性検証**: 2値削減/線形時間減衰/エビングハウス忘却モデルの3手法比較（卒検スコープからは外し今後の課題とする方針。`decisions.md`参照）。

## AVLMaps原論文の技術的事実確認（誤解しやすい点の注意）

原論文: C. Huang, O. Mees, A. Zeng, W. Burgard, "Audio Visual Language Maps for Robot Navigation," arXiv:2303.07522, 2023. 公式コード: github.com/avlmaps/AVLMaps（ベースのVLMapsはgithub.com/vlmaps/vlmaps）。

- **使用シミュレータ/ロボット**: シミュレーションはHabitat + Matterport3Dデータセット（MuJoCoではない）。実機実験（Section IV-E "Real World Experiment"）はRidgebackモバイルベース + Pandaマニピュレータ + RealSense D435カメラ（HSRではない）。
- **音声セグメンテーション**: 音量（振幅/エネルギー）ベースの単純な閾値判定による無音検出。分類モデル（YAMNet等）は使っていない。
  > "Whenever the volume is above a threshold, we mark this time step as the starting point of a segment. Whenever the volume of the sound is not larger than this threshold for a certain duration, we end the segment."
- **音源位置の扱い**: シミュレーションでも実機実験でも、音声は収集済みのRGB-D動画に事後的に人工合成されたものであり、物理的にスピーカーで再生し音響伝搬を経て録音したものではない。
  > "we artificially add sounds to the RGB-D video when the robot moves to certain locations."
  論文自身も「マップに記録している位置は真の音源位置ではなく、音が聞こえたとされる場所である」と明記している。
  > "The locations of sounds are not sound source locations but the places where the sounds were heard."
- **音源定位（DOA）**: 論文中に"sound source localization"や"direction of arrival"という用語は登場しない。「audio localization」はマップ上の位置検索を指す語として使われているのみ。DOA推定は本研究で新規に取り組む要素であり、AVLMapsには存在しない。

- **クロスモーダル統合**（2026-09-30確認、arXiv HTML版）: 各モダリティヒートマップ要素積 ℋ_target = ℋ₁ ⊙ ℋ₂ ⊙ … ⊙ ℋ_L、最大確率位置を目標位置とする。LLMが自然言語指示→Pythonコード生成、API: `robot.get_major_map(obj=, sound=, img=)`（減衰率ε=0.1、主目標用）・`robot.get_map(...)`（ε=0.01、補助用）・`robot.get_max_pos_3d(heatmap)`・`robot.move_to(pos)`・`robot.load_image(path)`。**公開コードに上記API・要素積統合・LLMプロンプトいずれも存在しない**（`AVLMap.index_object`/`index_area`/`index_sound`/`index_image`単独クエリのみ、呼出元`application/index_map.py`のみ）→ベースライン統合ロジックは論文記述から再実装要（strategy.md B8）
- **モダリティ別ヒートマップ定義（論文）**: Visual `max(1−ε·dist_xy(p,p_v),0)`／Object `max(1−ε·d_min(p),0)`／Area `max(max_i{s_ai−ε·dist_xy(p,p_ai)},0)`／Audio `max(max_i{s_si−ε·dist_xy(p,p_si)},0)`。**コードとの相違**: Audio、論文は全セグメントmax、コード`index_sound_2d`は加算（`dist_map +=`、`avlmap.py:129`）。Areaは論文・コード一致（max）
- **実機実験詳細（Section IV-E）**: 音声入力に背景ノイズ除去（コード`nr.reduce_noise`対応）。言語ベースマルチモーダルナビ20タスクで成功率50%、マルチモーダル空間目標10件で6/10成功
- **評価指標（シミュ）**: Top-1 recall（許容距離<0.5/1/1.5/2m）・平均最小距離。ナビは停止位置が正解から1m未満でサブゴール成功。比較対象VLMaps・AudioCLIP・wav2clip・ConceptFusion
- **データセット音声挿入仕様**（公式コード`dataset/generate_dataset.py`・`avlmaps/utils/audio_utils.py`、setup.md対応）: 移動中5秒区間（`5*fps`フレーム）単位で挿入、既存音位置全てから2.0m以上離れた区間のみ採用、区間後1秒空ける（2.0m/3.0mハードコード、引数`min_dist_between_sound`/`interval_s`未使用）。カテゴリはシーン内未使用のものから選択→1シーン1カテゴリ1回。正解は`range_and_audio_meta_level_N.txt`（`開始フレーム,終了フレーム,開始秒,終了秒,カテゴリ,wavパス`）。level_1=室内生活音、level_2=+人（非音声）、level_3=+動物
- **Habitat用経路無音検出の実態**: `segment_audio_with_silence`は`audio > silence_thres`（既定0）の符号付き振幅比較、1秒無音で区切る。合成音声の無音部＝完全な0前提で成立、実環境ノイズでは全区間が閾値超→巨大1区間化（決定11の裏付け）
- **AudioCLIP入力**: 各セグメント先頭5秒のみ使用（44.1kHz、220500サンプルへ0詰め）。音位置＝セグメント中全フレームのロボット位置リスト（単一点でない）

これらの事実は、AVLMapsを本研究で実環境録音・実機（sobit_edu、2026-10-22 HSRから変更、decisions.md決定12）・DOA推定を伴う条件で評価することが、原論文にない新しい検証軸を提供するという主張の根拠になる（詳細は`decisions.md`）。卒論・Q&Aで引用する際は、Section IV-E周辺のPDF原文を自身でも直接確認すること。

## 関連研究（2節より）

- VLMaps [1] / NLMap-SayCan [2]: 自然言語と画像情報の統合によるセマンティックマップ構築。
- AVLMaps [3]: VLMapsに音声-言語事前学習モデルを組み込み、音声情報も加味。本研究の直接のベースライン。
- Sai et al. [4]: 動的ウィンドウ・フロンティア探索、TSPの貪欲法による効率的な再訪問、画像ベースの動的セマンティックマップ更新（音声は扱わない）。
- Joana et al. [5]: 人間の忘却プロセスに着想を得たオンライン/オフライン・フィルタリングによる長期記憶の忘却手法。本研究の忘却モデル設計の参考。
