# SPEC 07 — ANIMATOR ⇄ OBAN BUILDER ブリッジ（PNG書き出しなしの連番往復）

> **この仕様書単体で実装可能なように書いてある。** 行番号は変動するため**必ずシンボル名でgrep**。
> 実装前に読むもの: `Animation_Paint/animator.html`（`buildProjectPayload` / `setupLiveSync` 周辺）、
> `Animation_Paint/composer.html`（`liveUpdateTrack` / `onLiveProjectUpdate` / `EX_DB` 周辺 — **受信側の正準**）、
> `OBAN_BUILDER/oban-builder.html`（`addFiles` / `IMGCACHE` / `panelImg` / `migrate` / `viewerHTML`）、`PIPELINE.md`。

## 0. 目的と大原則

- **目的**: animatorで描いた連番を、PNG書き出しせずにOBANのseqパネルとして使い、**animator側の修正が自動でOBANに反映**される（composerと同格の連携）
- **大原則: animator.html は改修ゼロ**（B3のみ例外・数行）。既存プロトコルをOBANが「composerのふりをして」話す。
  animatorは `composer-hello` / `request-sync` を受けると `project-update` を返す実装が既にある（`setupLiveSync` 参照）——OBANはこれをそのまま利用する
- OBANの保存データにピクセルは持たない（従来どおり参照のみ）。**絵の実体は常にanimatorのものが正**

## 1. 既存プロトコル（検証済み・変えないこと）

### 交換ストア（コールドスタート用）
- IndexedDB: `EX_DB='tdr_exchange'` / `EX_STORE='projects'` / `EX_VERSION=1` / keyPath=`projectId`
- animatorが保存時に最新PROJECT_v1ペイロードを書く。composerは起動時にここから読む（`indexedDB.open(EX_DB,...)` をcomposer.htmlでgrepし、**同じ読み出しコードを複製**する）
- 注意: file://運用のため**同一ブラウザ限定**（Chrome同士ならタブ/ウィンドウ間で共有される。既知の制限としてAnimation_Paint/CLAUDE.mdに記載あり）

### ライブ同期（ホットアップデート）
- `BroadcastChannel('tdr_live')`。メッセージ:
  - `{type:'animator-hello', projectId}` … animator起動通知
  - `{type:'composer-hello'}` … 受信側起動通知 → animatorは `gLiveActive=true` にして全ペイロード送信
  - `{type:'request-sync', projectId?}` … 指定IDの最新を要求（省略時は全部）
  - `{type:'project-update', projectId, payload}` … 本体。payload = PROJECT_v1
- **OBANも同じチャンネル・同じ語彙に参加する**（新メッセージ型は追加しない。animatorから見ればOBANはもう1つのcomposer）

### PROJECT_v1 ペイロード（`buildProjectPayload` が正）
```js
{ format:'PROJECT_v1', projectId, fps:24, width, height,
  workArea:{startFrame,endFrame},
  cells:[ {id,kind:'empty',duration} | {id,kind:'draw',duration,hidden,image:<PNG dataURL(alpha付)>} ],
  frameLayer:{...}, composer:{...} }   // OBANはcells/fps/width/height/projectId/workAreaだけ使う
```

## 2. OBAN側データモデル拡張

```js
PANEL.seq = { src:'ap', apId:<projectId>, n:<展開後フレーム数>,
              fps:24, mode:'loop', trigger:'always',
              apName:'', apAt:'' }        // 表示用: プロジェクト名/最終同期時刻
```
- 既存のファイル連番は `src` 無し（互換）。`migrate()` は `seq.src` 未知でも壊さない
- **タイムシート展開**: cells の `duration` を展開して per-frame の参照列にする
  `frameRefs=[cellIdx,...]`（`kind:'empty'`は直前セル維持=空なら透明）。`n=frameRefs.length`
  （workAreaで切るオプションはv1では**やらない**——全カット。fps初期値24=原寸再生、ユーザーが変えれば早回し）
- 画像キャッシュ: `IMGCACHE['ap:'+apId+'#'+cellIdx] = {img(dataURLから生成), ar}`
  ランタイム補助 `AP={ [apId]:{frameRefs, cells, w, h} }`（保存しない）
- `panelImg()` に分岐追加: `seq.src==='ap'` → `seqIdx()`（既存ロジック流用）→ `frameRefs[idx]` → IMGCACHE

## 3. 実装フェーズ

### B0 — EX_DB取り込み＋手動RELOAD（コールドパス）
1. バーに **`+ FROM ANIMATOR`** ボタン（--rouge）: EX_DB を `getAll` → モーダルで一覧
   （projectId / cells数 / exportedAt。命名はapp風モーダル、native prompt禁止）→ 選択で ap-seqパネル生成
   （ar=width/height、初期配置・ord・depth は `pushPanel` と同じ規則）
2. ペイロード→展開→IMGCACHE充填 の共通関数 `apIngest(payload)` を作る（B1でも使う）
3. チップ（ap-seq選択時）: 既存SEQ行に加えて `AP:<apName> ●SYNC <apAt>` バッジ＋ **`RELOAD`** ボタン（EX_DBから再取得）
4. **永続化**: localStorage/COPY PROJには `seq`（参照）だけが乗る。起動時 `load()` 後、ap-seqがあれば
   EX_DBから自動 `apIngest`。見つからなければプレースホルダ（既存の点線枠）＋トースト
   「ANIMATORでプロジェクトを開いて LIVE を押すと同期します」

### B1 — tdr_live 自動同期（ホットパス・本命）
1. `setupApLive()`: `new BroadcastChannel('tdr_live')`（`typeof BroadcastChannel==='undefined'` ガード必須）。
   起動時 `postMessage({type:'composer-hello'})` ＋ ap-seqの `apId` それぞれに `{type:'request-sync',projectId}`
2. 受信 `project-update`: apIdが一致するap-seqパネルに `apIngest(payload)` → **絵だけ差し替え**
   （panelの x/y/h/depth/ord/mode/trigger/fps は保持 — composerの `liveUpdateTrack` と同じ思想）。
   `n` が変わったら反映。トースト `● SYNCED <name>`（composerの `flashLive` 相当）
3. 追加コスト規律: OBANからの送信は hello / request-sync のみ。**受け身に徹する**（プロトコル汚染禁止）
4. 目的の体験: **animatorで1コマ直して保存（autosave）→ OBANのループが250ms後に新しい絵で回っている**

### B2 — EXPORTベイク（配布）
- ビューアはEX_DB/BroadcastChannelに依存できない（別PC/別ブラウザ）→ **EXPORT時にap-seqをdataURLで埋め込む**
  1. `viewerHTML()` でap-seqパネルごとに `frames:[dataURL,...]`（重複セルは1回だけ持ち `frameRefs` で参照）
  2. 既定は `canvas→toDataURL('image/webp',0.85)` に再エンコードして軽量化（PNG原寸はオプション）
  3. 書き出し前に概算サイズをトーストで警告（`>50MB` なら確認）
  4. ビューア側ローダー: `seq.src==='ap'` は `frames[]` から `Image` 生成（`./`+name 参照はしない）
- これで「PNG連番を書き出さずにLPまで届く」ルートが完成する

### B3（任意・唯一のanimator改修）— EDIT IN ANIMATOR ディープリンク
- チップに `EDIT IN ANIMATOR` → `window.open('../Animation_Paint/animator.html?open='+apId)`
- animator側: 起動時 `location.search` の `open` を見て該当プロジェクトをIndexedDBからロードする**数行だけ**追加
  （`node tools/check.js` 必須・Animation_Paint/CLAUDE.mdのデプロイ規律に従う。pushは明示依頼時のみ）
- B3未実装の間は同ボタンで animator.html を開くだけ＋トースト「プロジェクトを開いてLIVEを押してください」

## 4. 受け入れ基準

- [ ] animatorでカット作成→OBAN `+ FROM ANIMATOR` で一覧に出て、選択でループ再生される（PNG書き出しゼロ）
- [ ] animatorで1コマ修正→autosave→**OBANのプレビューに1秒以内に反映**（B1、位置/seq設定は保持）
- [ ] OBAN再起動後もap-seqパネルが自動復元（EX_DB経由）。EX_DBに無い場合はプレースホルダで壊れない
- [ ] EXPORTしたoban-viewer.htmlが**別ブラウザ/別PCで**ap-seqを再生できる（B2ベイク）
- [ ] ファイル連番seq・通常パネルの従来動作は完全不変（`src`無しの経路に触らない）
- [ ] BroadcastChannel/IndexedDB不可環境で例外なく素通し（tryガード）
- [ ] 実装後 `PIPELINE.md` の表を更新（animator→OBANの入口、ビューアのベイク出口）＋OBAN_BUILDER/CLAUDE.md追記

## 5. リスクと判断メモ

- **dataURLメモリ**: セル数が多い大判プロジェクトはIMGCACHEが張る。cells原寸（上限4K面積）×コマ数に注意。
  v1は素直に持ち、重ければB2と同じwebp再エンコードを取り込み時にも適用（`AP_INGEST_WEBP=0.85` フラグ）
- **empty cellの意味**: animatorのタイムシートで `empty` は「空白コマ」。展開時は**透明フレーム**として扱う
  （直前セル維持ではない——composerの `parseTrackFromJSON` の解釈をgrepで確認し、そちらに合わせること）
- **同名プロジェクトの多重更新**: project-updateは250ms debounce済み（animator側）。OBAN側の追加debounce不要
- OBANのundo履歴にap絵は乗らない（参照のみ）。RELOADはundo対象外でよい
