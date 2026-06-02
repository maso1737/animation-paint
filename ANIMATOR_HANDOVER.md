# ANIMATOR プロジェクト引継ぎ書
_新チャット冒頭にこのファイルを貼り付けてください_

---

## ⚑ 次チャットへの申し送り（最新・優先）

### 作業環境
- 実ファイルは `animator.html`（リポジトリ直下）。`index.html` はランディング、`composer.html` は合成。
- **master ブランチで直接作業・push**（worktree 運用は不要）。
- 反映先 GitHub Pages: https://maso1737.github.io/animation-paint/
- 変更後は必ず構文チェック:
  ```
  node -e "const fs=require('fs');const h=fs.readFileSync('animator.html','utf8');const m=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(x=>x[1]).filter(s=>s.length>200).join('\n;\n');new Function(m);console.log('OK')"
  ```

### 未着手・次チャット候補

**すぐできる（小〜中）**
- **手振れ補正の強化**：`state.smoothing` は4点移動平均で実装済み。pull/lazy-brush 方式への強化が良い。
- **→ANM 代替動線**：別ANIMATORを安全に別ウィンドウで開く（→COMPOSERと同方式）。現状は IMPORT JSON で手動。

**中規模フェーズ**
- **INDEX ページにANIMATOR一覧**：IndexedDB のプロジェクト一覧を index.html に表示し、開く/REF参照に追加できる動線。`+ FROM SAVED` の使いにくさを解消する（→「FROM SAVED について」参照）。
- **ANIMATOR↔COMPOSER 複数窓LIVE管理**：複数ANIMATORを別窓で同時開きのとき、どれがCOMPOSERとLIVE連携するかの帰属管理。

**将来フェーズ（大規模）**
- **線のコピペ＆選択移動/回転/スケール**（Ctrl矩形・Alt投げ縄）：大規模・見送り中。
- **ドックマネージャ（UI配置）**：設計コストが大きい。今はフローティング/リサイズで代替。

---

## プロジェクト概要
ブラウザベース・日本アニメ作画特化アニメーションエディタ。
単一HTMLファイル（`animator.html`）、IndexedDB自動保存、Chrome/iPad Safari対応。
COMPOSERと BroadcastChannel でリアルタイム連携。

---

## ファイル構成
- **`animator.html`** — メインの作画エディタ（約265KB）
- **`composer.html`** — マルチトラック合成（カメラ/KF/書き出し）
- **`index.html`** — ランディングページ
- **`CLAUDE.md`** — Claude Code 向けの開発メモ（短縮版）
- 保存場所: `C:\Users\so173\Documents\Claude\Projects\Animation_Paint\`
- GitHub: https://github.com/maso1737/animation-paint

---

## 技術スタック
- Canvas 2D（内部解像度可変・上限≒4K面積。各コマは `drawData` Uint8ClampedArray W×H×4）
- アンチエイリアスOFF、ブレゼンハム直線、自前ピクセル描画
- Pointer Events API（Apple Pencil / マウス / タッチ 統合）
- IndexedDB自動保存（差分・debounce 800ms）
- BroadcastChannel `tdr_live` によるリアルタイム連携

---

## レイヤー構成（`#canvas-wrap` 内、上から）

| canvas ID | 用途 | 書き出し |
|---|---|---|
| `bg-layer` | 背景（明度スライダー） | ○ |
| `ref-layer` | REF ANIMATOR（コマ同期） | ✕ |
| `refimg-layer` | REF IMAGE（原寸・複数・移動可） | ✕ |
| `frame-layer` | FRAME LAYER（描ける下絵・移動可） | ✕ |
| `onion-prev/next` | オニオンスキン | ✕ |
| `draw-layer` | 描画レイヤー | ○ |
| `guide-layer` | 解像度枠/セーフ（表示のみ） | ✕ |

---

## 実装済み機能（最新）

### キャンバス・描画
- ペン 1px / 2px / 20px、アンチエイリアスOFF
- 筆圧ペン（PEN ダブルクリックで黄バッジ ON/OFF）
- インクカラー 3色（主線黒・下書き水色・指示オレンジ）
- 消しゴム（サイズ独立記憶）
- 塗りつぶし（スキャンラインフラッドフィル。FILL ダブルクリックで透明消しゴムモード）
- ストローク補正（4点移動平均 / `state.smoothing`）
- 背景明度縦スライダー
- ピンチズーム＋パン / FLIP（表示反転） / MIRROR（左右対称・軸ドラッグ・ダブルクリックで垂直⇄水平）
- **CANVAS SIZE 可変**：左上ラベルクリック→設定パネル。上限≒4K面積。既存の絵は中央フィット保持。

### FILL PALETTE（カスタム可）
- 初期48色。スロット選択→スポイト(Alt+クリック)で上書きカスタム。
- ＋/−でスロット追加/削除（最低1色）。⇩/⇧でJSON書き出し/読込。
- localStorage `animator_palette_v1` に保存（ブラウザ全体共通）。

### REF パネル（`topbar REF` ボタン）
- **パネル開閉＝キャンバス上の全参照表示マスター ON/OFF**（閉じると全参照非表示＋EDIT OFF）
- **パネル最上部の共通行**：MOVE ボタン / 中央ボタン / 選択状況表示
  - MOVE モード中：画像はキャンバスでクリック→最前面化＋ドラッグ移動、ホイールで拡縮
  - ANIMATOR/FRAMEはリスト/見出しクリックで選択してからMOVEでドラッグ
  - 中央ボタン：選択中参照を原寸(1:1)中央に戻す

- **FRAME LAYER**（描ける下絵）：
  - EDIT ON で描画対象が frame-layer に切り替わる
  - 見出しクリックで MOVE 選択 → ドラッグ移動・ホイール拡縮（描いた内容ごと移動）
  - CSS transform で移動（描画バッファは保持。描画時は `toFrameLocal()` で逆変換）

- **REF IMAGE**（複数）：
  - LOAD で複数選択読込・原寸中央・カスケード配置。× IMG で選択/最後の1枚を削除。不透明度スライダー。
  - meta に `refImages[]`（src/x/y/scale/opacity）で保存・復元

- **REF ANIMATORS**（JSON連番）：
  - `+ JSON FILE` で追加（複数選択可）
  - `+ FROM SAVED`：同ブラウザ内で → COMPOSER 等経由で送信した `tdr_exchange` DB の一覧から追加（→「FROM SAVED について」参照）
  - リスト項目クリックで選択（重なりでも選べる）→ MOVE でドラッグ移動・ホイール拡縮
  - VIS で表示ON/OFF。× で削除。→ANM ボタンは削除済み（誤タップ防止。別ANIMATORはIMPORT JSONで開く）
  - 手元より長いJSONは再生・スクラブでREF全尺まで追従（`effectiveTotalTicks()`でタイムライン延長）
  - transform（x/y/scale）を refs DB に保存・復元

### 設定パネル（⚙ ボタン / `,` キー / 左上ラベルクリック）
- **CANVAS SIZE**：W×H 入力 + プリセット。変更時は確認ダイアログ→中央フィット。
- **CANVAS GUIDES**：解像度枠 ON/OFF+px 入力 / セーフフレーム ON/OFF+% 入力（guide-layer に描画のみ）
- **KEYBOARD SHORTCUTS**：全アクションをキー再割当（localStorage `animator_keymap_v1`）。RESET DEFAULTS。
- 下端ドラッグ＋段スナップで高さ調整（ビューポート超過しない）
- REFパネル・パレット（フローティング時）も同様のリサイズグリップ

### タイムライン
- `kind: 'draw' | 'empty'` の2種セル。duration 1〜120。末尾タップで追加。
- セル並び替えドラッグ / 複数選択(Shift) / コピペ / 移動 / 削除
- Alt+ホイール / iPad ペンスクラブでセル基準幅伸縮

### ワークエリア（時間ルーラー上の黄色帯）
- 左右ハンドルドラッグで範囲指定（実タイムライン内に収まる）
- **Wキー**：選択フレームがあればその範囲にFIT、なければ全範囲
- **帯スライド（移動）**：中ボタンドラッグ（マウス）またはタッチ。左クリック/ペンはタイムライン操作に委ねる
- REFが手元より長いとき全範囲再生でREF末尾まで延長（ただし帯/ハンドルは実タイムライン内）
- time-track の空き領域ダブルクリック → 全範囲

### LIVE 連携（BroadcastChannel `tdr_live`）
- ANIMATORの autosave 完了 → COMPOSERへ `project-update` をブロードキャスト
- COMPOSERが受信 → `projectId` 一致トラックの絵だけ差し替え（KF/transform/マーカー保持）
- `→ COMPOSER` ボタン：別ウィンドウで COMPOSER を起動し LIVE も有効化。再クリックで前面化。
- topbar の `LIVE ○/●` で接続状態を表示。手動クリックで強制再送可。
- 同一ブラウザ・同一オリジンのタブ間のみ。

### 書き出し
- **EXPORT**：作業解像度そのまま PNG（BG白固定 or Shift で透過）
- **SEQ PNG**：全コマを作業解像度 BG透過の連番 PNG で ZIP 一括書き出し（JSZip CDN）
- **EXPORT JSON**：COMPOSER連携用プロジェクトJSON（cells に PNG data URL を埋め込み）
- **IMPORT JSON**：別プロジェクトを読み込んで現在のプロジェクトを置換

### Undo / Redo
- グローバルログ `gUndo[] / gRedo[]`（canvas / timeline / framelayer の3系統を時系列統合）
- キャンバスUndoは各コマ独立（50ステップ）。コマをまたぐ統合Undoはしない（仕様）
- タイムライン履歴 `tlHistory[]`（20ステップ・idベースマージで絵を温存）

### ショートカット（デフォルト・⚙で変更可）
| キー | アクション |
|---|---|
| B / E / G | ペン / 消し / 塗り |
| 1 / 2 / 3 | サイズ 1px / 2px / 20px |
| O | オニオンスキン |
| N / D | 新規 / 複製フレーム |
| S | SPLIT（スクラブ位置で分割） |
| W | ワークエリア=選択FRAME範囲 / 全範囲 |
| Space | 再生 / 停止 |
| ← / → | コマ移動 |
| Del / BS | フレーム削除 |
| F | FIT |
| M | MIRROR |
| X | FLIP |
| R | REFパネル開閉 |
| , | 設定パネル開閉 |
| Tab | ツールバー左右入替 |
| Alt+クリック | スポイト（選択スロット上書き） |
| Ctrl+Z / Y | Undo / Redo |
| 指2本タップ / 指3本タップ | Undo / Redo |

---

## 主要な state フィールド

```js
const state = {
  tool, penSize, eraseSize, inkColor, fillColor,
  pressurePen, fillErase,
  bgBright, zoom, fitMode, panX, panY,
  frames,              // [{id, kind, duration, hidden, drawData, thumbCanvas, history, histIdx, ...}]
  currentFrame, fps, playing, loop, onionOn, smoothing,
  cellBaseW, cellStretchPerTick,
  workStart, workEnd,  // ワークエリア (tick単位, exclusive)
  flipped, mirrorOn, mirrorAxis, mirrorAxisX, mirrorAxisY,
  timeUnit,            // 'frames' | 'seconds'
  toolbarSwapped, paletteFloat, paletteFloatPos,
  selectedFrameIds,    // Set<frameId>
  frameLayer: { visible, opacity, editMode, x, y, scale },
  refAnimators: [{ id, name, projectId, visible, opacity, color, ticks, totalTicks, fps, source, x, y, scale }],
  refImages:    [{ id, img, src, x, y, scale, opacity, visible }],
  refMoveMode, refSel, // refSel: {type:'img'|'anim'|'frame', id?}
  refMasterShow,       // REFパネル連動マスター表示
  guides: { resOn, resW, resH, safeOn, safeA, safeB },
  projectId,           // BroadcastChannel連携用ID
  composerData,        // COMPOSERのKF等を保持してEXPORT時に乗せる
};
// グローバルUndo
const gUndo = [], gRedo = [];
// タイムライン履歴
const tlHistory = []; let tlHistIdx = -1;
// パレット（localStorage）
let gPalette = [...]; let gPalSlot = N;
// キーマップ（localStorage）
let gKeymap = {};
```

---

## アーキテクチャ上の注意点

- **タッチ**：全タッチは `#stage` の pointerdown/move/up で一元管理（`touchPtrs: Map`）。drawCanvas は pen/mouse のみ。
- **FRAMEレイヤー移動**：`applyFrameLayerTransform()` で CSS transform。描画時は `toFrameLocal(x,y)` で逆変換してバッファ座標へ。
- **REF IMAGE 最前面化**：`bringImageFront(id)` が `state.refImages` 末尾へ splice → `renderRefImg` が後に描画。
- **CANVAS SIZE 変更**：`setCanvasSize()` → `setupCanvas()` → 全フレームの `drawData` を `resampleDrawData()` で中央フィット（nearest）再サンプル。Undo履歴はリセット。
- **ショートカット**：`SHORTCUT_ACTIONS` 配列にアクション登録 → `gKeymap`(localStorage) で照合。`handleShortcutCapture()` がキャプチャモード中は全 keydown を横取り。
- **LIVE 連携**：`flushSave` 完了後 `broadcastProjectDebounced()` が `gLiveActive` なら送信。`gComposerWin` で別窓への参照を保持（再クリックで前面化）。
- **削除＝空ブロック変換**：drawフレーム→empty（タイムライン尺維持）。emptyの削除は完全splice。
- **effectiveTotalTicks()**：手元 totalTicks と表示中REFの最長の大きい方。再生・スクラブ・ルーラーがREF尺まで延長する根拠。

---

## FROM SAVED について
`+ FROM SAVED` は、**同じブラウザ内で → COMPOSER 等を経由して送信した共有DB（`tdr_exchange`）に残っているプロジェクト一覧**から参照を追加する機能。
- 最初は空で分かりにくい（同ブラウザ内でプロジェクトを → COMPOSER 送信して初めて候補に出る）
- 将来的には **index ページにANIMATOR一覧** を出して、そこから開く/REF参照に追加できる動線が理想（別フェーズ）

---

## 既知の制限・注意点
- ライブ連携は同一ブラウザ・同一オリジンのタブ間のみ（異なるPCは対象外）
- キャンバス拡大はメモリ×コマ数で増加（8Kは多コマ不可。上限は4K面積）
- FRAMEレイヤーを移動した状態でのフラッドフィルは `toFrameLocal()` 経由（移動+スケールが大きいと描画位置が微妙にズレる可能性あり）
- タイムラインUNDO後 IndexedDB に即時反映されるが、大量undo時はDBと一時的に乖離する可能性あり
- Apple Pencilの描き始めに若干の反応遅延あり（許容範囲とのこと・将来改善候補）

---

## 開発スタイル
- 単一HTMLで完結。外部依存は CDN（JSZip 等）のみ。
- 変更は Edit（差分）で最小限に。全書き換えは避ける。
- 構文チェック（`new Function`）→ Pages か `file://` で実機確認。
- push は明示依頼時のみ。大きい・不可逆な変更は先に方針を確認。
