# SPEC_06 — SATSUEI KIT（撮影処理キット）＋ OBAN→COMPOSER コンバータ

撮影処理（ディフュージョン/パラ/色調/グレイン/ビネット/モーションブラー/DOF/グリッチ…）を
**専用アプリではなく「1つの正準実装＋スキル」**として定義し、必要なアプリに焼き込む。
`vj-audio-export-kit` / `camera-rig-orbit-capture` と同じ運用。単一HTML・コード複製方針に準拠。

- 正準実装: `satsuei-fx-kit` スキルの `references/satsuei_fx_core.js`（**構文チェック＋スタブスモーク済み**）。
  P0実装後は `composer.html` 内のコピーが「生きた正準」になる（スキルの references は固定スナップショット）
- 行番号は変動するため、**統合時は必ずシンボル名で grep** すること（このSPECの行番号は 2026-07-12 時点の目安）

## 0. 思想（変えないこと）

- 撮は**カメラ側のグローバル後処理1本（fxチェーン）だけ**。レイヤーごとのエフェクトスタック・
  調整レイヤー・トラックマットは**やらない**（AE化しない歯止め。MOTION_COMIC_SPEC の
  「撮影処理はやらない」条項は本SPECで「**撮=グローバルチェーン1本のみ許可**」に改訂）
- AEルートは残す: 撮チェーン `enabled:false`（既定）なら従来と完全に同一出力
- 対応表:

| ルート | 撮の掛かり方 | tier |
|---|---|---|
| animator→composer→AE | 撮OFF（従来どおり連番持ち込み） | — |
| animator→composer→動画/連番 | EXPORT時にfxチェーン適用 | VIDEO=rt / PNG連番=final |
| animator→OBAN→LP | ビューア実行時にfxチェーン（リアルタイム） | rt |
| animator→OBAN→composer→動画 | P3コンバータでTAKE→CAMERAトラック化→上と同じ | final |
| composer EXPORT WEB | ビューア実行時（P2の後、任意P2b） | rt |

## 1. tier（品質段階）

| tier | どこで走るか | 内容 |
|---|---|---|
| `draft` | composerプレビュー | frame型のみ・mb/dofスキップ可。編集中の当たり確認 |
| `rt` | LPビューア / EXPORT VIDEO（実時間レンダのため） | frame型＋**mb近似**（カメラ速度の方向ブラー8tap） |
| `final` | EXPORT PNG連番 | 全部＋**mb本物**（サブフレーム蓄積・シャッター角準拠）＋layer DOF |

## 2. FXスキーマ（v1・全アプリ共通）

```js
fx = {
  version: 1,
  enabled: false,            // false = 完全素通し（後方互換の要）
  chain: [                   // 順序 = パス実行順。エフェクト追加 = エントリ追加
    { t:'mb',        on:false, p:{ shutter:180, samples:8, rtStrength:1.0 } },
    { t:'dof',       on:false, p:{ focusZ:0, range:400, maxBlur:12 } },
    { t:'diffusion', on:true,  p:{ threshold:0.55, radius:1.0, gain:0.8, mode:0 } },
    { t:'para',      on:false, p:{ angle:120, c1:'#ffc890', a1:0.30, c2:'#281866', a2:0.25, mid:0.5, soft:0.45 } },
    { t:'grade',     on:true,  p:{ lift:0, gamma:1, gain:1, sat:1, temp:0 } },
    { t:'vignette',  on:true,  p:{ amount:0.25, soft:0.55 } },
    { t:'grain',     on:true,  p:{ amount:0.06, size:1.5, mono:1 } },
    { t:'glitch',    on:false, p:{ amount:0.15, bands:24, rgb:0.5, speed:8 } }
  ]
}
```

- **前方互換**: 未知の `t` は console.warn してスキップ（実装済み）。古いアプリで新しいプロジェクトを開いても壊れない
- **保存場所**:
  - composer: `state.fx`。`saveJSON`（`workArea:` を保存している箇所、composer.html:1911 付近）に `fx: state.fx` を追加。loadJSON側は `fx:` が無ければ `makeDefaultFx()`
  - OBAN: `PROJECT.take.fx`（TAKEに従属。`migrate()` で欠損補完）
  - 書き出しビューア: JSONごと埋め込み

## 3. エフェクトの3型（拡張の骨格）

| 型 | 実行場所 | 該当 | 追加のしやすさ |
|---|---|---|---|
| `frame` | GPUフルフレームパス（コア内） | diffusion / para / grade / vignette / grain / glitch / mb(rt) | ◎ シェーダ1個＋switch1ケース |
| `layer` | 合成前・レイヤー単位（ホスト側） | dof | ○ `dofBlurPx()` ヘルパで `ctx.filter` |
| `temporal` | 書き出しループ（ホスト側） | mb(final) | △ ループ改修（§6.2、1回だけ） |

**新frame型エフェクト追加手順**（パラ・グリッチ級なら30分作業。コアのコメントにも記載）:
1. `FX_DEFS` に `{label, tier, ui:[{k,label,min,max,step}...]}` を追加
2. `FRAG` に同名フラグメントシェーダを追加
3. `api.apply` の switch に uniform 設定ケースを追加
4. `makeDefaultFx().chain` に既定エントリ追加（`on:false`）
→ **FXモーダルUIは FX_DEFS から自動生成なのでUI改修ゼロ**。将来: zoomblur / chromatic aberration / posterize / halftone など

## 4. コアAPI（satsuei_fx_core.js）

```js
const fx = createSatsueiFX();           // WebGL1。不可なら fx.available=false
const out = fx.available ? fx.apply(srcCanvas, fxParams, info) : srcCanvas;
// out = fx.canvas（WebGL canvas）。ctx.drawImage(out,0,0,w,h) で表示/書き出し先へ
// info = { t:秒(グレイン/グリッチのアニメ用), velX, velY(uv/フレーム, rtモーションブラー用) }
dofBlurPx(z, dofParams)                 // → ctx.filter='blur(Npx)' 用の半径
makeDefaultFx(); FX_DEFS;               // window.SatsueiFX に公開
```

注意点（実装済みの決定事項）:
- `preserveDrawingBuffer:true`（applyの直後にdrawImageで読むため）
- アップロードは `UNPACK_FLIP_Y_WEBGL=true`。y方向を使うシェーダはデザイン空間 `pd=vec2(uv.x,1-uv.y)` で計算
- 解像度変更は `apply` 内で自動（resize）。頻繁な解像度切替はターゲット再生成コストがあるので、プレビューと書き出しで別インスタンスにしてもよい

---

## 5. P0 — composer 統合（最優先）

**ゴール**: FXボタン→モーダルでチェーン編集、プレビュートグル、EXPORT VIDEO/PNG に撮が乗る。

1. **コア貼り付け**: `satsuei_fx_core.js` 全文を composer.html の `<script>` 冒頭部（`'use strict'` の直後あたり）へ複製。マーカーコメント `/* ===== SATSUEI FX CORE (satsuei-fx-kit) ===== */` 〜 `/* ===== /SATSUEI FX CORE ===== */` で囲む（更新時にブロックごと差し替えるため）
2. **state**: `state.fx = SatsueiFX.makeDefaultFx()`。`saveJSON`/`loadJSON`/IndexedDB系（`workArea` と同じ経路を grep）に追従。PROJECT_v2 再IMPORT時の欠損は `makeDefaultFx()` で補完
3. **UI**: topbar `#btn-export-video`（composer.html:359）の左に `<button id="btn-fx">FX</button>` と `<button id="btn-fx-preview" class="toggle">FX PREVIEW</button>`。
   - FXモーダル（既存のapp風モーダル流用。native confirm/prompt 禁止）: `state.fx.chain` を順に列挙。各行 = ON/OFFチェック＋`FX_DEFS[t].label`＋`ui` 配列から `<input type=range>` を自動生成（`FX_DEFS[t].colors` があれば `<input type=color>` も）。`enabled` のマスタートグルを最上部に
   - スライダー変更は即 `state.fx` へ反映＋debounce保存（`commitD` 相当）。undo対象外でよい（v1）
4. **プレビュー（draft）**: `drawCurrentFrame()`（composer.html:1745）を改修:
   - FX PREVIEW ON かつ `state.fx.enabled` のとき、`drawFrame` を直接 `ctx` ではなくオフスクリーン `gFxSrcCanvas`（state.width×height, 使い回し）へ描き、`fx.apply(gFxSrcCanvas, state.fx, {t:state.currentFrame/state.fps})` の結果を `ctx.drawImage`。OFF時は従来経路そのまま
   - draft では `mb` はスキップ（velX/velY を渡さなければ自動で素通し＝実装済み挙動）
5. **EXPORT VIDEO（rt）**: `exportVideoComposer()`（composer.html:1979）のループ内 `drawFrame(c2d,OUT_W,OUT_H,i)`（2001, 2010）を:
   ```js
   drawFrame(cc2d, OUT_W, OUT_H, i);              // cc = 使い回しオフスクリーン
   const cam0=getCamAt(Math.max(0,i-1)), cam1=getCamAt(i);
   const info={ t:i/state.fps,
     velX:cam1&&cam0 ? (cam1.x-cam0.x)/state.width  : 0,   // コンポpx → uv
     velY:cam1&&cam0 ? (cam1.y-cam0.y)/state.height : 0 };
   const out = (state.fx.enabled&&gFx.available) ? gFx.apply(cc, state.fx, info) : cc;
   c2d.drawImage(out, 0, 0, OUT_W, OUT_H);        // oc（captureStream対象）へ
   ```
   実時間レンダのため tier=rt（mbは近似のまま）。4KでGPUパスが間に合わない環境向けに、モーダルに「EXPORT時FX OFF」チェックは付けない（`enabled` で足りる）
6. **EXPORT PNG（final・P1まではrtと同じ）**: `exportPNG()`（2028）の `drawAt` コールバックに同じフックを入れる
7. **検証**: `node tools/check.js` 必須。目視: FX全OFF＝従来出力とピクセル一致（enabled:false は apply 未呼び出し経路であることをコードで保証）

## 6. P1 — final品質（MB本物＋DOF）

### 6.1 前提: drawFrame の小数フレーム対応
- `getKfValue`（composer.html:644）は線形補間なので小数フレームを渡しても正しい値を返す（**floor していないことを実装確認**）
- `drawOneTrack`（1715）の**セル参照だけ** `const fi=Math.min(Math.floor(frameIdx), track.totalFrames-1)` に変更（現状は小数だと `frames[2.5]`=undefined で絵が消える）。`applyTrackChain`/`getCamAt` へは小数のまま渡す＝カメラとトランスフォームだけサブフレーム補間され、作画のコマは踏み替えない（アニメ撮として正しい挙動）

### 6.2 MB final（temporal・exportPNGのみ）
`exportZipPNG` の `drawAt` を差し替え:
```js
const e = state.fx.chain.find(e=>e.t==='mb');
if(state.fx.enabled && e && e.on){
  const N=e.p.samples||8, win=(e.p.shutter||180)/360;   // シャッター角→フレーム窓
  acc2d.clearRect? null : 0; // acc = 使い回しオフスクリーン
  for(let k=0;k<N;k++){
    const sub = i - win/2 + win*(k+0.5)/N;              // i を中心に前後へ開く
    drawFrame(cc2d, OUT_W, OUT_H, Math.max(0, sub));
    acc2d.globalAlpha = 1/(k+1);                        // 逐次平均（1/N固定でも可）
    acc2d.drawImage(cc, 0, 0);
  }
  acc2d.globalAlpha = 1;
  src = acc;
} else { drawFrame(cc2d, OUT_W, OUT_H, i); src = cc; }
// → 以降 P0 §5-6 と同じ fx.apply（infoのvelは渡さない＝rt mbは走らない）
```
※ mbエントリがfinal経路で有効な間は frame型の`mb`ケースを走らせない（vel未指定なら素通し＝実装済み）

### 6.3 DOF（layer型）
`drawOneTrack`（1715）の `tCtx.drawImage(f.image, ...)` 直前:
```js
const dof = state.fx.enabled && state.fx.chain.find(e=>e.t==='dof' && e.on);
if(dof && gFxTier!=='draft' && !track.parent){          // root層のみ（子は親の面に乗る）
  const r = SatsueiFX.dofBlurPx(tr.z||0, dof.p);
  if(r>0.3) tCtx.filter = 'blur('+(r*(tw/state.width)).toFixed(1)+'px)';  // 出力解像度スケール
}
tCtx.drawImage(...); tCtx.filter='none';
```
- `ctx.filter` は高コスト。`gFxTier` は書き出し中のみ `'final'`（プレビューはdraft=DOFなし）
- focusZ はモーダルの FOCUS Z スライダー（トラックZと同じ単位。PERSP_FOCAL=1000 系）

## 7. P2 — OBAN_BUILDER / ビューア統合（rt）

1. **データ**: `PROJECT.take.fx`（§2）。`migrate()` に補完追加。COPY/PASTE PROJ にも自然に乗る
2. **ビルダーUI**: TAKEモード（--ice）のバー右端に `FX` ボタン→FXモーダル（composerと同じ自動生成型。ice配色）。ビルダーの `renderWorld`（oban-builder.html:363）プレビューは**FXなしのまま**でよい（軽量優先）。`Space` のループPREVIEW中のみ適用するオプションは任意
3. **コアの持ち方**: ビルダー内に `<script id="satsuei-core">`（コア全文・**バッククォート/`</script>` 文字列を含まない**ことは確認済み）として置き、ビルダー自身もそこから使う。`viewerHTML()`（783）でその `textContent` をビューアへ注入:
   ```js
   const CORE = document.getElementById('satsuei-core').textContent;
   // テンプレート内: <script>${CORE}${S}\n<script> ... 既存ビューアJS
   ```
   （ビューアテンプレートの生バッククォート禁止ルールは維持される）
4. **ビューア `frame()` 改修**（テンプレート内 908 付近）:
   - シーン描画先を可視 `cv` からオフスクリーン `sc`（同サイズ2D、onResizeで追従）へ変更
   - 描画後: `const out = FXOK ? fx.apply(sc, FXP, {t:tS, velX:clamp(cam.x-pcx,-.05,.05), velY:clamp(cam.y-pcy,-.05,.05)}) : sc; cx.clearRect(...); cx.drawImage(out,0,0);`（`pcx/pcy`=前フレームのcam、初期化は現在値。OBANの cam.x/y は画面幅/高さ比なのでそのまま uv/フレーム）
   - `FXOK = FXP && FXP.enabled && fx.available`。`fx.available=false`（WebGL不可）なら sc を直描き＝**素通しフォールバック必須**
   - DOF（任意・軽量版）: `drawP` で `depth→z写像`（§8）→ `dofBlurPx` → `cx.filter`。モバイル配慮で v1 は省略可
5. **検証**: ビューア実生成→構文チェック→スクロール往復＋ATTRACTスモーク（既存の検証手順）。`?fx=0` クエリでFX強制OFFを付ける（性能問題の逃げ道）

## 8. P3 — OBAN→COMPOSER コンバータ（TAKEのキー化）

**MVP = カメラだけ変換**（パネルの絵はcomposer側で別途IMPORT。全自動化はP3bで検討）。

1. **入口**: OBAN_BUILDERに `COPY FOR COMPOSER` ボタン → PROJECT_v2 互換JSONをクリップボード → composerの IMPORT JSON に貼る（既存動線に乗る。ファイル授受なし）
2. **時間軸**: モーダルで `fps`（既定24）と `尺（秒)` を指定 → `totalFrames = fps*秒`。`buildTake()` の各セグメント `p0/p1` を `f = round(p * totalFrames)` に展開
3. **キー生成**（`type:'camera'` トラック1本、`frames:[]`, `projectId:null`）:
   - dwell区間: 始点/終点に**同値キー2枚**（ホールド）
   - travel区間: 終端キーに ease写像 — `linear→ez:0` / `smooth→ez:0.5` / `inout→ez:1` / `outCubic・inCubic→ez:0.5`（composerのezはsmoothstepブレンドなので近似と割り切る。SPEC注記）
4. **値の写像**（OBAN cam → composer CAMERAプロパティ）:
   - `X = oban.x * Wc`、`Y = oban.y * Hc`（Wc/Hc=コンポ解像度。OBANのx/yは画面幅/高さ比）
   - **ズームは案A採用: `SCL = oban.z`**（光学ズーム）。OBANの depth別ズーム視差（`zi=1+(z-1)*lerp(0.55,1.25,depth)`）は失われるが構図とタイミングは保たれる。視差が欲しいカットは composer 側で Z ドリーに手動で振り直す（`Z ≒ PERSP_FOCAL*(1-1/oban.z)` が出発点）
   - パネルを composer に持ち込む場合のレイヤーZ写像（参考値）: `z = PERSP_FOCAL*(1/lerp(0.7,1.2,depth) - 1)` → depth 0→+428.6 / 0.5→+52.6 / 1→-166.7
   - `take.fx` はそのまま `fx:` へコピー（スキーマ共通なので変換不要）
5. **検証**: OBANのPREVIEWとcomposerの再生を並べて目視。dwell位置のフレーム一致を数点確認

## 9. 実装フェーズまとめ

| Phase | 内容 | 規模感 |
|---|---|---|
| P0 | コア複製＋composer: state/FXモーダル/プレビュー/EXPORT VIDEO・PNG(rt) | 中 |
| P1 | 小数フレーム対応＋MB final蓄積＋DOF layer | 小〜中 |
| P2 | OBAN: take.fx＋ビルダーFXモーダル＋ビューア注入(rt)＋`?fx=0` | 中 |
| P2b(任意) | composer EXPORT WEB ビューアにも同じrt統合 | 小 |
| P3 | COPY FOR COMPOSER（TAKE→CAMERAトラックJSON） | 小 |
| P3b(検討) | パネル画像の自動トラック化（seq/quadマスクは対象外） | — |

## 10. 受け入れ基準

- [ ] `fx.enabled:false` で全アプリ従来出力とピクセル一致（applyを呼ばない経路）
- [ ] WebGL不可環境で素通し動作（ビューア含む）
- [ ] composer: FX全ON で EXPORT VIDEO / PNG連番 が完走、`node tools/check.js` 緑
- [ ] PNG連番: mb on/off でブラー差が目視確認できる（シャッター180°）
- [ ] OBANビューア: file://直開き・ATTRACT走行・`?fx=0` すべて動作、60fps近辺（FHD）
- [ ] コンバータ: dwell/travel のタイミングがOBANプレビューと体感一致
- [ ] 新エフェクトを FX_DEFS+FRAG+switch+default の4点セットだけで追加できる（zoomblurで素振りして確認）
