# SpringBoard: システムPicture-in-Picture(PGPictureInPicture* / SBPIP*)の内部構造

(2026-10-10, SpringBoard PID例60707, iOS 16.7.16。システムPiPウィンドウの自由なリサイズ/移動を解放するtweak「PiPFreedom」のデバッグ中、ピンチリサイズが見た目上ウィンドウを角に引っ張る挙動を調査。スクリプトは`~/Frida/springboard_recon/dump_pip_methods.js`等。)

## ⚠️ 最大の罠: 公開class-dumpや既存tweakは、全く別の(古い)リビジョンを前提にしていることがある

**iOS-Runtime-Headers等の公開class-dump集や、既存の公開tweak(FreePIP, BigPiPEnergy)は、同じクラス名でも全く違うメソッドセットを持つ、別バージョンのプライベートフレームワークを前提にしていることがある。** iOS 16.7.16のこの端末では、`SBPIPContainerViewController`/`SBPIPContentViewLayoutSettings`は、その2つのtweakがフックしている`_constrainContentViewSize:`等のメソッドを**一つも持っていない**(ワイルドカード無しの、存在確認済みクラスに対する完全なメソッドダンプで確認)。

Logosで存在しないセレクタを`%hook`すると**クラッシュも警告も出ない無音の失敗**になる——これが原因で、「サイズ無制限化の仕組みが効いている」と丸々1セッション思い込んでいたが、実際には一度も発火しておらず、観察していた挙動(画面近くまでの自由なリサイズ)は100%無改造の標準iOS PiP挙動だった。

**教訓**: システムUIクラスに対して公開tweakのフック一覧を信頼する前に、**必ず実機でそのクラスの現在のメソッド一覧をダンプする**(`dump_pip_methods.js`が再利用可能なテンプレート——既知のクラス名を`objc_getClass`で確認してから、`class_copyMethodList`でインスタンス/クラスメソッド双方を無差別ダンプ)。

## 確認できた現行(iOS 16.7.16)クラス構成

- **`SBPIPContainerViewController`**(125メソッド) — `contentViewController`(iOS14/15時代の`pictureInPictureViewController`との分裂は解消済み)、`forcePictureInPictureToFrame:`(ウィンドウ枠を直接設定する実在のミューテーター)、`interactionControllerDidBeginSizeChange:behavior:`/`...DidEndSizeChange:`(サイズ変更ジェスチャーの開始/終了デリゲート)、`_updateContentViewFrame:reason:usingDisplayArrangementSpace:`(現行の実際のフレーム更新エントリポイント)
- **`SBPIPInteractionController`**(138メソッド、リサイズ/位置ロジックのほとんどがここに移動している) — `canonicalPosition`(`Q`型、ウィンドウが属する「角/領域」を表す保存値、観測値は`0`/`3`)、`_roundPosition:forSize:`(スナップ計算と思われる、未呼び出し確認)、`_sizeChangeBeganWithBehavior:`(**名前に反してピンチの`Changed`ティック毎に発火**、1回のジェスチャーで約190回)、`_sizeChangeEnded`(正しく1回だけ発火)
- **`SBPIPContentViewLayoutSettings`**(22メソッド) — サイズ上限/下限クエリが4種類に分かれている(`maximumPossible`/`maximumPreferred`/`minimumPossible`/`minimumPreferred`)
- **`PGPictureInPictureViewController`**(108メソッド) — `setStashed:`/`stashed`/`stashProgress`(画面端ドッキング、今回のバグとは無関係)、`updateHostedWindowSize:`(**呼んでも`view.bounds`に一切効果なし、確認済み——リサイズ用途に使わない**)

## 読み取り専用観察で判明した「角に引っ張られる」バグの正体

実際にPiPウィンドウをドラッグ→ピンチリサイズを3回行う間、上記メソッド群を読み取り専用で観察:

```
canonicalPosition が序盤に~10回クエリされ、値は0→(ドラッグ後)3に変化
  -- その後ジェスチャー中は一切クエリされない
_sizeChangeBeganWithBehavior(behavior=1) が pinchのChangedティック毎に発火(~190回/ジェスチャー)
canonicalPosition がもう1回だけクエリされ(値は3のまま)、直後に:
_sizeChangeEnded が1回だけ発火
```

**結論: `canonicalPosition`は「このウィンドウがどの角に属するか」を表すキャッシュ値で、リサイズジェスチャーの"終了時"に一度だけ読み取られ、リサイズ後の再アンカー位置を決めるのに使われる**——ピンチ中に継続的に再計算されているわけではない。これが、一度でもドラッグしたウィンドウをピンチリサイズすると、開始位置に関係なく必ずその保存された角方向にさらに引っ張られる理由。ユーザー本人が、tweakのカスタマイズを無効化した状態でも同じ挙動を確認し、**これは無改造の標準iOS PiP挙動である**ことが裏付けられた。

## ⚠️ `forcePictureInPictureToFrame:`はデリゲートコールバック内から同期呼び出しするとabortする

`-interactionControllerDidEndSizeChange:`(システム自身のアンカースナップ処理が完了した直後に発火するデリゲート)の中から、この実在する「ウィンドウ枠を直接設定する」APIを同期的に呼んでみたところ、`Error: abort was called`(Frida側で例外捕捉、SpringBoard自体はクラッシュせず)。ソフトな前提条件/再入アサーション失敗と思われる。「システム自身の処理がちょうど終わった」ことを示すデリゲートコールバックの中から**まさにそのタイミングで**同期的に呼ぶのが再入ガードに引っかかっている可能性が高いが、`dispatch_async`で次のランループティックに遅延させる案は未検証——1回でも実際の(非致命的でも)abortが観測された時点で、このスキルの既存方針(§4の「ここで止める」前例)に従い深追いを中断した。

## 未着手の手がかり

- `setInteractivelyResizing:`(名前からして「今まさにライブリサイズ中か」のフラグ、未検証)
- `_roundPosition:forSize:`/`_canonicalPositionForPoint:`(角スナップの実アルゴリズムの最有力候補、存在確認のみで未観察。CGPoint/CGSize構造体引数はFridaの通常の`args[]`では取れず、`this.context.d0`/`d1`等の生のFPUレジスタから読む必要がある)
- `forcePictureInPictureToFrame:`の`dispatch_async`遅延呼び出し(未検証)
