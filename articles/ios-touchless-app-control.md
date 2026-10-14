# ホーム画面到達後、任意のアプリをタッチレスで操作する(アプリ起動 + ボタン動作の直接呼び出し)

[ios-touchless-wake-unlock.md](ios-touchless-wake-unlock.md)でSSH+Fridaだけでホーム画面まで到達できるようになった後の話。「アプリを起動して、何か操作して、結果を得る」をタッチ座標の勘に頼らずに行う方法。具体例として「Mail.appを起動し、宛先・件名・本文を入れて実際に送信する」を最後まで通した。

## 結論だけ先に

1. **アプリ起動**: `-[LSApplicationWorkspace defaultWorkspace]`の`-openURL:`(`mailto:`等のURLスキーム)または`-openApplicationWithBundleID:`でアプリを起動・前面化できる。SpringBoardプロセスに繋いだままで良い(アプリ自体の別プロセスにattachし直す必要はない)。
2. **画面上のボタンをタップで操作するのは当てにならない**: スクリーンショットから座標を正確に割り出しても(今回はピクセル解析で中心座標を特定)、合成タッチイベントがボタンを反応させないことがある(今回のMail送信ボタンがまさにこれ — 理由は未解明、おそらくiOS16のコンテキストメニュー付きボタン(`menuOnLongPress`)特有の認識ロジックのせい)。
3. **確実なのは「ボタンの実体(target/action)を特定して直接呼ぶ」こと**: 対象アプリのプロセスにFridaをattachし、`UIApplication.sharedApplication.windows` → `rootViewController.presentedViewController` → ... とウィンドウ階層を辿って、押したいボタンに対応する`UIBarButtonItem`/`UIButton`の`target`と`action`(セレクタ)を読み取る。
4. **その呼び出しは普通に`objc_msgSend`で直接呼ぶとメインスレッドアサートで`abort`する**(クラッシュはしない、Fridaが安全に止める)。かといって`dispatch_async_f` + `NativeCallback`は[過去に判明した通り破損/危険](ios-touchless-wake-unlock.md)。**`-[NSObject performSelectorOnMainThread:withObject:waitUntilDone:]`を使うと、GCDの生コールバックを経由せずObjective-Cランタイム自身のスレッド受け渡し機構でメインスレッドに乗るので安全に動く**。

この「target/actionを特定 → `performSelectorOnMainThread:withObject:waitUntilDone:`で呼ぶ」は、Mail送信に限らず**任意のアプリの任意のボタン動作**に応用できる汎用パターン。

## 手順

### 1. アプリをURLスキーム付きで起動する(SpringBoardプロセスから)

```javascript
var objc_getClass = new NativeFunction(Module.findGlobalExportByName("objc_getClass"), "pointer", ["pointer"]);
var sel_registerName = new NativeFunction(Module.findGlobalExportByName("sel_registerName"), "pointer", ["pointer"]);
var objc_msgSend_ptr = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "pointer", ["pointer", "pointer"]);
var objc_msgSend_ptr_ptr = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "pointer", ["pointer", "pointer", "pointer"]);
var objc_msgSend_bool_ptr = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "bool", ["pointer", "pointer", "pointer"]);
function sel(name) { return sel_registerName(Memory.allocUtf8String(name)); }

var mailto = "mailto:someone@example.com?subject=" + encodeURIComponent("件名") + "&body=" + encodeURIComponent("本文");

var nsStringCls = objc_getClass(Memory.allocUtf8String("NSString"));
var urlStr = objc_msgSend_ptr_ptr(nsStringCls, sel("stringWithUTF8String:"), Memory.allocUtf8String(mailto));
var nsURLCls = objc_getClass(Memory.allocUtf8String("NSURL"));
var url = objc_msgSend_ptr_ptr(nsURLCls, sel("URLWithString:"), urlStr);

var workspaceCls = objc_getClass(Memory.allocUtf8String("LSApplicationWorkspace"));
var workspace = objc_msgSend_ptr(workspaceCls, sel("defaultWorkspace"));
var result = objc_msgSend_bool_ptr(workspace, sel("openURL:"), url); // result=1 で成功
```

`LSApplicationWorkspace`はSpringBoardに限らずほぼ全プロセスから使える(今回はSpringBoardにattachしたまま呼んだ)。`openApplicationWithBundleID:`でバンドルIDを直接指定して起動することも可能(URLスキームを使わない単純な起動)。

### 2. (つまずいた点)座標タップでSendボタンを押そうとして失敗

スクリーンショット(`_UICreateScreenUIImage`)をPythonのPILでピクセル解析し、青いSendボタンの中心座標を正確に特定(`(1117, 362)` / 正規化`(0.899, 0.164)`)。[swipe_up.js](ios-touchless-wake-unlock.md)と同じIOHIDデジタイザタッチ合成でその座標をタップしたが、**何も起きなかった**(キーボードも閉じない、ボタンのハイライトもない = タッチがどこにも当たっていない)。1セッション内でタップ前後のスクリーンショットを連続取得して確認したので、プロセス間の遅延(過去にハマった罠)が原因ではないことも確認済み。

原因は未解明(Mail送信ボタンがiOS16のコンテキストメニュー対応ボタン`menuOnLongPress`付きで、合成タッチの認識ロジックと相性が悪い可能性)。**座標タップは万能ではない**という教訓として記録。

### 3. target/actionを特定して直接呼ぶ

Mail.app自体のプロセス(`MobileMail`、`ps aux | grep -i mobilemail`でPID取得)にFridaをattachし、ウィンドウ階層を辿ってSendボタンの実体を特定:

```javascript
var appCls = objc_getClass(Memory.allocUtf8String("UIApplication"));
var app = objc_msgSend_ptr(appCls, sel("sharedApplication"));
var windows = objc_msgSend_ptr(app, sel("windows"));
var window0 = /* windows[0] */;
var rootVC = objc_msgSend_ptr(window0, sel("rootViewController"));
var composeNav = objc_msgSend_ptr(rootVC, sel("presentedViewController")); // ComposeNavigationController
var composeVC = objc_msgSend_ptr(composeNav, sel("topViewController"));    // MFMailComposeController
var navItem = objc_msgSend_ptr(composeVC, sel("navigationItem"));
var rightItem = objc_msgSend_ptr(navItem, sel("rightBarButtonItem"));
var target = objc_msgSend_ptr(rightItem, sel("target"));  // == composeVC
var action = objc_msgSend_ptr(rightItem, sel("action"));  // SEL "send:"
```

判明した実体: `-[MFMailComposeController send:]`(`sender`として自分自身の`UIBarButtonItem`を渡す、標準的なIBActionの形)。

### 4. メインスレッド上で安全に呼ぶ

```javascript
var objc_msgSend_void_sel_ptr_bool = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "void", ["pointer", "pointer", "pointer", "pointer", "bool"]);

objc_msgSend_void_sel_ptr_bool(
    composeVC,
    sel("performSelectorOnMainThread:withObject:waitUntilDone:"),
    sel("send:"),
    rightItem,
    0 // waitUntilDone: NO -- 呼び出し元(Fridaのスレッド)をブロックしない
);
```

実行後、スクリーンショットでコンポーズ画面が閉じて受信ボックス画面に遷移したことを確認、さらに実際にGmail受信箱(別端末から目視)にも到達していることを確認した。`MobileMail`・`SpringBoard`ともにクラッシュなし。

## 教訓

- **座標タップは「本物の入力経路を通る」という意味では安全だが、確実とは限らない** — ボタンの実装次第(特にiOS16のコンテキストメニュー付きボタン)で合成タッチが認識されないケースがある。失敗しても即座に危険というわけではないので(今回も両プロセス安定)、気軽に試してダメなら次の手(target/action直接呼び出し)に進めば良い。
- **`-send:`のような「本来UIイベントでしか呼ばれないメソッド」を直接`objc_msgSend`で呼ぶとメインスレッドアサートで安全に`abort`する**(クラッシュはしない)。これは[過去に`-isLockScreenVisible`等で遭遇した現象](ios-touchless-wake-unlock.md)と同じパターン。
- **`dispatch_async_f` + `NativeCallback`は使わない**(過去にSpringBoardを4回クラッシュさせた実績あり)。代わりに**`-[NSObject performSelectorOnMainThread:withObject:waitUntilDone:]`** を使うと、Objective-Cランタイム自身の仕組みでメインスレッドに乗せられるので安全。`waitUntilDone:NO`にしておけば万一ハングしてもFrida側のスレッドをブロックしない。
- この「target/action特定 → `performSelectorOnMainThread:withObject:waitUntilDone:`で呼ぶ」パターンは、Mail送信に限らず**任意のアプリの任意のボタンを安全に押す**汎用手法として使える。
