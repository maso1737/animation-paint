# SPEC 05 — OBAN BUILDER V2「極める」（コマ・シーケンス・撮影処理）

> **この仕様書単体で実装可能なように書いてある（会話コンテキスト不要）。**
> 前提: `OBAN_BUILDER/oban-builder.html`（V1）と `OBAN_BUILDER/CLAUDE.md` を先に読むこと。
> V1の思想を維持する: **触ってすぐわかる／操作の基本は2手／設定はカードとチップに隠す**。
> 参考視覚: `SCROLL_TENSION_LP/scroll-tension-lp.html` の CH05→06（全面カバー→明暗反転、スライドIN）。

## 0. 実装フェーズ（各フェーズ単体でリリース可能・順序厳守）

| フェーズ | 内容 | 難度 |
|---|---|---|
| **V2-A 品質の土台** | z-order分離 / undo-redoスタック / UIホバー時ホイール修正 / ガイド表示 / KFフォーカス+矢印移動 | 低 |
| **V2-B シーケンスとコマ** | 連番シーケンスパネル / 四角マスク / FRAME（1段ネスト+内部パララックス） | 中 |
| **V2-C 演出と撮影** | スライドIN・寄り引きドリフト / カバー反転・ホワイトアウトワイプ / 撮影FX（グレイン・ディフュージョン・パラ） | 中 |
| **V2-D テキストと反応** | 縦書きテキスト / EN字幕トラック(ON/OFF) / ビューアのクリックFX | 中 |

**各フェーズの完了条件（共通）**: ①EXPORTビューアが同じ見た目を再生できる（ビルダーとビューアの描画関数はコード複製で同期。テンプレート変更時は両方のスモークを回す） ②既存スモーク（builder本体+生成ビューアの構文/往復/ATTRACT）通過 ③V1プロジェクトJSONが読める（マイグレーション: 欠けたフィールドは既定値補完、`version:2` で保存）。

---

## 1. データモデル V2（正準）

```js
PROJECT={version:2,name:'oban',
  panels:[PANEL...],          // ルート直下のパネル
  frames:[FRAME...],          // コマ（1段ネスト。FRAMEの中にFRAMEは不可）
  texts:[TEXT...],
  take:{version:1,name,kf:[{x,y,z,dwell,ease}]},
  fx:{grain:0, diffusion:0, para:{on:false,angle:30,strength:0.35}, clickFx:'invert'},  // 撮影処理
}
PANEL={id,name,x,y,h,depth,ord:0,ar,
  seq:null | {n,fps:12,mode:'loop'|'once'|'pingpong',trigger:'always'|'enter'},  // 連番
  parent:null|frameId}        // FRAMEの子か
FRAME={id,name:'FRAME 01',x,y,h,depth,ord:0,ar:1.777,
  quad:[{x:0,y:0},{x:1,y:0},{x:1,y:1},{x:0,y:1}],   // マスク4頂点（フレームローカル正規化、斜めOK）
  par:0.6,                    // 内部パララックス強度 0..1
  fx:{in:'none'|'slide-l'|'slide-r'|'slide-t'|'slide-b', drift:'none'|'push-in'|'pull-out', driftAmt:0.12, kf:null|kfIndex},
  wipe:null|'invert'|'whiteout'}   // 全面カバー時に世界を反転/白飛ばし
TEXT={id,type:'v'|'sub',str,x,y,size:24,parent:null|frameId,kf:null|kfIndex}
```

- **描画順**: `sort by (depth asc, ord asc)`。ルートのpanels/framesは同一リストとして混在ソート
- **座標系**: V1と同じワールド画面単位。FRAMEの子はフレームローカル（フレーム中心原点、フレーム高=1）

---

## 2. V2-A 品質の土台

### 2-1 z-order分離（「前にいる画を前に」）
- `ord` フィールド追加（既定0、新規投入は `max(ord)+1` = 新しいものが手前）
- PLACE選択中: `[`=SEND BACK / `]`=BRING FRONT（ord±1.5して正規化）。チップに `◀BACK/FRONT▶` ボタン併設
- depthは視差専用、ordは重なり専用と割り切る（混ぜない）

### 2-2 undo/redoスタック
- `history=[]`, `hIdx`。全ミューテーション前に `pushHistory()`（JSONスナップショット、上限60）
- `Ctrl+Z`=undo / `Ctrl+Shift+Z` と `Ctrl+Y`=redo
- スライダー（dwell/depth/fx系）は**300msデバウンスで1エントリに合体**（連続ドラッグで履歴が溢れないように）

### 2-3 UIホバー時のホイール
- グローバルwheelハンドラの先頭に `if(e.target.closest('#bar,#cards,#chip,#fx-panel'))return;`（preventDefaultしない=UIが普通にスクロール）。カーソルがUI上にある間はビューもズームしない

### 2-4 ガイド表示（`G`キー / ビルダーのみ・書き出さない）
- 16:9セーフフレーム: アクションセーフ93% + タイトルセーフ90%（細線、mono 8pxでラベル）
- 中央十字 + 三分割線（十字より薄く）
- ONOFF状態は localStorage に記憶

### 2-5 KFフォーカスと行き来
- `selKf` 概念を追加。カードクリック=選択（--iceの太枠+カード背景薄光）。`▸GO`は従来通り
- **`↑`/`↓`**: selKf±1して**自動GO**（カメラがそのKFへ、カードはscrollIntoView）
- キャンバス側: 選択KFの点に二重リング+dwell窓の帯を表示。「いまどこを触っているか」を常に可視化

---

## 3. V2-B シーケンスとコマ

### 3-1 連番シーケンスパネル
- ドロップ時に `^(.*?)(\d{3,5})\.(png|webp|jpe?g)$` でグループ化。**同prefixが4枚以上→1つのseqパネル**に自動集約（トースト表示「○○を24枚のシーケンスとして追加」）。3枚以下はバラのパネル
- 再生: `mode` — `loop`（目パチ・なびき・粉塵パーティクル向け）/ `once`（ワンアクション）/ `pingpong`
- `trigger` — `always`（常時再生）/ `enter`（画面内30%以上に入った瞬間に再生開始。逆スクロールで画面上方へ抜けたらリセット=もう一度入り直すと再演）
- チップ拡張（seq選択時のみ）: MODE select / FPS 4–30 / TRIGGER select / フレーム数表示
- ビューアのローダーはseq全フレームをカウントに含める
- **用途メモ**: 全面に降る粉塵は「大きく置いたloopシーケンス+depth手前+ord最前」で成立する（専用機能は作らない）

### 3-2 四角マスク（quad）
- FRAMEの `quad` 4頂点で `ctx.beginPath→moveTo/lineTo×4→clip()`。斜めコマOK。多角形・円は**やらない**
- 編集: PLACEでFRAME選択→`M`（またはチップのMASKボタン）でマスク編集モード: 4隅にハンドル（--rouge□）、ドラッグで移動。`Shift`=X/Y軸ロック。`Esc`で抜ける
- quadはフレームローカル正規化座標なので、フレームの移動/拡縮とは独立

### 3-3 FRAME＝1段ネストコンポ（本仕様の核）
- **NEW FRAMEボタン**（PLACEモード）: 画面中央に空フレーム追加
- パネルをフレームに入れる: パネルドラッグ中にフレーム上でドロップ→`parent=frameId`（トースト「FRAME 01 に格納」）。チップに `EJECT` ボタン（ルートへ戻す）
- **フレーム内編集**: FRAMEをダブルクリック→**ENTER FRAME**（パンくず `ROOT ▸ FRAME 01` を#barに表示、他要素は40%減光）。中では子パネルだけが選択/移動対象。`Esc`または パンくずROOTクリックで復帰
- **内部パララックス**（コマの窓の中で絵がずれる）:
```js
// フレームのスクリーン矩形R（従来のprect）を出したあと、子を描く:
const lx=(cam.x - f.x)*f.par, ly=(cam.y - f.y)*f.par;      // ローカルカメラ
for(const c of f.children sorted (depth,ord)){
  const k=lerp(0.15,0.85,1-(c.depth??0.5));                 // 奥ほど動かない
  const sx=R.cx + (c.x - lx*k)*R.h*  (c側zi係数);           // R.h=フレーム高px
  // quadでclipしてから描画
}
```
  係数は実装時に見た目で微調整してよいが、**「カメラが右に動くと、窓の中の奥の絵は左に残る」**方向を守る
- 制約: FRAMEの中にFRAME不可（parentがframeのものにframeは入れない）。TEXTは入れてよい

---

## 4. V2-C 演出と撮影

### 4-1 コマのIN演出とドリフト（TENSION CH06のスライドインの型）
- `fx.kf` でKFに紐づけ（nullなら自動: フレーム中心に最も近いKF）
- タイムライン位置: そのKFの**直前travel区間**=IN演出の窓 / **dwell区間**=ドリフトの窓（すべてPの純関数=可逆）
- `in:'slide-l'` 等: travel進行 f(0→1) で `offset=lerp(±1.2画面, 0, ease)`。マスクは動かさず**中身と枠ごと**スライド
- `drift:'push-in'`: dwell進行で `scale 1→1+driftAmt`（ゆっくり寄る）。`pull-out` は逆。イージングは smooth 固定
- チップ（FRAME選択時）: IN select / DRIFT select / AMT range / KF番号

### 4-2 カバーワイプ（TENSION CH05→06 の明暗逆転の型）
- `wipe:'invert'`: そのフレームが `push-in`（または travel 中の接近）で**ビューポートを100%覆った瞬間**、世界が反転する
  - 実装: `buildTake()` 後に P を0..1で400サンプルし、対象フレームのスクリーン矩形がビューポートを包含する最小Pを `wipeP` として記録（決定的・可逆）。`P>=wipeP` で `world.invert=true`
  - 反転の実体: 背景 `#0c061d ⇄ #f4f1ea`、パネル描画後に `ctx.globalCompositeOperation='difference'` で白矩形…ではなく**シンプルに** stage全体へ CSS `filter:invert(1) hue-rotate(180deg)` をトグル（ビルダー/ビューア共通で確実）
- `wipe:'whiteout'`: 同様に覆った瞬間から白フェード（0.4区間で白→次の絵）。ホワイトアウト明け先は次KFの画
- 1プロジェクトに複数ワイプ可。X-RAY的デバッグ: ガイドON時、wipePの位置をKFパス上に◇マークで表示

### 4-3 撮影FX（軽い撮影処理・全体にかかる）
新モードタブ **`3. FX 撮影`**（--iceとは別の琥珀色 `#FFC94F` を撮影系アクセントに）。パネルは右側（cardsと同じ場所を切替）:

| FX | UI | 実装 |
|---|---|---|
| **GRAIN**（スクリーントーンのざらつき） | 0–1スライダー | 256²ノイズcanvasを4種事前生成、毎フレームランダム選択+オフセットでタイル描画。`globalCompositeOperation='overlay'`（非対応なら'soft-light'）、alpha=grain*0.18 |
| **DIFFUSION**（アニメ撮影のディフュージョン） | 0–1スライダー | ワールドcanvasを1/2解像度にコピー→`filter:'blur(8px) brightness(1.15)'`→`'screen'`でalpha=diffusion*0.55重ね |
| **PARA**（パラ=画面の隅を落とすグラデ露出） | ON/角度/強さ | 角度方向のlinear-gradient（黒→透明）を`'multiply'`で重ねる。強さ=グラデ終端alpha |
- FXはビルダーのプレビューに常時反映+EXPORTに焼き込み（ビューアも同じパイプライン）。パフォーマンス予算: FX全ONで60fps（M1級）。diffusionはoffscreen半解像度必須

---

## 5. V2-D テキストと反応

### 5-1 縦書きテキスト（読み文字）
- TAKEとは独立の **`T` ボタン/キー**: クリック位置に縦書きテキスト追加→その場で入力（DOM contenteditable を座標に重ねる方式。確定でcanvas描画データ化）
- 描画: canvasに1文字ずつ縦積み `ctx.fillText`（`font: bold {size}px var(--jp)`、長音「ー」は90°回転）。色は白/黒トグルのみ（にじみ縁取り: 4方向1pxシャドウ）
- 選択/移動/削除はPLACEモードでパネル同様（ordはテキストが常に最前グループ）

### 5-2 EN字幕トラック（HANGARの解説の扱い）
- `type:'sub'`: KFに紐づく英語字幕。カード側で入力（テキスト欄、kf選択）
- ビューア: 画面下部中央 mono 11px、そのKFのdwell窓でフェードイン/アウト。**`?sub=0`** で非表示、ビューア右下に `SUB ON/OFF` 小ボタン
- ビルダーのプレビューでも同じ表示

### 5-3 ビューアのクリックFX（LPが反応する）
- `fx.clickFx`: `'invert'`（0.18秒 filter:invert 白黒反転フラッシュ）/ `'white'`（ホワイトパルス）/ `'none'`
- pointerdownで発火（ATTRACT解除と共存: 両方起きてよい）。連打はキュー1つ（多重発火しない）
- ビルダーのFXタブで選択+その場でテスト可能

---

## 6. UI/キーマップ（V2完成形）

| キー | 動作 |
|---|---|
| `1`/`2`/`3` | PLACE / TAKE / FX 撮影 |
| `C` | CAPTURE（TAKE） |
| `Space` | PREVIEW（ループ・触ると停止） |
| `↑`/`↓` | KF選択±1 + GO（TAKE） |
| `[`/`]` | 選択要素を背面/前面へ（PLACE） |
| `M` | マスク編集（FRAME選択時） |
| `T` | 縦書きテキスト追加 |
| `G` | ガイド（セーフフレーム/十字/三分割） |
| `Ctrl+Z` / `Ctrl+Shift+Z`/`Ctrl+Y` | undo / redo |
| `Delete` | 選択削除 / `Esc` フレーム内→ルート→選択解除→プレビュー停止（カスケード） |
| ダブルクリック | FRAMEに入る |

配色: 配置=--rouge / カメラ=--ice / 撮影=#FFC94F / 背景PLUM（KINETIC STAGE準拠）。

## 7. リスクと判断メモ

- **ビューア肥大**: V2フル搭載でテンプレート~400行。ビルダーとビューアの描画コア（renderWorld/fxパイプライン/シーケンス再生）は**同一ソースの複製**とし、変更時は必ず両方に同じdiffを当てる（CLAUDE.mdに明記）
- **`filter:invert`** はcanvas全域に効くため撮影FXと重なっても破綻しないが、GRAINのoverlay合成は反転後に不自然になる場合がある→反転中はGRAINを'soft-light'に落とす
- **once+enterトリガー**は時間再生（スクロール純関数ではない）。マスコット文法（LP_Model_CR）と同じ割り切りであることをCLAUDE.mdに書く
- **contenteditable on canvas座標**: file://でIME入力可を確認（Chrome/Edge想定）。NGならprompt()フォールバック
- V1プロジェクトの `panels[].parent` 等の欠損は読み込み時に補完。**書き出しは常にversion:2**
