# PIPELINE.md — ツール間の入口/出口マップ

「え、そういう切り口からできたの！？」を偶然でなく設計で見つけるための1枚。
**新しいルートは「出口Aの形式と入口Bの形式が合うか」をこの表で探す。**
ツールの入出力を変えたら必ずこの表を更新すること（各プロジェクトのCLAUDE.mdより先にここ）。

## 入口/出口 一覧

| ツール | 入口（読めるもの） | 出口（書き出すもの） | 撮影処理 |
|---|---|---|---|
| **animator.html** | 画像(REF/下絵) / プロジェクトJSON(IndexedDB) | ANIMATOR_v1 JSON / ライブ連携(BroadcastChannel `tdr_live`) | なし（作画に専念） |
| **composer.html** | PROJECT_v2 / PROJECT_v1 / ANIMATOR_v1 / IMAGE_v1(PNG·JPEG·WebP) / audio / ライブ連携 | 4K連番PNG(zip) / 動画(MP4·WebM) / EXPORT WEB(スクロールビューアHTML) / PROJECT_v2 JSON | **P0〜: fxチェーン**（VIDEO=rt / PNG=final / EXPORT WEB=rt・P2b〜） |
| **OBAN_BUILDER** | 画像D&D(単品/連番seq・**MANGA PLATEのPNG含む**) / プロジェクトJSON(クリップボード) / **SPEC_07: + FROM ANIMATOR(EX_DB)＋ライブ連携(`tdr_live`受信・PNG書き出し不要)** | oban-viewer.html(単一HTML・画像は同フォルダ参照・**ap-seqはdataURLベイク同梱**・**SPEC_09 P4: FRAME枠線含む**) / プロジェクトJSON / **P3: COPY FOR COMPOSER(PROJECT_v2=CAMERAトラック+fx同梱・クリップボード)** / **SPEC_07 B3: EDIT IN ANIMATOR(`?open=`ディープリンク)** | **P2〜: take.fx→ビューアrt** |
| **manga-plate.html** | REF画像(下敷き・表示のみ) / PLATE_v1 JSON(クリップボード) | 透過PNG(elem別/全体/×4 SEEDS連番) / PLATE_v1 JSON | なし（素材生成に専念） |
| **oban-viewer.html** | 同フォルダの画像ファイル | （最終出力・スクロールLP） | rt実行時（`?fx=0`でOFF） |
| **EXPORT WEBビューア** | （画像は埋め込み済み） | （最終出力・スクロールLP） | P2b(任意)でrt |
| **AE** | 4K連番PNG | 完成動画 | AE側（fx OFFで持ち込む） |
| **VERIFY_HARNESS** | 検証対象HTML（`window.__HARNESS__`契約実装） / baseline PNG / API録画fixtures | 統合結果JSON(summary.json) / diff PNG / baseline PNG | 検証専用（fx対象外） |

## 共通フォーマット

- **fxスキーマ v1**（SPEC_06 §2）— 全ツール共通・無変換で持ち回り。未知エフェクトはスキップ（前方互換）
- **TAKE**（SPEC_01）— `{kf:[{x,y,z,dwell,ease}]}`。OBAN / LP_Model_CR 系で共通。P3でcomposer CAMERAトラックへ変換可
- **PROJECT_v2** — composerの保存形式。IMPORT JSONは複数回で追加合成できる。**P0〜: トップレベル `fx:`（fxスキーマv1）を同梱**（欠損時は composer 側で `makeDefaultFx()` 補完。PROJECT_v1 単一トラック保存には乗らない）

## 成立している制作ルート

1. animator → composer →（fx OFF・連番）→ AE → 動画 …従来
2. animator → composer →（fx final）→ 動画/連番 …AE不要ルート(P0〜)
3. animator → OBAN_BUILDER → oban-viewer(LP) …スクロールコンテンツ(fxはP2〜)。**SPEC_07〜: PNG書き出し不要（+ FROM ANIMATOR＋ライブ同期→EXPORTでdataURLベイク）**
4. animator → OBAN_BUILDER →（P3変換）→ composer →（fx final）→ 動画
5. animator → composer → EXPORT WEB(LP) …fx rt実行(P2b〜)。fx有効時のみコア同梱・`?fx=0`でOFF
6. manga-plate →（透過PNG）→ OBAN（コマ内=FRAME子/飛び出し=ルート）/ composer …トーン・スピード線・枠の板（SPEC_09）
7. manga-plate →（×4 SEEDS連番）→ OBAN seqパネル（loop）…集中線がバタつく演出

## 関連スキル

- `satsuei-fx-kit` — 撮影処理チェーンの移植（正準コア＋レシピ）
- `composer-timeline-kit` / `camera-rig-orbit-capture` / `object-rig-gizmo-capture` / `vj-audio-export-kit`
- `single-html-verify` — 決定論VRT＋パフォーマンスバジェット＋APIモック検証（正準 `VERIFY_HARNESS/`）
