COMPOSER v0.7（単一HTML）の続き開発用ハンドオフ。

【v0.7 MOTION COMIC — 詳細は MOTION_COMIC_SPEC.md】
- track.type: 'anim'|'image'|'camera'、track.tid（恒久ID）、track.parent（親のtid）
- 画像インポート: PNG/JPEG/WebP→cells1枚の擬似JSON（IMAGE_v1）→importJSON。importFiles()でJSON/画像/audio混在D&D
- CAMERA: 普通のトラック（frames空）として実装＝KF系UIを全部流用。1つまで(dedupeCameras)。projectId=null。◉=カメラON/OFF
  X/Y=パン(レイヤーZでパララックス) Z=ドリー ROT/SCL=全体ラップ(applyCamWrap)。カメラなし時は従来式と完全一致（後方互換）
- 親子: applyTrackChain再帰でctx変換合成。子のZ無効/OP非継承/循環はUI除外+深度8上限。親付き子はハンドル非表示
- バグ修正: PROJECT_v2再IMPORTでKF全損（トラック要素にformatが無く復元条件が偽）→ composerセクション有無だけで判定に変更。per-track width/heightも保存するよう修正

【概要】
ANIMATORの作画コマを複数トラックで重ね、トランスフォーム/キーフレームでカメラワーク・タイミングを付け、4K連番PNGで書き出す合成ツール。tDRスタイル（--acid:#36FF00 / --neon:#F9FF47 / モノスペース / 黒基調）。

━━━━━━━━━━━━━━━━━━━━━━━━━━
■ マルチトラック
- state.tracks[] = [{projectId,name,width,height,cellsRaw,cellInfos,frames,totalFrames,keyframes,markers,visible,solo}]
- tracks[0]=最背面(BG)、tracks[N-1]=最前面。drawFrame()が0→N-1で重ね描き
- UI表示はAE/PS準拠：上=前面、下=BG（rebuildAllTrackUIは i=N-1→0 で生成）
- 各トラックボタン：◉表示/非表示, S(SOLO), PNG(単体4K書出), ✕削除
  ※ ANM(ANIMATOR送信)ボタンは廃止（LIVE連携と二重で混乱するため）。sendTrackToAnimatorは未使用で残置
- ⠿ドラッグで並び替え、名前ダブルクリックでインライン編集
- width/height はトラック固有（解像度違いの基点ずれ対策。下記レンダリング参照）

■ インポート（IMPORT JSON に一本化、+TRACK廃止）
- importJSON(json)：空なら新規読込、既存があれば「追加」。複数回で2つ3つと重ねられる
- PROJECT_v2=loadJSON（全置換 or 追加）、PROJECT_v1/ANIMATOR_v1=単一トラック追加
- finishImport()が共通の後処理（totalFrames/workArea/UI再構築/announceLive）
- SPEC_06 P3受け（OBANの COPY FOR COMPOSER 用）: ①カメラだけのPROJECT_v2でもKF最終フレームから totalFrames を確保（parseTrackFromJSON）②追加IMPORTでも `fx:` があれば normalizeFx で state.fx を引き継ぐ。既存CAMERAがある状態で貼るとカメラは捨てられる（dedupeCameras=既存優先）
- ドラッグ&ドロップ対応（JSON / audio両対応）

■ レンダリング（drawOneTrack）
- sx=出力W/state.width, sy=出力H/state.height（コンポ→出力）
- トラック固有 trW/trH を使い drawW=trW*sx, drawH=trH*sy で「コンポ中央に原寸配置」（AE準拠）。解像度違いでも基点がずれない
- Z軸疑似3D：persp=PERSP_FOCAL(1000)/max(1,1000+Z)、eff=scale*persp で乗算（scaleと独立）。Z>0=奥、Z<0=手前。TU演出はZを0→負へ
- 透明背景なし＝bgBrightのグレーで塗りつぶし後に重ね描き

■ トランスフォーム / プロパティ（ALL_PROPS）
- 順序：x,y,z,s,ax,ay,rot,op（UI表示順=AX/AY→X/Y/Z→ROT→SCL→OP）
- PROP_STEP / PROP_MIN / PROP_MAX / propDefOf（s,op既定=1, 他=0）/ fmtProp / clampProp
- commitProp(prop,value,record)：現フレームにキーが無ければ作成＝「変化した瞬間に即キー」
- インスペクター(#kf-*)とFX HUD(#fx-*)は同じprop群。インスペクターのトランスフォームは常時表示、FX HUDのみP/S/A/R/T/Uの表示トグルに連動（applyPropVisibleは#fx-hudのみ対象）
- ◀▶ステッパ(.num-step, data-step/data-fxstep + data-dir)、Shift=×10
- 数値スクラブ：ラベル(.k)と入力欄自体をドラッグで変更。入力欄はクリック=編集 / ドラッグ=スクラブ（ドラッグ開始までcapture/preventDefaultしない）。Shift=×10, Ctrl=×0.1
- updateKfUI/updateFxHud は ALL_PROPS をループ（document.activeElement!==el のとき値上書き＝編集中は保持）

■ キーフレーム（線形補間＋イーズ）
- 構造：track.keyframes[prop] = [{f,v,ez}]（ezは0=リニア / 1=最大。smoothstepへ ez 量ブレンド）
- getKfValue：区間内 t を ez で smoothstep へブレンド。ez=Math.max(両端)
- 即キー：数値変更/スクラブ/ステッパ/ハンドルドラッグで現フレームにキー生成
- KFダイヤ（renderKfDiamonds）：data-track/data-f を持つ。形でイーズ表示
  - リニア=鋭い菱形(ネオン) / 最大=円(マゼンタ, .ez-max) / 中=角丸(水色, .ez-mid※インスペクターEASEボタンの0.5用)

■ KF選択モデル
- kfSel = [{t,f}]（トラックまたぎ）。kfSelHas/Set/Toggle/Clear, refreshKfSelClasses（.sel付与）
- ダイヤ：クリック=単一選択+seek / Shift+クリック=トグル / Ctrl(Cmd)+クリック・ダブルタップ=イーズ切替(cycleEase 0↔1)
- ドラッグ=移動（単一）。複数選択キーを掴むとグループ移動（groupSnap+2パスで衝突回避, 端でクランプ）。Shift+ドラッグ=複製。ドラッグ直後も選択維持→Del可
- マーキー選択：#tl-tracks上でドラッグ（サムネはpointer-events:none/draggable=false）。横=ダイヤ中心X / 縦=トラック行全体と交差で判定。Shiftで追加
- コピペ：copyKeyframes(選択優先, 未選択時はトラック全体, 最小フレーム基準の相対items) / pasteKeyframes(再生ヘッド基準で additive・同一/別トラック可)。Ctrl+C/V でも可
- 削除：Del/BackSpace = 選択分 or 再生ヘッド位置（deleteSelectedOrCurrentKf）
- イーズ一括：インスペクターのEASEボタン（クリック=中0.5 / Shift=最大1.0）。選択優先

■ マーカー（グローバル/ワークエリア）
- state.markers = [{frame,label}]（旧：トラック単位。現在はM=ルーラー上のグローバルマーカー）
- renderGlobalMarkers()：#tl-ruler-markers に配置。ドラッグ移動 / クリックseek / ダブルクリックでメモ(prompt) / Ctrl(Cmd)+クリックで削除(AE準拠)
- PROJECT_v2 top-level markers として保存/復元、undoスナップに gmarkers
- 旧 per-track markers(renderMarkers) は読込互換で残置

■ ビューポート（pan/zoom = ANIMATOR準拠）
- state.view={zoom,panX,panY,baseW,baseH}。applyView()が #viewport-inner に transform
- ホイール転がし=ズーム(カーソル基点 zoomViewportAt) / 中ボタンドラッグ=PAN / ダブルクリック=resetView(FIT)
- touch-action:none（液タブのペン途切れ対策）。ドラッグ移動量は screenToCanvasScale() でズーム補正
- 位置ハンドル(□)=X/Yオートキー、アンカーハンドル(×)=AX/AYオートキー（button!==0は無視＝中ボタンと競合しない）
- setupViewport末尾で必ず drawCurrentFrame()（canvas.width再設定でクリアされる→リサイズ直後の黒画面対策）

■ ガイド（解像度枠 / セーフフレーム）
- #guide-canvas（viewport-inner内, pointer-events:none）。renderGuides()が state.guides で描画。表示のみ・書き出し非合成
- 設定パネル(⚙)で ON/OFF・サイズ・セーフ%。lwはズーム補正

■ 設定パネル（フローティング, ANIMATOR準拠）
- ⚙ #btn-settings でトグル。ヘッダドラッグ移動、下端 #settings-resize でリサイズ（.set-sectの境目スナップ）
- CANVAS GUIDES + KEYBOARD SHORTCUTS（再割当UI）。※解像度変更はcomposerでは行わない

■ ショートカット（登録制＋再割当, localStorage:composer_keymap_v1）
- SHORTCUT_ACTIONS[].{id,label,def,prevent,run(e)}。gKeymapでキー→action。設定パネルで変更
- 既定：Space再生 / ←→コマ(Shift=10) / Home/End / B,N ワークエリア / M マーカー / i キー追加 / J,K 前後キーへ / Del キー削除 / F FIT / P/S/A/R/T/U HUD表示 / X FX HUD / , 設定
- Ctrl+Z/Shift+Z=undo/redo、Ctrl+C/V=KFコピペ、BackSpace=削除、Escape=メニュー/選択/HUD/設定を閉じる

■ タイムコード
- frameToTimecode(idx)=AE準拠 H:MM:SS:FF（1始まり, 末尾=コマ）。先頭=0:00:00:01
- 右下TIME=タイムコード。コーナーの「FRM/」クリックで コマ⇔タイムコード切替（state.timeMode）

■ タイムライン
- グリッド行 var(--tl-h) を #tl-resize の上端ドラッグで高さ変更
- トラック区切り線強調、ストリップ高はトラック高に追従、サムネ object-fit:cover（歪み防止）
- KFダイヤ拡大+黒フチ+ホバー点滅。最小トラック高46px（下回ればスクロール）
- ◀▶コマ送りボタンは廃止（ショートカットのみ）

■ ワークエリア / 再生 / オーディオ / 書き出し / Undo
- ワークエリア：B/N、ルーラーのハンドルドラッグ、再生ループ
- 4K PNG：EXPORT 4K PNG（全可視合成）/ 各トラックPNG。書き出しは drawFrame/drawOneTrack を OUT_W/OUT_H で
- AUDIO：♪ボタン/ドラッグ、波形、オフセット、ミュート（state外の const audio）
- Undo：cloneEditState（tracks.keyframes+markers, gmarkers, selectedTrack）, recordHistory（変更後記録）, 上限50

■ ライブ連携（BroadcastChannel 'tdr_live'）
- ANIMATOR保存(autosave)→project-update。COMPOSERは projectId一致トラックの「絵だけ」差し替え（KF/transform/表示状態は保持, liveUpdateTrackがwidth/heightも追従）
- announceLive()=composer-hello+requestSyncAll。setupLiveSync（起動+700ms遅延再通知+window focus時）と、loadJSON/finishImport完了時に呼ぶ＝LIVEボタンを押さなくても自動反映
- ANIMATOR側は gLiveActive が立つと autosave毎に broadcast。composer-hello / request-sync / animator-hello で立つ

━━━━━━━━━━━━━━━━━━━━━━━━━━
【コードの注意点】
- KF関数はkfsパラメータ明示。op=0は isNaN(v)?1:v で判定
- ALL_PROPS にprop追加時は #kf-*/#fx-*/#dot-*/#fx-dot-* のUIも要追加（updateKfUI/Fxがループ参照）
- KFダイヤのドラッグ中はrenderKfDiamondsを呼ばない（pointer capture喪失）。pointerup後に再描画
- data-track属性でDOM→state.tracks[idx]対応
- rebuildAllTrackUI()で全再構築（#tl-audio-row退避→再追加）。renderKfDiamonds/renderMarkers/renderGlobalMarkersはDOM追加後
- drawOneTrack(ctx,w,h,frame,track,hasSolo) 第6引数hasSolo必須
- SOLOボタンは el.querySelectorAll('.tl-tbtn')[1]
- 確認ダイアログ(confirm)は全廃方針

【未実装 / 将来候補】
- FRAME参照画像（複数ANIMATOR参照）
- イーズの数値カーブ編集 / グラフエディタ
- モーションブラー、調整レイヤー的なエフェクト

【変更後チェック】
node -e "const fs=require('fs');const h=fs.readFileSync('composer.html','utf8');const m=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(x=>x[1]).filter(s=>s.length>200).join('\n;\n');new Function(m);console.log('OK')"
