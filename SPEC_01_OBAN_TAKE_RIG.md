# SPEC 01 — OBAN TAKE RIG（大判カメラのテイク編集システム / モーションコミックへの道）

> **この仕様書単体で実装可能なように書いてある（会話コンテキスト不要）。**
> 実装前に必ず読むもの: `LP_Model_CR/scroll-model-lp.html`（大判PAN方式の現行実装）、
> スキル `camera-rig-orbit-capture`（CAPTURE式カメラKFの正準）、`edit-mode-spec`（EDITモードUIの正準）、
> `kinetic-stage-design-system`（配色: カメラ系=--ice。ESCカスケードの優先順位）。

## 0. 目的と思想

- LP（`scroll-model-lp.html` 系）の「カメラが大判の上を移動する」演出を、**ワールド（素材配置）とテイク（カメラの旅程）に分離**する
- テイク＝JSONデータ。同じワールドに別テイクを与えれば**別ルートの「別テイク」が量産できる**
- 最終目標は**モーションコミック制作系**: 画像（コマ/パネル）を並べ、カメラの旅程を打つだけで、内容に合った構成・展開のスクロール作品が作れること
- **最重要制約: 触ってすぐわかること。** 操作は「構図を作る → CAPTURE」の2手だけを絶対の基本とし、細かい調整（dwell/ease/z）は後から触れる**オプション**として置く。設定項目を増やすほど失敗という前提で設計する

## 1. 全体ロードマップ（3フェーズ。各フェーズ単体でリリース可能）

| フェーズ | 何ができるようになるか | 実装先 |
|---|---|---|
| **P0 TAKEデータ分離** | カメラパスをJSON化。ビューアとして再生のみ | `scroll-model-lp.html` 改修 |
| **P1 ページ内TAKEエディタ** | Eキーで編集モード→CAPTUREでテイク作成、COPY/PASTEで保存 | 同上に追加 |
| **P2 OBAN BUILDER（単体アプリ）** | 画像フォルダ投入→パネル配置→テイク→単一HTML書き出し（モーションコミック） | 新規 `OBAN_BUILDER/` |

P0→P1は同一ファイル内の増築。P2は別プロジェクト（P0/P1のコードを複製して使う。共有ライブラリ化はしない＝プロジェクト共通方針）。

## 2. データモデル（全フェーズ共通・これが本体）

```js
// TAKE = カメラの旅程。順序付きKF列。scroll全長をパス弧長に正規化して再生する
const TAKE = {
  version: 1,
  name: "take-A",
  world: "hangar-v1",          // ワールドIDとの対応チェック用（不一致なら警告のみ、再生は可能）
  kf: [
    // 1KF = 「どこを・どの寄りで・どれだけ見せるか」
    { x: 0,    y: 0,    z: 1.0, dwell: 1.0, ease: "smooth" },
    { x: 0,    y: 1.25, z: 1.0, dwell: 1.0, ease: "smooth" },
    { x: 0.1,  y: 1.28, z: 1.6, dwell: 0.6, ease: "inout"  },  // z>1 = 寄り（ズーム）
    { x: 4.7,  y: 1.28, z: 1.0, dwell: 2.0, ease: "linear" },  // xが動けば横PAN
  ],
  wipes: [
    // ワイプはKF区間に紐づく（kfIndex i→i+1 の移動中に発生）
    { after: 3, seq: "wipeegg", swap: 0.385 }   // swap = ワイプ素材のフルカバー率位置(0-1)
  ]
};
```

- **x, y**: ワールド座標（画面単位。現行 `scroll-model-lp.html` と同じ系）
- **z**: ズーム倍率（1=等倍、2=2倍寄り）。**Z軸＝マルチプレーン**: 描画時に各レイヤー/アクターの `depth`（0=最奥〜1=最前、省略時0.5）に応じて `parallaxScale = 1 + (z-1) * depth` で拡大量を変えると、寄るほど層が開く（＝マルチプレーン撮影台）
- **dwell**: そのKFでの滞在の重み（相対値）。**travel**: KF間の移動の重みは `dist(kf[i],kf[i+1])` から自動計算し、`dwell:travel` 比でスクロール全長を配分する（ユーザーに尺の数値入力をさせない）
- **ease**: 区間イーズ。`SEG_EASE.length === kf.length - 1` の不変条件は camera-rig スキルと同じ
- 再生: `P(0..1) → 弧長パラメータ → cam{x,y,z}`。**スクロールの純関数**（逆走可逆）を維持する

### スクロール↔テイクのマッピング実装

```js
// build: kfからタイムテーブルを構築（P0で実装、以後共通）
// segs = [...{type:'dwell'|'travel', kfA, kfB, w}]  w=重み
// total = Σw。P*total を消化しながら該当セグメントを線形探索し、
// dwell中: cam=kf固定（±マウス微揺れ）/ travel中: ease(f)で補間
function buildTimetable(take){ /* 上記 */ }
function camAt(P){ /* → {x,y,z} */ }
```

## 3. P0 — TAKEデータ分離（ビューア化）

1. `scroll-model-lp.html` の `S0_ST` / `scene0..3` 内のカメラ計算を `camAt(P)` 呼び出しに置換
2. 現行の演出と**同じ動きになるTAKE JSON**（default take）をファイル内に定数として埋める（後方互換の検証を兼ねる）
3. `?take=<url>` パラメータ or `window.TAKE_OVERRIDE` でJSON差し替え可能に（file://ではfetch不可の場合があるため、**`<script id="take-b" type="application/json">` 埋め込み複数＋`?take=take-b`で選択**を正とする）
4. ワイプはTAKEの `wipes[]` から発火位置を再計算（現行のWIPES定数を廃止せず、default takeから生成する形に）

**受け入れ基準**: 差し替えテイクを1本同梱し（例: 逆回りで巡るtake-B）、`?take=take-b` で全編破綻なく再生・逆走できること。

## 4. P1 — ページ内TAKEエディタ（Eキー）

edit-mode-spec のUI構造をそのまま使う。**操作は最小2手**:

```
E → EDIT MODE（スクロール駆動を切断。ドラッグ=パン / ホイール=ズーム(z) / Shift+ドラッグ=微調整）
  構図が決まったら ▣ CAPTURE（Cキー）→ KFカードが右パネルに積まれる
Esc → 通常モードへ（ESCカスケードは KINETIC STAGE の優先順位に従う）
```

- KFカード: `▸ GO / ↻ UPDATE / ✕`＋区間 `EASE` セレクト＋ `DWELL` スライダー（camera-rigスキルのカードUIの2D版。配色 --ice）
- `▶ PREVIEW`: スクロールと独立にテイクを自動再生（P2のATTRACT MODEと同じ走行系を使う → SPEC_02参照）
- **COPY PARAMS / PASTE PARAMS** ボタンでTAKE JSONをクリップボード経由で入出力（シリーズ共通文法）
- ワイプ編集はP1では**やらない**（JSONを手で書く。UIを増やさない）
- undo: CAPTURE/UPDATE/削除の直前状態を1段だけ戻せれば十分（`Ctrl+Z`）

**受け入れ基準**: マウスだけで「3駅+寄り1つ」のテイクを60秒以内に作れて、COPY→PASTEで往復できること。

## 5. P2 — OBAN BUILDER（モーションコミック用・単体アプリ）

新規フォルダ `OBAN_BUILDER/oban-builder.html`（単一HTML）。**「画像を置く→カメラを打つ→書き出す」の3ステップ**しかない画面にする。

1. **PLACE**: 画像を複数ドラッグ&ドロップ → 大判キャンバスにパネルとして配置。パネルは object-rig の2D版操作（ドラッグ=移動、ハンドル=拡縮、ホイール=depth変更。配色 --rouge）。縦積みが基本の初期配置（コミックのページ順）
2. **TAKE**: P1と同一のTAKEエディタ（--ice）。コマからコマへ寄る・引く・流すをCAPTUREで打つ
3. **EXPORT**: 単一HTMLビューアを書き出し（`<a download>`）。画像は**同梱しない**（相対パス参照。フォルダごと配布する運用。=LP_Anime_CR/LP_Model_CRと同じ）。ビューアはP0のコードを内蔵したテンプレート文字列

- 保存: PLACE+TAKE全体を1つのJSON（PROJECT）としてCOPY/PASTE + `localStorage` 自動保存
- **やらないこと（P2スコープ外）**: 音、ページ内テキスト組版、アニメ連番の取り込み（静止画パネルのみ。連番は次版）、モバイル
- エクスポートしたビューアには ATTRACT MODE（SPEC_02）を標準搭載

**受け入れ基準**: 手持ちの画像10枚から、ブラウザだけで3分以内に「スクロールで読めるモーションコミック」1本が書き出せること。

## 6. デザイン・キー割当（KINETIC STAGE準拠）

- カメラ系UI=--ice / パネル配置系UI=--rouge / 背景PLUM系
- キー: `E`=EDIT, `C`=CAPTURE, `Space`=PREVIEW再生, `Esc`=カスケード（モーダル→PLAY→EDIT解除の順）
- 既存LPの `G`（X-RAY）と衝突しないこと

## 7. リスクと判断メモ

- **z（ズーム）とDOMテキスト**: DOMワールドレイヤーはtransform scaleで追従させるが、拡大時のボケ対策として `z>1.4` でフォントサイズ再計算（P0では transform のみで可）
- 弧長配分は「駅が近いと速く通過してしまう」ので、travel重みに下限（例: 0.25）を設ける
- P2のdrag&drop画像は `URL.createObjectURL` でプレビューし、EXPORT時に**ファイル名だけ**をJSONに書く（同梱しない方針のため、ユーザーがフォルダ構成を変えると壊れる——README1行をエクスポートHTML先頭コメントに含める）
