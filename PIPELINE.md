# PIPELINE.md — ツール間の入口/出口マップ

「え、そういう切り口からできたの！？」を偶然でなく設計で見つけるための1枚。
**新しいルートは「出口Aの形式と入口Bの形式が合うか」をこの表で探す。**
ツールの入出力を変えたら必ずこの表を更新すること（各プロジェクトのCLAUDE.mdより先にここ）。

## 入口/出口 一覧

| ツール | 入口（読めるもの） | 出口（書き出すもの） | 撮影処理 |
|---|---|---|---|
| **animator.html** | 画像(REF/下絵) / プロジェクトJSON(IndexedDB) / **2026-07〜: REFパネル `+ JSON / SEQ` に連番画像を複数選択 → 1件のREF ANIMATOR（ファイル名の数字順・1枚=1コマ）** | ANIMATOR_v1 JSON（ファイル名 `animator_<ISO>.json`） / ライブ連携(BroadcastChannel `tdr_live`) | なし（作画に専念） |
| **composer.html** | PROJECT_v2 / PROJECT_v1 / ANIMATOR_v1 / IMAGE_v1(PNG·JPEG·WebP) / audio / ライブ連携 / **SPEC_07: トラックの`Re`でEX_DBから絵を取り直し** / **2026-07〜: 連番画像4枚以上（`^(.*?)(\d{3,5})\.(png\|webp\|jpe?g)$` の同名グループ）はIMPORT・D&Dとも1トラックのシーケンスに自動集約（OBANの `addFiles` と同一判定）** | 連番PNG(zip・作業解像度) / 動画(MP4·WebM) / EXPORT WEB(スクロールビューアHTML) / PROJECT_v2 JSON（ファイル名 `composer_<ISO>.json`） / **SPEC_07: トラックの`ANI`で `animator.html?open=` ディープリンク** | **P0〜: fxチェーン**（VIDEO=rt / PNG=final / EXPORT WEB=rt・P2b〜） |
| **OBAN_BUILDER** | 画像D&D(単品/連番seq・**MANGA PLATEのPNG含む**) / プロジェクトJSON(**2026-07〜: EXPORT JSON / IMPORT JSON＝ファイル入出力に統一。旧 COPY/PASTE PROJ のクリップボード方式は廃止**) / **SPEC_07: + FROM ANIMATOR(EX_DB)＋ライブ連携(`tdr_live`受信・PNG書き出し不要)** | oban-viewer.html(単一HTML・画像は同フォルダ参照・**ap-seqはdataURLベイク同梱**・**SPEC_09 P4: FRAME枠線含む**・**V2-D: 縦書きテキスト/EN字幕(`?sub=0`)/クリックFX含む**) / プロジェクトJSON(**V2-D〜: `texts[]`+`clickFx`含む**) / **P3: COPY FOR COMPOSER(PROJECT_v2=CAMERAトラック+fx+obanPanels配置同梱・クリップボード。composer側で画像を先にIMPORTしておくと名前一致で配置が自動適用=P3b。textsは対象外)** / **SPEC_07 B3: EDIT IN ANIMATOR(`?open=`ディープリンク)** | **P2〜: take.fx→ビューアrt** |
| **manga-plate.html** | REF画像(下敷き・表示のみ) / PLATE_v1 JSON(クリップボード) | 透過PNG(elem別/全体/×4 SEEDS連番) / PLATE_v1 JSON | なし（素材生成に専念） |
| **econte.html** | 紙ネーム/ラフの写真(D&D・IMPORT) / パレットJSON | **動画コンテ WebM/mp4（P1・実時間録画・C#/尺焼き込み可）** / **P2予定: animator REF(`tdr_live`)・カラースクリプト一覧PNG** | なし（プリプロに専念） |
| **oban-viewer.html** | 同フォルダの画像ファイル | （最終出力・スクロールLP） | rt実行時（`?fx=0`でOFF） |
| **EXPORT WEBビューア** | （画像は埋め込み済み） | （最終出力・スクロールLP） | P2b(任意)でrt |
| **AE** | 4K連番PNG | 完成動画 | AE側（fx OFFで持ち込む） |
| **VERIFY_HARNESS** | 検証対象HTML（`window.__HARNESS__`契約実装） / baseline PNG / API録画fixtures | 統合結果JSON(summary.json) / diff PNG / baseline PNG | 検証専用（fx対象外） |

## 共通フォーマット

- **fxスキーマ v1**（SPEC_06 §2）— 全ツール共通・無変換で持ち回り。未知エフェクトはスキップ（前方互換）
- **TAKE**（SPEC_01）— `{kf:[{x,y,z,dwell,ease}]}`。OBAN / LP_Model_CR 系で共通。P3でcomposer CAMERAトラックへ変換可
- **PROJECT_v2** — composerの保存形式。IMPORT JSONは複数回で追加合成できる。**P0〜: トップレベル `fx:`（fxスキーマv1）を同梱**（欠損時は composer 側で `makeDefaultFx()` 補完。PROJECT_v1 単一トラック保存には乗らない）。**2026-07〜: トラック `type:'null'`（描画されない親専用ヌル）、カメラトラックの `parent`（NULLのtid）、KFの任意 `ei`/`eo`（influence% 0-100）を追加**（いずれも旧リーダーでは無視されるだけの後方互換フィールド）

## 成立している制作ルート

1. animator → composer →（fx OFF・連番）→ AE → 動画 …従来
2. animator → composer →（fx final）→ 動画/連番 …AE不要ルート(P0〜)
3. animator → OBAN_BUILDER → oban-viewer(LP) …スクロールコンテンツ(fxはP2〜)。**SPEC_07〜: PNG書き出し不要（+ FROM ANIMATOR＋ライブ同期→EXPORTでdataURLベイク）**
4. animator → OBAN_BUILDER →（P3変換）→ composer →（fx final）→ 動画
5. animator → composer → EXPORT WEB(LP) …fx rt実行(P2b〜)。fx有効時のみコア同梱・`?fx=0`でOFF
6. manga-plate →（透過PNG）→ OBAN（コマ内=FRAME子/飛び出し=ルート）/ composer …トーン・スピード線・枠の板（SPEC_09）
7. manga-plate →（×4 SEEDS連番）→ OBAN seqパネル（loop）…集中線がバタつく演出
8. 紙ネーム/ラフ写真 → econte（BOARD切り出し→SHEET絵コンテ→TIMELINE）→ **動画コンテ WebM/mp4** …プリプロ（SPEC_10 P0+P1）。**P2で → animator REF（本作画へ）／ → カラースクリプト一覧PNG を接続予定**

## 関連スキル

- `satsuei-fx-kit` — 撮影処理チェーンの移植（正準コア＋レシピ）
- `composer-timeline-kit` / `camera-rig-orbit-capture` / `object-rig-gizmo-capture` / `vj-audio-export-kit`
- `single-html-verify` — 決定論VRT＋パフォーマンスバジェット＋APIモック検証（正準 `VERIFY_HARNESS/`）

## 外部プロジェクトとの接点（LP_motion-graphics）

- **PARALLAX_LAB**（`LP_motion-graphics/PARALLAX_LAB/parallax-lab.html`）— 「寄り」の3流儀（①撮影台マルチプレーン／②3Dドリー／③スケール）比較教材ラボ。
  係数は本リポジトリの実機値と同一（`k(depth)=lerp(0.55,1.25,depth)` / `persp=F/(F+Z−camZ), F=1000` ≒ composerの`PERSP_FOCAL`）。
  **TAKEブリッジ**: OBAN_BUILDERの**TAKE JSON**（SPEC_01 §2 `PROJECT.take`、COPY PROJで取得可）を貼ると、
  同じカメラワークを3流儀で並べて再生できる＝**実プロジェクトの差分検証機**。
  **設計・仕様は `SPEC_12_PARALLAX_TAKE_BRIDGE.md`（P0/P1/P2未着手）**。
  実装はPARALLAX_LAB側の対応で、本リポジトリ側は**TAKE JSON形式（`kf:[{x,y,z,dwell,ease}]`）を変えるときにSPEC_12との互換性を意識**すれば足りる。
- 上記以外、`LP_motion-graphics/CLAUDE.md`のSPEC_06に関する記述（「composer P0/P1・P3は未着手」）は古い
  （本リポジトリでは両方とも実装済み）。あちらのドキュメントなので本リポジトリからは編集しない。
