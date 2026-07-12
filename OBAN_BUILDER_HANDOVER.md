# CLAUDE.md — OBAN_BUILDER

**OBAN BUILDER**（SPEC_01 P2、2026-07実装）。モーションコミック製造機。画像を置く（PLACE）→カメラを打つ（TAKE）→単一HTMLビューアを書き出す（EXPORT）の3ステップだけの単体アプリ。単一HTML・依存ゼロ・`file://` 直開きOK。

## ファイル

- `oban-builder.html` — 本体（エディタ）
- 書き出される `oban-viewer.html` — 自己完結ビューア（画像は同梱せず**同フォルダのファイル名参照**。画像フォルダに置いて配布する運用）

## 操作（触ってすぐわかる、が最優先）

| 操作 | 内容 |
|---|---|
| 画像をD&D | パネル追加（縦積みで初期配置）。複数OK |
| **1. PLACE**（--rouge） | パネルドラッグ=移動 / 空白ドラッグ=視点パン / ホイール=視点ズーム / 選択チップで SIZE±・DEPTHスライダー・DELETE |
| **2. TAKE**（--ice） | 構図を作って `C`=CAPTURE。KFカードで ▸GO / ↻UPD / ✕ / EASE / DWELL。`Space`=ループPREVIEW（触ると停止） |
| **3. EXPORT** | `oban-viewer.html` をダウンロード。ATTRACT MODE標準搭載（`?attract=1&hud=0`） |
| COPY/PASTE PROJ | プロジェクトJSONのクリップボード入出力（file://で拒否されたらprompt窓にフォールバック） |
| `Ctrl+Z` | 1段undo / `Esc` プレビュー停止・選択解除 / `1`/`2` モード切替 |

## V2-A 品質の土台（2026-07 実装済み / SPEC_05）

- **z-order分離**: `ord`（重なり）とdepth（視差）を独立管理。`[`/`]`キーとチップの◀BACK/FRONT▶で前後移動（ord正規化つき）。新規投入は常に最前
- **undo/redoスタック**: `commit()`=保存+履歴push（上限60）。スライダー系は`commitD()`で300ms合体。`Ctrl+Z`/`Ctrl+Shift+Z`/`Ctrl+Y`。CLEARもundo可能
- **UIホバー時ホイール**: `#bar/#cards/#chip`上ではビューをズームせずUIスクロールに委譲
- **ガイド（G / GUIDEボタン）**: 16:9枠+ACTION93%+TITLE90%+中央十字+三分割。状態はlocalStorage記憶。**書き出しには含まれない**
- **KFフォーカス**: カードクリック=選択（--iceハイライト、カメラは動かない）、`▸GO`or`↑↓`=選択+ジャンプ。キャンバス上は二重リング+dwell帯で「いま触っているKF」を常時表示
- プロジェクトは `version:2`（`migrate()`がV1を自動補完）。ビューアもordソート対応

## V2-B シーケンスとコマ（2026-07 実装済み / SPEC_05）

- **連番シーケンス**: ドロップ時に `prefix+数字3〜5桁+拡張子` でグループ化、**同prefix4枚以上→自動で1つのseqパネル**（3枚以下はバラ）。チップで MODE（loop=目パチ・なびき・粉塵 / once=ワンアクション / pingpong）・FPS 4-30・TRIGGER（always / **enter**=画面30%進入で再生開始、画面外でリセット→再入場で再演）
- **FRAME＝1段ネストコンポ**: `+FRAMEボタン` で追加。パネルを**フレームに重ねて離すと格納**（チップのEJECTで復帰）。**ダブルクリックで中に入って編集**（パンくず `ROOT ▸ FRAME 01`、他要素は減光、Escで復帰）。ネストは1段のみ（フレームinフレーム禁止）
- **quadマスク**: フレーム選択→`M`（またはチップMASK）で4隅ハンドル編集（斜めOK、Shift=軸ロック、範囲-0.5〜1.5）。多角形・円は非対応（仕様どおり）
- **内部パララックス**: `PAR` スライダー（0..1）。カメラ移動に対しコマの窓の中で子パネルが `k=lerp(0.15,0.85,1-depth)` でずれる＝奥ほど残る
- ヒット判定/並べ替え/削除はコンテキスト対応（フレーム内では兄弟内でord正規化、フレーム削除時は子をルートへ放出）
- ビューア完全同期: seq全フレームロード・quadクリップ・内部パララックスを書き出しに含む

## V2-C 演出と撮影（2026-07 実装済み / SPEC_05 §4 + SPEC_06 P2先行）

### FRAMEのIN演出・ドリフト・カバーワイプ（チップ2行目）
- `IN`（slide-l/r/t/b）: 紐づくKFの**直前travel区間**でコマが枠ごとスライドイン。`DRIFT`（push-in/pull-out）: **dwell区間**でゆっくり寄る/引く（AMT 0〜0.4）。`KF`は自動（最寄り）or明示。すべてPの純関数＝逆走可逆
- `WIPE`（invert/whiteout）: コマが**画面を100%覆った最小P**を400サンプルで事前計算（`computeWipeP()`、リサイズ/テイク変更で再計算）。invert=ステージ全体が明暗反転（TENSION CH05→06の型）、whiteout=覆った瞬間白→次の絵。ガイドON時はKFパス上に琥珀◇で発火点表示
- ビルダーでは**PREVIEW（Space）中のみ**発動。編集中はバッジ表示のみ

### SATSUEI FX（撮影処理 — satsuei-fx-kit準拠・SPEC_06 P2をcomposer P0より先行実装）
- 正準コアを `<script id="satsuei-core">` にマーカー付きで複製（**現時点の生きた正準**。skillのreferencesと同一内容。更新はブロックごと差し替え）
- `PROJECT.take.fx`（fxスキーマv1・全アプリ共通・COPY/PASTE PROJに自然に乗る）。`migrate()`が欠損補完
- **FXボタン/[F]キー**→琥珀色モーダル。UIは `FX_DEFS` から自動生成（**エフェクト追加でUI改修ゼロ**）: diffusion/para/grade/vignette/grain/glitch/mb/dof。変更はcommitD（undo対象）
- 適用tier: ビルダーPREVIEW=**draft**（velなし=mb素通し）/ 書き出しビューア=**rt**（カメラ速度velで方向ブラーmb近似）。編集中はFXなし（軽量優先）
- **絶対条件の遵守**: `enabled:false`はapply未呼び出し経路・WebGL不可は素通し（`gfx.available`）・ビューア`?fx=0`で強制OFF・コアにバッククォート/`</script>`文字列なし（検証スクリプトがチェック）
- ビューアはシーンを常時オフスクリーン`sc`に描き→FX→可視`cv`へ（fx OFF時も同経路のblitのみ＝ピクセル一致）

## データモデル

```js
PROJECT={version:2,name,
  panels:[{id,name,x,y,h,depth,ord,ar,parent:null|frameId,
    seq:null|{pre,pad,start,n,ext,fps,mode,trigger}}],   // 画像実体は持たない
  frames:[{id,name,x,y,h,depth,ord,ar,quad:[4×{x,y}],par}],  // 子はフレームローカル座標(中心原点・フレーム高=1)
  take:{version:1,name,kf:[{x,y,z,dwell,ease}]}}       // SPEC_01のTAKE(単一シーン版)
```

- タイムテーブルは `buildTake()`（dwell/travel重み自動配分、travel下限0.25）— `LP_Model_CR` のP0実装と同型（ワイプ無し版）
- **マルチプレーン**: パン係数 `lerp(0.7,1.2,depth)`、ズーム係数 `1+(z-1)*lerp(0.55,1.25,depth)`。ビューアも同式
- `localStorage('oban-project')` に自動保存。**画像はファイル名だけ**保存されるので、再起動後は同じ画像を再ドロップすると復元（名前一致で紐づけ）

## 検証

Node構文チェック＋DOMスタブスモーク済み。ビューアはテンプレートから実生成→構文チェック→スクロール往復＋ATTRACT自動走行スモーク済み。**ビューアテンプレート内に生バッククォートを置かない**（検証スクリプトがチェックしている。`</script>` は `S` 変数経由）。

## 制限（v1スコープ / SPEC_01 §5どおり）

- 静止画パネルのみ（連番アニメ取り込みは次版）。音・テキスト組版・モバイルなし
- CLEARはnative confirm（modalConfirm化はTODO）
- ビューアはビルダーのコード複製（共有ライブラリ化しない方針どおり）。テンプレートを変えたら両方のスモークを回すこと
