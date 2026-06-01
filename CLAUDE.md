# CLAUDE.md — Animation Paint

ブラウザベースの日本アニメ作画特化エディタ。単一HTMLで完結、IndexedDB自動保存、Chrome / iPad Safari 対応。

## ファイル構成（リポジトリ直下）
- `animator.html` — メインの作画エディタ（実体。最重要）
- `composer.html` — マルチトラック合成（カメラ/キーフレーム/書き出し）
- `index.html` — ランディングページ
- `ANIMATOR_HANDOVER.md` — 詳細な実装メモ／設計履歴（深掘りはこちら）

GitHub: https://github.com/maso1737/animation-paint
Pages: https://maso1737.github.io/animation-paint/

## 変更後に必ず行う構文チェック
```
node -e "const fs=require('fs');const h=fs.readFileSync('animator.html','utf8');const m=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(x=>x[1]).filter(s=>s.length>200).join('\n;\n');new Function(m);console.log('OK')"
```
composer.html も同様（ファイル名を差し替え）。実機確認は Pages か `file://` で。

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
- **ライブ連携**: `BroadcastChannel('tdr_live')`。ANIMATOR保存→COMPOSERへ project-update。COMPOSERは projectId一致トラックの絵だけ差し替え（KF/transform保持）。`→ COMPOSER` は別ウィンドウで開く。
- **ガイド**: 解像度枠/セーフフレームは `guide-layer` に表示のみ（書き出し非合成）。
- **座標/入力**: 全ポインタ処理は `#stage` で一元化。`toCanvasCoord()` が flip/mirror 換算。

## 既知の制限
- ライブ連携は同一ブラウザ・同一オリジンのタブ間のみ。
- キャンバス拡大はメモリ×コマ数で増える（8Kは多コマ不可。上限4K面積）。
