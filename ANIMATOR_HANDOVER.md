# ANIMATOR プロジェクト引継ぎ書
_新チャット冒頭にこのファイルを貼り付けてください_

---

## プロジェクト概要
ブラウザベース・日本アニメ作画特化アニメーションエディタ。
単一HTMLファイル（`index.html`）、IndexedDB自動保存、Chrome/iPad Safari対応。

---

## 現在のファイル
- **最新版**: `index.html`（約130KB）
- 保存場所: `C:\Users\so173\Documents\Claude\Projects\Animation_Paint\`
- GitHub: https://github.com/maso1737/animation-paint
- Pages: https://maso1737.github.io/animation-paint/

---

## 技術スタック
- Canvas 2D（内部解像度 2048×1152、書き出し4K）
- アンチエイリアスOFF、ブレゼンハム直線、自前ピクセル描画
- Pointer Events API（Apple Pencil / マウス / タッチ 統合）
- IndexedDB自動保存（差分保存）
- PWA化は未実施（GitHub Pages → Safari「ホーム画面に追加」で確認中）

---

## 実装済み機能（最新版）

### レイヤー機能（各コマ側にはレイヤーは持たない（軽量化））
- FRAMEレイヤー: 下書き/参考用レイヤー（白/黒ベタ平面を透明度調整してトレース。書き出しに含まれない）
- 参照画像読み込み → トレースレイヤーに組込み

### 参照ANIMATOR（複数ANIMATOR参照 / REFパネル）
- topbar `REF` ボタンでフローティングパネル開閉（`#ref-panel`、ヘッダドラッグ移動）
- 他のANIMATORプロジェクト（PROJECT_v1/ANIMATOR_v1 JSON）を **コマ同期で重ね表示**
  - 追加元: `+ JSON FILE`（複数選択可） / `+ FROM SAVED`（共有DB tdr_exchange の保存済み）
  - 各参照: 表示ON/OFF、不透明度スライダー、`→ ANM`（その素材のANIMATORを開く=連携更新）、削除
- 専用 `#ref-layer` キャンバス（bg-layerの直後、frame-layerの背面）へ毎フレーム合成
  - `source.cells` を1ティック=1画像へ展開（`expandRefTicks`）。参照側が尽きたら非表示
  - 現在ティック同期: 非再生時=`frameStartTick(currentFrame)` / 再生時=再生ティック
  - 書き出し / COMPOSER / オニオン には**含まれない**（純粋に作画参照用）
- 永続化: IndexedDB `STORE_REFS`（DB_VERSION 3）。変更時のみdebounce保存（描画autosaveとは別系統）
- 5レイヤー分けの運用: ANIMATORを5本作り、各々で他4本をREF参照。COMPOSERでまとめて同期

### キャンバス・描画
- ペン 1px / 2px / 20px、アンチエイリアスOFF
- インクカラー 3色（主線黒・下書き水色・指示オレンジ）
- 消しゴム（ペンと独立してサイズ記憶）
- 塗りつぶし（スキャンラインフラッドフィル）
- 48色パレット（右サイドパネル）
- ストローク補正（4点移動平均）
- 背景明度縦スライダー（左ツールバー）
- ピンチズーム＋パン（キャンバス）
- Undo/Redo（履歴50ステップ、フレーム単位独立）
- **FLIP**（表示左右反転のみ）/ **MIRROR**（左右対称ペイント、中央軸）
- ミラーガイドライン（`#mirror-guide`、stageの子要素としてJS座標で配置）

### ツールバー構成
- **左**: ペン・消し・塗り / サイズ1px・2px・20px / 主線・下書・指示 / BG縦スライダー / UNDO↩ REDO↪ ボタン
- **右**: 48色FILL PALETTEのみ
- **トップバー右**: FLIP / MIRROR / EXPORT 4K

### タイムライン
- `kind: 'draw' | 'empty'` の2種セルで構成
- duration（コマ打ち数）:
  - 1〜3コマ: 固定幅 + 内側仕切り線
  - 4コマ以上: セル横伸縮 + 下部目盛り
  - セル右端スクラブで 1〜120コマまで自由変更（最大15秒制限）
- 末尾エリアタッチ: 「前フレームを伸ばす + 1コマの新フレーム追加」
- セル並び替え: ペン/マウスでドラッグ
- **指でタイムラインを横スワイプ = ネイティブスクロール**
- Alt+ホイール / ipadはtime-track上ペンスクラブでセル基準幅伸縮
- **フレーム非表示トグル** (`f.hidden`): セルホバーで●/✕ボタン表示。非表示セルは暗いオーバーレイ。再生・オニオンスキンでスキップ
- **SPLIT**: 現フレームをコマ数半分に分割。同じ絵を両方にコピー
- 複数選択
- フレームセルをShift+クリック or タップで追加選択
- 選択中セルはハイライト表示
- コピー/ペースト
- コピー: 選択フレームをクリップボード
- ペースト: currentFrameの直後に挿入
- 複数選択での丸ごと移動
- 元位置には empty ブロックが残る（長さ維持）
- 選択中のフレームをまとめて削除

### ストリップツールバー（再生エリア）
- PLAY / STOP / FPS入力
- **IN**: 現フレームの開始コマ（スクラブ → 前フレームのdurationが変わる）
- **OUT**: 現フレームの終了コマ（スクラブ → 現フレームのdurationが変わる）
- **TIME**: 全コマ数（タップでコマ↔秒トグル）
- ONION / SPLIT / + FRAME / DUPLICATE / DELETE

### ワークエリア（時間メモリ上部）
- ネオンイエローの帯で再生・書き出し範囲を表示
- 左右ハンドルドラッグで範囲指定
- 中央ボディドラッグで範囲移動
- work-area-body ダブルタップで「全範囲 → 現在フレームのみ → 元の状態」サイクル
- **time-track空きエリアのダブルクリック/タップ → 全範囲**
- 再生はワークエリア内のみをループ

### 時間ルーラー（time-track）
- **12コマ = 1k** 単位でkラベル（12f=1k, 24f=2k …）
- `cellStretchPerTick ≥ 12px` → 1コマ単位の小目盛り、`≥ 5px` → 3コマ、それ以外 → 6コマ
- 秒モード時: 小数1桁の秒ラベル
- 再生中カーソルは毎RAF（1コマ単位）でリアルタイム移動

### UNDO / REDO（2系統）
- **キャンバス系統** (`state.lastEditWas === 'canvas'`): 現フレームの描画履歴 (50ステップ)
- **タイムライン系統** (`state.lastEditWas === 'timeline'`): フレーム構成スナップショット (20ステップ)
  - `tlHistory[]` + `tlHistIdx` でスナップ管理
  - `pushTimelineHistory()` は addFrame / deleteCurrentFrame / splitCurrentFrame / スクラブ終了（変更時のみ）で自動呼出
  - スナップは drawData 参照 + history の shallowコピー で保存（メモリ効率と正確性を両立）
- **切替ロジック**: Ctrl+Z / 指2本タップ → `lastEditWas` に基づいて対象を自動判定。キャンバス履歴がない場合はタイムライン系にフォールバック

### 操作系
- **Apple Pencil**: 常に描画
- **指1本**: ペンツール時=パン、消しゴム/塗りつぶし時=描画
- **指2本**: ピンチズーム + パン
- **指2本タップ**: Undo
- **指3本タップ**: Redo
- **キーボード**:
  - B=ペン、E=消し、G=塗り、1/2/3=サイズ、O=オニオン
  - N=新規、D=複製、**S=SPLIT**、Del/Backspace=削除
  - Space=再生、←→=コマ移動
  - **W=ワークエリア全範囲リセット**
  - Ctrl+Z/Y=Undo/Redo

### 保存・書き出し
- IndexedDB自動保存（800ms遅延、差分保存）
- 4K PNG書き出し（現在のフレームのみ、BG白固定）

---

## 主要な state フィールド

```js
const state = {
  tool, penSize, eraseSize, inkColor, fillColor, bgBright,
  zoom, fitMode, panX, panY,
  frames,           // [{id, kind, duration, hidden, drawData, thumbCanvas, history, histIdx, ...}]
  currentFrame, fps,
  playing, loop,
  onionOn,
  smoothing,
  cellBaseW,              // セル基準幅 (px)
  cellStretchPerTick,     // duration 4以上の1コマあたり幅 (px)
  workStart, workEnd,     // ワークエリア (コマ単位、exclusive)
  flipped, mirrorOn, mirrorAxisX,
  timeUnit,               // 'frames' | 'seconds'
  lastEditWas,            // 'canvas' | 'timeline' ← UNDO系統切替に使用
};
// タイムライン履歴（stateの外）
const tlHistory = [];
let tlHistIdx = -1;
```

### フレームオブジェクト
```js
{
  id,           // ユニークID
  kind,         // 'draw' | 'empty'
  duration,     // コマ打ち数 (1〜120)
  hidden,       // 非表示フラグ (draw のみ)
  drawData,     // Uint8ClampedArray (draw のみ)
  thumbCanvas,  // サムネ canvas要素 (draw のみ)
  history,      // スナップ配列 (draw のみ)
  histIdx,      // 現在位置 (draw のみ)
  strokeCount,
  dirty,
}
```

---

## アーキテクチャ上の注意点

- **タッチ**: 全タッチ処理は `#stage` の `pointerdown/move/up` で一元管理（`touchPtrs: Map`）。drawCanvas は pen/mouse のみ。
- **ミラーガイド**: `#mirror-guide` は canvas-wrap の兄弟要素（transform の影響を受けない）。位置は `updateMirrorGuide()` がJS座標で計算。
- **FLIP + 座標**: `toCanvasCoord()` 内で flipped 時に x を反転。ミラーも考慮済み。
- **削除=空ブロック変換**: deleteCurrentFrame は drawフレーム→empty 変換（タイムライン尺維持）。emptyの削除は完全splice。
- **timeline-cursor**: `#time-track-inner` 内の絶対配置 div。再生中は `tickToPx(tick)` で毎RAFで left を直接更新（updateTimeTrack呼び出しなし）。
- **IndexedDB**: v1→v2 移行コードあり（frameOrderの形式変換）。

---

## 既知の問題・注意点

- Apple Pencilの描き始めに若干の反応遅延あり（許容範囲とのこと、将来改善候補）
- `tlHistory` のスナップは drawData を参照渡しで保持するため、restoreTL 後は古い参照が frame.drawData に入る場合がある（syncFrameFromCanvas で随時更新されるため実用上は問題なし）
- タイムラインUNDO後 IndexedDB に即時反映（scheduleSave）されるが、大量undo時はDBと一時的に乖離する可能性あり

---

## 開発スタイル
- 短いプロンプト＋即実行スタイル（「go!」「見せてー」）
- 単一HTMLファイルで完結させる原則
- テストは `node -e "new Function(scriptContent)"` で構文確認後、実機で最終確認
- 変更は Edit（str_replace）で差分のみ修正（全書き換え最小限）
- GitHub push は明示的に依頼された時のみ

---

## UI調整（実装済みメモ）
- ツールバー左右入替は grid-template-areas で実装（#app.swapped で areas/列幅を反転、
  stageは常に中央。崩れ防止）。⇄ボタンはツール列最上部（.tb-swap）。ショートカット Tab
- topbar の FRAME群（編集/表示/不透明度/CLR/IMG）は REFパネルへ集約。
  REFパネル構成: ①FRAME LAYER（EDIT/✕/CLR/IMG＋OPACITY）②REF ANIMATORS（参照リスト）
  topbarは REF ボタン1個に集約。REFパネル開閉ショートカット R
- ショートカット変更: F=FIT / M=MIRROR / X=FLIP / R=REFパネル / Tab=ツールバー左右入替
  （旧 F=FRAME編集 は廃止。FRAME編集はREFパネルのEDITボタン）
- SPLITボタンは廃止。Sキーのみ（カーソル下=state.scrubTickで分割）
- timeline-cursor に透明ヒットエリア（::after 幅18px）追加でつかみやすく
- ホイール拡縮は zoomAt() をステージ中心基準の逆写像に修正（カーソル下が動かない）

## 次回実装候補（優先度おまかせ）

### 機能要望メモ

**ANIMATOR↔COMPOSER直結ボタン** （将来的）
- localStorageかIndexedDBで同一ブラウザタブ間の完全リアルタイム連携
、ポスト通信（BroadcastChannel API）

**キャンバス**
- ~~主線/下書き/指示ボタンはERASE時は主線ONのみON~~ → 実装済み（applyInkUIForTool）
- ~~筆圧ペン（ペンボタンWクリックで変更）~~ → 実装済み
  （state.pressurePen / pressureRadius / drawDotRadius・drawLineRadius。PEN Wクリックで黄バッジ）
- ~~透明で塗りつぶし消し（塗りつぶしボタンWクリックで変更）~~ → 実装済み
  （state.fillErase。floodFillが(0,0,0,0)で領域クリア。FILL Wクリックで桃バッジ）
- 無限キャンバス（書き出しは4Kのみ）。左上の2048×1152をタップで解像度枠の表示/変更。セーフフレーム80%/90%＋のりしろ1.2倍表示
- 線のコピペ＆選択/移動/回転/スケール（ペイント中にCtrl押すと矩形選択、Alt押すと投げ縄にかわる）
- FILL PALETTEのフローティング化（右パネルから切り離してドラッグ移動できるウィンドウに）
- 参照画像パネル フローティングUI（画像読み込み（File API）・ホイール拡縮 + パン・スポイトでFILL PALETTEに色取り込み）
- ~~ツールバーの左右入れ替え~~ → 実装済み（topbar `⇄`。#app.swapped でgrid列入替、meta.toolbarSwappedで永続化）

**タイムライン**
- ~~SPLITツール: クリックした位置で分割~~ → 実装済み（state.scrubTick=スクラブ/クリック位置、
  splitAtPlayhead()。SPLITボタン/Sキー共にクリック位置で分割、範囲外は中央フォールバック）
- ~~トラック レイヤー機能~~ → **REF参照ANIMATORで解決済み**（ANIMATOR側はトラックを持たず、
  複数ANIMATORをREFパネルでコマ同期参照。レイヤー分けは別ANIMATORを作りREFで重ねる運用）

**書き出し**
- ~~連番PNG BG透過 ZIP一括書き出し（COMPOSER/AE合成前提）~~ → 実装済み
  （topbar `SEQ PNG`。exportSequencePNG()。1ティック=1枚で4K透過、保持コマは複製、JSZip CDN追加）

**保存**
- ローカルファイル保存/読み込み（IndexedDB補完用） ※EXPORT/IMPORT JSONで概ね充足

**Settings UI**
- フローティングSettingsウィンドウ
- ショートカット変更
- ツールバー左右取り外しレイアウト変更

### 開発バックログ（工数大きめ）
1. **複数選択** (Shift+クリック) + コピー/ペースト + 複数移動/削除
2. **4K連番PNG書き出し（zip）** — JSZipでzip圧縮
3. **FILL PALETTEのフローティング化**
4. **参照画像パネル**（フローティング、スポイト付き）
5. **タイムライン共通レイヤー**（全コマ共通の背景トレース用）
6. **PWA化**（manifest.json + service worker）
7. **GitHubエクスポート/インポート**（zip形式）
