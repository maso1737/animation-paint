# EXPORT WEB 仕様書 — モーションコミック HTML ビューア（Phase 1）

COMPOSER（`composer.html`）に「EXPORT WEB」ボタンを追加し、**単一HTMLのスクロール駆動ビューア**を書き出す機能の実装仕様。この文書だけで実装が完結するように書いてある。実装前に composer.html の該当行を必ず読むこと。

## 0. ゴール（ユーザー体験）

書き出された HTML を開くと：

1. コンポジションが中央にレターボックス表示される（黒背景）。
2. **ページをスクロールすると、タイムラインがスクラブされる**。カメラワーク（パン/ドリー/ズーム/回転）も、ANIMATOR で描いたパラパラアニメのコマ送り（track.frames の進行）も、すべてスクロール位置に同期して動く。
3. **マウスを動かすとカメラが微追従**し、Z 深度差のあるレイヤーが多層パララックスする（COMPOSER の透視計算そのまま）。
4. 外部依存ゼロ。`file://` 直開きでも GitHub Pages でも動く。全画像は dataURL 埋め込み。

## 1. UI 追加（composer.html）

- トップバーの `#btn-export-video`（composer.html:336 付近）の**隣**に `<button id="btn-export-web" disabled>EXPORT WEB</button>` を追加。disabled 解除は既存 export 系ボタンと同じ箇所で行う（`btn-export-video` の disabled を触っている箇所を grep して同列に追加）。
- クリックで**設定モーダル**（既存の app 風モーダルのスタイルを流用。native `confirm`/`prompt` は禁止）を開く：

| 項目 | ID | 型/範囲 | 既定値 |
|---|---|---|---|
| スクロール感度 | `web-px-per-frame` | number 4–100 (px/フレーム) | 20 |
| マウスパララックス強度 | `web-parallax` | number 0–200 (コンポpx、0=OFF) | 40 |
| 画質 | `web-quality` | 0.5–1.0 (WebP quality) | 0.8 |
| マーカーをチャプター表示 | `web-chapters` | checkbox | ON |
| 推定ファイルサイズ | `web-est-size` | 表示のみ | — |

- 推定サイズ：エンコード前は「ユニーク画像数 × 解像度からの概算」でよいが、**書き出し実行後に実サイズ（`blob.size`）を flashLive 等で報告**すること。
- 「EXPORT」実行中は既存の EXPORT OVERLAY（composer.html:205 付近のスタイル）を流用して進捗表示。画像エンコードは枚数ループなので `i/N` を出す。

## 2. 書き出しデータの収集

### 2.1 範囲と対象

- フレーム範囲 = **ワークエリア** `state.workArea.startFrame`〜`endFrame`（composer.html:550, 931-936）。ビューア内ではこの範囲を 0 起点に正規化する。
- 対象トラック = `visible` なトラックすべて（`solo` はビューアでは**無視**。カメラは `type:'camera'` かつ visible のとき有効）。
- `state.width/height/bgBright/fps`、グローバルマーカー `state.markers`（チャプター用、範囲内のみ、frame は 0 起点に補正）。

### 2.2 画像のユニーク化（重要）

`track.frames[i].image` は**セルごとに共有された同一 Image オブジェクト**（composer.html:882-888。同 duration 中は同じ `img` 参照）。よって：

1. トラックごとに `Map<Image, index>` でユニーク画像を集める（`kind==='draw'` かつ `!hidden` のフレームのみ）。
2. 各ユニーク画像を canvas に描いて `toDataURL('image/webp', quality)` でエンコード。**戻り値が `data:image/webp` で始まらない場合（Safari 等のフォールバック）は `toDataURL('image/png')` を使う**。canvas サイズは原寸（`img.naturalWidth/Height`）。`imageSmoothingEnabled=false`。
3. フレーム列は `frameMap`: 範囲内の各フレーム → ユニーク画像 index（hidden/empty は `-1`）の整数配列にする。`type:'image'` の静止画トラックは画像1枚＋全フレーム同 index になるので自然に軽くなる。

### 2.3 埋め込み JSON（`VIEWER_DATA`）

```js
{
  width, height, bgBright, fps,
  totalFrames,                       // ワークエリア長
  markers: [{f,label}],              // 0起点
  camera: {keyframes} | null,        // カメラトラックのKF（範囲補正はしない※下記）
  tracks: [{                         // 描画順 = state.tracks の順（カメラ除く）
    name, tid, parent,               // parent は tid 参照（親が非表示/範囲外でもKF評価に必要なので、親トラックは非visibleでも keyframes と width/height だけは含める。frameMapは空でよい＝描画されない）
    width, height,                   // track.width/height（無ければ comp サイズ）
    keyframes,                       // ALL_PROPS 8種の {f,v,ez} 配列
    frameMap: [int...],              // 長さ totalFrames
    images: ["data:image/webp;...", ...]
  }],
  cfg: { pxPerFrame, parallax, chapters }
}
```

※ KF の `f` はワークエリア補正せず**絶対フレームのまま**入れ、ビューア側で `frameIdx + startFrame` を評価に使う方式でもよい（`startFrame` を cfg に入れる）。どちらでも可だが、frameMap とKF評価で基準がズレないよう**必ず統一**すること。

## 3. ビューア HTML の生成

`buildViewerHTML(data)` を composer.html に追加。テンプレートリテラルで単一 HTML 文字列を組み立て、`Blob` → `a.download = 'motion_comic.html'`（既存の動画 DL 導線 composer.html:1967 と同型）。

### 3.1 移植する関数（COMPOSER が正準）

以下を**文字列として埋め込む**（コピーして `state` 参照を `VIEWER_DATA` 参照に書き換える。ロジック改変禁止）：

- `getKfValue` / `getTransform`（composer.html:588-607）— ez イージング含めそのまま
- `PERSP_FOCAL = 1000`（composer.html:530）
- `applyCamWrap`（1627-1634）
- `applyTrackChain`（1637-1658）— `state.tracks.find(...)` を data.tracks 検索に置換
- `drawOneTrack` / `drawFrame`（1659-1688）— solo 分岐は削除。`f.image` はプリロード済み `Image` の配列参照（`images[frameMap[fi]]`）に置換。`frameMap[fi] === -1` なら描画スキップ
- `ALL_PROPS` と `propDefOf` 相当の既定値（getKfValue が KF 空のとき返す既定値の仕組みを丸ごと持っていく）

### 3.2 ビューアの構造

- `<canvas>` は `position:fixed` 中央、CSS で `max-width:100vw; max-height:100vh` レターボックス。内部解像度は `data.width×height`。
- スクロール領域：`document.body` の高さ = `totalFrames × pxPerFrame + 100vh`。
- **rAF ループ**：
  - `targetFrame = scrollY / pxPerFrame`、`smoothFrame += (targetFrame - smoothFrame) * 0.15`
  - マウス：`pointermove` で `mx,my`（-1〜1 正規化、画面中央基準）。`smx += (mx - smx) * 0.05`
  - 描画フレーム番号 = `clamp(round(smoothFrame), 0, totalFrames-1)`
  - カメラ変換 = `getTransform(frame, camera.keyframes)`（カメラ無しなら `{x:0,y:0,z:0,s:1,rot:0,ax:0,ay:0,op:1}`）に **`cam.x += smx * parallax; cam.y += smy * parallax`** を加算してから `drawFrame` へ渡す
  - 前回とフレーム・パララックス値が変わらないときは再描画スキップ（誤差 0.1px 未満は同値扱い）
- **ロード**：全 Image を `decode()` して Promise.all 完了までプログレスバー（`n/N`）。完了後にフェードインして rAF 開始。
- **チャプター**（cfg.chapters 時）：右端に縦のドット列。各マーカー位置 `f × pxPerFrame` へ `scrollTo({behavior:'smooth'})`。ホバーで label 表示。スナップは実装しない（Phase 2）。
- タッチデバイス：ネイティブスクロールに任せる。パララックスはポインタが無ければ単に 0（ジャイロは Phase 2、**未実装であることを書き出しモーダルに注記**）。

## 4. 制約・エッジケース

- カメラトラック無しでもエラーにしない（スクラブのみ＋パララックスは全レイヤー等倍で効く＝Z 差が無ければ揺れは平行移動になる。それで正しい）。
- `depthPx < 1`（カメラ面より手前）は既存どおり非表示。
- トラック 0 本 or workArea 長 0 のときはボタンを押した時点で flashLive でエラー表示して中断。
- 生成 HTML 内の `</script>` 文字列問題：JSON を `<script type="application/json">` に入れる場合は `<\/script>` エスケープ必須。テンプレートリテラル内のバッククォート/`${}` エスケープにも注意。
- 数百 MB 級になる入力（多コマ×4K）はあり得る。エンコード後の合計が 200MB を超えたら警告を出して続行確認（app 風モーダル）。

## 5. 検証（必須。「実装しました」で終わらせない）

1. `node tools/check.js` がパスすること（生成テンプレート内の JS は check 対象外だが、composer.html 本体の構文/配線チェックは必須）。
2. サンプルプロジェクト（アニメトラック2枚以上＋Z 差＋カメラ KF＋マーカー）で実際に書き出し、preview ツール等でブラウザ検証。
3. **動作チェック表**を報告に添付：

| パラメータ | 期待される変化 | 有効になる条件 | 結果 |
|---|---|---|---|
| スクロール感度 最小⇔最大 | 同じスクロール量で進むフレーム数が変わる | 常時 | |
| パララックス 0⇔最大 | 0で完全静止／最大でマウス追従が明確 | ポインタのあるデバイス | |
| 画質 0.5⇔1.0 | ファイルサイズと画質が変化 | 常時 | |
| チャプター ON/OFF | ドット列の表示/非表示、クリックでジャンプ | マーカーが範囲内に存在 | |
| スクロール→コマ送り | 手描きシーケンスの絵が切り替わる | frameMap に複数画像 | |
| スクロール→カメラ | パン/ズーム/回転がKFどおり進行 | カメラトラック visible | |
| Zパララックス | 奥のレイヤーほど動きが小さい | トラックに Z 差 | |

条件付きでしか効かない項目（パララックス×タッチ端末、チャプター×マーカー無し）は、モーダル側に条件を注記してグレーアウトまたはラベル併記すること。

## 6. スコープ外（Phase 2 以降 — 今回は実装しない）

ループ区間、ジャイロパララックス、マーカースナップ、複数プロジェクト連結、テキスト/フキダシレイヤー、重複画像のトラック間共有。

## 7. コミット

実装完了・検証済み後も **push は明示依頼があったときのみ**（CLAUDE.md 準拠）。
