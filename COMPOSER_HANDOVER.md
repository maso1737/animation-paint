COMPOSER v0.5（単一HTML）の続き開発です。

【現在の実装済み仕様】（v0.5 時点）

■ マルチトラック
- state.tracks[] 配列でトラック管理
- tracks[0] = BG（最背面）、tracks[N-1] = 最前面（描画順）
- UI表示はAE/PS準拠：上が前面レイヤー、下がBG
- コンポジット描画：drawFrame() が tracks[0]→[N-1] を順に重ね描き
- 各トラックが独立した keyframes / markers / visible / solo / cellInfos を保持

■ トラック操作
- IMPORT JSON ボタン：全トラック置き換えで読み込み
- + TRACK ボタン：既存トラックを保持したまま新トラックを最前面に追加
- 各トラックに ◉（表示/非表示）、S（SOLO）、ANM（→ANIMATOR送信）、PNG（単体書き出し）、✕（削除）ボタン
- タイムラインのトラック行クリックで選択（KF編集対象が切り替わる）
- 選択中トラックをハイライト表示（--acid 色）
- ⠿ドラッグハンドルでトラック並び替え（D&D、ドロップインジケーター付き）
- トラック名ダブルクリックでインライン編集（Enter確定/Escキャンセル）

■ SOLO
- Sボタンで対象トラックをソロ（それ以外を非表示）
- 複数トラックを同時ソロ可能
- ソロOFFのトラックはUIで薄暗く表示（.soloed-off）
- drawOneTrack() がhasSoloフラグで判断

■ 動的トラック高さ
- updateTrackHeights() がトラック数・オーディオ行の有無に応じて高さを自動調整
- CSS変数 --track-h（32px〜54px）で制御
- rebuildAllTrackUI() / loadJSON() / resize 時に再計算

■ KFシステム（全関数にkfsパラメータ）
- getKfValue(prop, frame, kfs) / getTransform(frame, kfs)
- hasKfAt(prop, frame, kfs) / setKf(prop, frame, value, kfs)
- removeKf(prop, frame, kfs)
- selTrack() → 選択中トラックオブジェクト
- kfsNow() → 選択中トラックのkeyframesを返す

■ プロパティ（全7種）
- X/Y（位置）、SCL（スケール）、AX/AY（アンカー）、ROT（回転）、OP（不透明度）
- 線形補間

■ 数値スクラブ（AE-style）
- FX HUD・インスペクターのプロパティラベルをドラッグで値を変更
- Shift = ×10、Ctrl = ×0.1
- KFが現フレームに存在する場合のみ値を更新（既存change evtと同挙動）
- カーソルは ew-resize

■ キーボードショートカット（AE準拠）
- Space: 再生/停止
- ←/→: 1フレーム移動
- Home/End: 先頭/末尾
- B/N: ワークエリア イン/アウト設定
- M: 選択中トラックにマーカー追加/削除
- P: X/Y表示トグル
- S: Scale表示トグル
- A: Anchor表示トグル
- R: Rotation表示トグル
- T: Opacity表示トグル
- U: KFがあるプロパティのみ表示
- X: FX HUDトグル
- Ctrl+Z / Ctrl+Shift+Z: Undo/Redo
- ESC: FX HUD閉じる

■ ワークエリア
- B/Nキーでイン/アウト設定（AE準拠）
- タイムラインにハンドル表示・ドラッグ操作
- 再生はワークエリア内でループ

■ UNDO/REDO
- 変更後記録方式（recordHistory()）
- 全トラックのkeyframes+markersをスナップショット
- 上限50ステップ

■ FX HUD（浮動パネル）
- ヘッダドラッグで移動
- 全プロパティ表示・編集・KFドット
- P/S/A/R/T/Uショートカットと連動

■ ビューポート
- 位置ハンドル（□）ドラッグ：選択中トラックにX/Yオートキー
- アンカーハンドル（×）ドラッグ：選択中トラックにAX/AYオートキー

■ オーディオ（BGM/SE）
- ♪ AUDIOボタンまたはドラッグ&ドロップ（audio/*）で読み込み
- AudioContext で再生・停止
- アニメーション再生と同期（play/pause/loop 時に自動連動）
- タイムライン下部に波形表示（キャンバス描画）
- 波形エリアを左右ドラッグでオフセット調整（開始フレームずらし）
- ミュートボタン（◉）・削除ボタン（✕）
- state外の const audio オブジェクトで管理

■ PNG書き出し
- EXPORT 4K PNG：全可視トラックをコンポジットして連番PNG→ZIP
- 各トラックPNGボタン：そのトラック単体を4K書き出し（KF変形込み）
- ANMボタン（各トラック）→ IndexedDB経由でANIMATORに送信

■ インポート
- PROJECT_v1（単一トラック）
- PROJECT_v2（複数トラック）
- ANIMATOR_v1（互換読込）
- ドラッグ&ドロップ対応（JSONファイル・オーディオファイル両対応）

【未実装（将来フェーズ）】
- FRAME参照画像（ANIMATORでの複数ANIMATOR参照）

【デザイントーン】
tDRスタイル（--acid:#36FF00、--neon:#F9FF47、モノスペース、黒基調）

【コードの注意点】
- KF関数はすべてkfsパラメータを明示（グローバル参照なし）
- 「変更後に記録」方式：recordHistory() は操作完了後に呼ぶ
- op=0 の扱い：parseFloat()||1 ではなく isNaN(v)?1:v で判定
- KFダイヤモンドのドラッグ中はrenderKfDiamonds()を呼ばない
- data-track属性でDOM要素→state.tracks[idx]をマッピング
- rebuildAllTrackUI() でトラック全再構築（オーディオ行 #tl-audio-row を保存して再追加）
- renderKfDiamonds(trackIdx) / renderMarkers(trackIdx) はDOMに要素が追加された後に呼ぶ
- const audio = {...} はstate外のグローバルオブジェクト
- drawOneTrack(ctx,w,h,frame,track,hasSolo) の第6引数hasSoloを必ず渡す
- UI SOLOボタンのインデックス：el.querySelectorAll('.tl-tbtn')[1] = soloBtn

今のcomposer.htmlを渡すので、ゼロから作らず
このファイルに追記する形で進めてください。
