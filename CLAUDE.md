# CLAUDE.md — Animation Paint（ANI × OBN × COM）

ブラウザベースの日本アニメ作画特化エディタ群。**ANI**mator（作画）× **OB**a**N**（モーションコミック）× **COM**poser（合成/書き出し）の3ツール。単一HTMLで完結、IndexedDB自動保存、Chrome / iPad Safari 対応。

## ファイル構成（リポジトリ直下）

アプリ本体（単一HTML）:
- `animator.html` — メインの作画エディタ（実体。最重要）
- `oban-builder.html` — OBAN BUILDER。モーションコミック製造機（画像→PLACE→TAKE→単一HTMLビューア書き出し）。パイプライン上は animator と composer の間
- `composer.html` — マルチトラック合成（カメラ/キーフレーム/書き出し）
- `manga-plate.html` — パラメトリック漫画素材（コマ枠/トーン/集中線/流線→透過PNG。SPEC_09）
- `econte.html` — ECONTE。プリプロ（紙ネーム写真→BOARD切り出し→SHEET絵コンテ→加筆。SPEC_10。パイプライン上は animator の前）
- `index.html` — ランディングページ（**OBAN 追加予定**）
- `inbetween_lab.html` / `inbetween_warp_lab.html` — 中割り実験ラボ
- `tools/check.js` — 依存ゼロのスモークチェック（構文/配線/ID重複/デッドコード）

ドキュメント（ハンドオーバー／仕様）:
- `ANIMATOR_HANDOVER.md` — animator 実装メモ／設計履歴（深掘りはこちら）
- `COMPOSER_HANDOVER.md` — composer 実装メモ
- `OBAN_BUILDER_HANDOVER.md` — OBAN BUILDER 実装メモ（旧 OBAN_BUILDER/CLAUDE.md）
- `ECONTE_HANDOVER.md` — econte 実装メモ（cuts[]共有ペイント・TIMELINE書き出しの注意点）
- `MOTION_COMIC_SPEC.md` / `EXPORT_WEB_SPEC.md` — モーションコミック／WEB書き出し仕様
- `PIPELINE.md` — 3ツールの入口/出口フォーマット表。**入出力を変えたら必ず更新**。新ルート探しはまずここ
- `SPEC_01_OBAN_TAKE_RIG.md` — 大判カメラTAKE化の元仕様（SPEC_01 P2 = OBAN BUILDER）
- `SPEC_05_OBAN_BUILDER_V2.md` — OBAN BUILDER V2 拡張仕様
- `SPEC_06_SATSUEI_KIT.md` — 撮影処理キット（fx共通スキーマ・composer/OBAN統合）
- `SPEC_07_ANIMATOR_OBAN_BRIDGE.md` — animator⇄OBAN 連番往復ブリッジ仕様
- `SPEC_09_MANGA_PLATE.md` — パラメトリック漫画素材ツール仕様（P4=OBANコマ枠線含む）
- `SPEC_10_ECONTE.md` — プリプロツール仕様（cuts[]単一データ・BOARD/SHEET/TIMELINE設計。P0+P1実装済み、P2=animator連携/カラースクリプト）

GitHub: https://github.com/maso1737/animation-paint
Pages: https://maso1737.github.io/animation-paint/

## 残タスク・予定タスク一覧（Claude Artifact）

https://claude.ai/code/artifact/a57e3c0b-064f-4e1b-97d2-a158602ab07b

各SPEC/HANDOVERを横断した「まだ終わっていないもの」の棚卸しページ。
**新チャットでこのページを更新するときは、必ず上のURLを Artifact ツールの `url` に渡すこと**
（渡さずに新規publishすると別URLが発行され、ここのリンクが古くなる）。

## 変更後に必ず行うチェック
```
node tools/check.js
```
6ファイル（animator / oban-builder / composer / index / manga-plate / econte）すべての 構文 / JS→HTML の id 配線 / id 重複 / 未参照関数 を一括検査（問題があれば exit 1）。※type属性の無い `<script>` のみJS扱い（`type="application/json"` 等のデータブロックは除外）。実機確認は Pages か `file://` で。

## デプロイ
- `animator.html` / `composer.html` を直接編集 → 構文チェック → **明示依頼があったときのみ** master に commit & push。
- コミットメッセージ末尾に:
  `Co-Authored-By: Claude <noreply@anthropic.com>`

## 開発スタイル
- 単一HTMLで完結。外部依存は CDN（JSZip 等）のみ。
- 変更は Edit（差分）で最小限に。全書き換えは避ける。
- 短いプロンプト＋即実行。push は依頼時のみ。
- 大きい/不可逆な変更（キャンバスのピクセルパイプライン等）や性能トレードオフがある場合は、先に方針を確認。

## アーキテクチャ要点
- **解像度**: `CFG.WORK_W/WORK_H` は**可変**（左上ラベル/設定パネルで変更。上限≒4K面積 `CFG.MAX_AREA`）。各コマは `drawData`(Uint8ClampedArray, W×H×4)。書き出しは作業解像度そのまま。
- **レイヤー**(canvas, すべて WORK サイズ): bg / ref / frame / onion×2 / draw / guide。`setupCanvas()` で一括リサイズ、`applyZoom()` で表示スケール＋`renderGuides()`。
- **state**: ツール/ズーム/frames/再生/guides など一元管理。タイムライン履歴は `tlHistory`、Undoは `gUndo/gRedo` の一元ログ。
- **保存**: IndexedDB（差分・debounce）。meta に workW/workH・guides・ワークエリア等。
- **ショートカット**: `SHORTCUT_ACTIONS` 登録制＋`gKeymap`(localStorage)。設定パネル⚙で再割当。
- **FILL PALETTE**: `gPalette`(localStorage `animator_palette_v1`)。スロット選択中にスポイトで上書き、＋/−/JSON入出力。
- **ライブ連携**: `BroadcastChannel('tdr_live')`。ANIMATOR保存→COMPOSERへ project-update。COMPOSERは projectId一致トラックの絵だけ差し替え（KF/transform保持）。`→ COMPOSER` は別ウィンドウで開く。**OBAN も同じチャンネル・同じ語彙に参加**（animator から見ればもう1つの composer。詳細は SPEC_07）。
- **ガイド**: 解像度枠/セーフフレームは `guide-layer` に表示のみ（書き出し非合成）。
- **座標/入力**: 全ポインタ処理は `#stage` で一元化。`toCanvasCoord()` が flip/mirror 換算。

## 既知の制限
- ライブ連携は同一ブラウザ・同一オリジンのタブ間のみ。
- キャンバス拡大はメモリ×コマ数で増える（8Kは多コマ不可。上限4K面積）。

## 整理メモ（2026-07 統合時）
- 旧 `LP_motion-graphics/` から本フォルダへ統合。`oban-builder.html` は元 `OBAN_BUILDER/` から**ルート直下へフラット化**したため、SPEC_06/07 中の `OBAN_BUILDER/oban-builder.html` という記述は本フォルダでは `oban-builder.html` に読み替える。
- LP 個別プロジェクト（SCROLL_*_LP 等）や AP 非関連の SPEC_02/03/04 は `_済/` に退避済み。
