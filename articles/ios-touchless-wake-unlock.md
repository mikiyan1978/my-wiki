# iOSをSSH経由だけで触れずに起こしてホームまで到達させる(Frida + IOHIDEvent)

ジェイルブレイク済みiPhoneに、**物理的なタッチや画面確認を一切せず**(=SSHだけ)画面をオンにし、ロック画面を上スワイプでホーム画面まで進め、そのままアプリを起動する手順。[`runningboardd-recon`](runningboardd-recon.md)の調査中、外出先からSafariを起動したくて必要になった。

**前提条件**: パスコード/Face IDを設定していないこと(palera1nではデフォルトでそうなっている)。パスコードがある場合、電源ボタンのシミュレーションで画面を起こすところまでは安全にできるが、**ロック解除そのものは認証が必要な領域なので意図的に行わない**。

---

## 結論だけ先に

**最終的に確実に動いた手順(この順番、全てHIDレベルの合成イベント)**:

1. 電源ボタン押下を再現 → 画面オン
2. Homeボタン押下を再現(Consumerページ、usage `0x40`)
3. 1本指の上スワイプを再現

この3つを順番に送ると、`isUILocked`が`1`→`0`に変わり、実際にホーム画面に到達する(第5章で検証済み)。電源ボタンだけ、スワイプだけ、Homeボタンだけ、では単独では解除まで到達しなかった — **3つの組み合わせで初めて本物の解除シーケンスとして認識される**らしい(詳しい内部挙動は第5章の観察ログ参照)。

**往復の再現性も確認済み**: ホーム画面の状態から電源ボタンを1回押すだけで画面オフ・自動再ロック(`isUILocked`が`0`→`1`)、そこから再び上記3点セットでホーム画面(`isUILocked`が`1`→`0`)に戻れる。この「オフ→ホーム→オフ→ホーム」のサイクルを2周連続で確認し、毎回`isUILocked`の値とスクリーンショット(実際の時刻・アイコン表示)の両方で裏取りした。SpringBoardのクラッシュ・PID変化も一切なし。

いずれも**本物の入力経路(IOHIDEventSystemClient)**を通すので、SpringBoardの内部メソッドを直接叩くより安全(後述の「やってはいけないこと」参照)。

## 環境

- iPhone 8 Plus, iOS 16.7.16, palera1n rootful脱獄
- `frida-server -l 0.0.0.0:27042 -D`(起動済み。落ちていたら`ssh root@<device> "killall -9 frida-server; nohup /usr/sbin/frida-server -l 0.0.0.0:27042 -D > /var/mobile/frida_server.log 2>&1 &"`)
- Mac側: `frida==17.19.0`のvenv、`~/Frida/springboard_recon/run_attach.py`(タイムアウト付きattach→detachランナー)
- 対象プロセス: `SpringBoard`(`ps ax | grep -i springboard`でPIDを取得)

対象がSpringBoardである点に注意: ここでの呼び出しは`Interceptor.attach`を一切使わない、**その場で呼んで即detachする一回限りの呼び出し**なので、`script.unload()`禁止ルールの対象外(読み取り専用フックの話とは別物)。

## 1. 画面を起こす(電源ボタン相当)

`IOHIDEventSystemClientCreate` → `IOHIDEventCreateKeyboardEvent`(Consumerページ、Power usage) → `IOHIDEventSystemClientDispatchEvent`で、電源ボタンのDOWN/UPを合成する。backboardd/SpringBoardは本物のボタン操作と全く同じに扱う。

`wake_screen.js`:
```javascript
var IOHIDEventSystemClientCreate = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSystemClientCreate"), "pointer", ["pointer"]);
var IOHIDEventCreateKeyboardEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventCreateKeyboardEvent"), "pointer", ["pointer", "uint64", "uint32", "uint32", "int", "uint32"]);
var IOHIDEventSystemClientDispatchEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSystemClientDispatchEvent"), "void", ["pointer", "pointer"]);
var mach_absolute_time = new NativeFunction(Module.findGlobalExportByName("mach_absolute_time"), "uint64", []);

var kHIDPage_Consumer = 0x0C;
var kHIDUsage_Csmr_Power = 0x30;

setTimeout(function () {
    var client = IOHIDEventSystemClientCreate(ptr(0));
    var downEvent = IOHIDEventCreateKeyboardEvent(ptr(0), mach_absolute_time(), kHIDPage_Consumer, kHIDUsage_Csmr_Power, 1, 0);
    IOHIDEventSystemClientDispatchEvent(client, downEvent);
    setTimeout(function () {
        var upEvent = IOHIDEventCreateKeyboardEvent(ptr(0), mach_absolute_time(), kHIDPage_Consumer, kHIDUsage_Csmr_Power, 0, 0);
        IOHIDEventSystemClientDispatchEvent(client, upEvent);
    }, 120);
}, 200);
```

実行:
```bash
python3 run_attach.py <SpringBoardのPID> wake_screen.js 3
```

画面が既にオンの場合はこれで消える(トグル動作)。オフのときに1回押せば点く、という単純な物理ボタンの挙動そのまま。

## 2. ロック画面を上スワイプで解除する

ここが本題。電源ボタンだけでは画面が点くだけで、ホーム画面には進まない(パスコードが無くても、上スワイプの操作自体は必要)。

`IOHIDEventCreateDigitizerEvent`(親イベント)に`IOHIDEventCreateDigitizerFingerEvent`(指1本分の子イベント)を`IOHIDEventAppendEvent`で載せ、DOWN→MOVE(複数回)→UPの順で時間差をつけて送る。実装は[xuan32546/IOS13-SimulateTouch](https://github.com/xuan32546/IOS13-SimulateTouch)の`pccontrol/Touch.xm`を参考にした(同種の手法は[SimulateTouch](https://github.com/iolate/SimulateTouch)など他の実装にも見られる、ジェイルブレイク界隈で広く知られたテクニック)。

`swipe_up.js`:
```javascript
var IOHIDEventSystemClientCreate = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSystemClientCreate"), "pointer", ["pointer"]);
var IOHIDEventSystemClientDispatchEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSystemClientDispatchEvent"), "void", ["pointer", "pointer"]);
var IOHIDEventCreateDigitizerEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventCreateDigitizerEvent"), "pointer", ["pointer", "uint64", "uint32", "uint32", "uint32", "uint32", "float", "float", "float", "float", "float", "int", "int", "uint32"]);
var IOHIDEventCreateDigitizerFingerEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventCreateDigitizerFingerEvent"), "pointer", ["pointer", "uint64", "uint32", "uint32", "uint32", "float", "float", "float", "float", "float", "int", "int", "uint32"]);
var IOHIDEventAppendEvent = new NativeFunction(Module.findGlobalExportByName("IOHIDEventAppendEvent"), "void", ["pointer", "pointer", "uint32"]);
var IOHIDEventSetIntegerValue = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSetIntegerValue"), "void", ["pointer", "uint32", "int64"]);
var IOHIDEventSetFloatValue = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSetFloatValue"), "void", ["pointer", "uint32", "float"]);
var IOHIDEventSetSenderID = new NativeFunction(Module.findGlobalExportByName("IOHIDEventSetSenderID"), "void", ["pointer", "uint64"]);
var mach_absolute_time = new NativeFunction(Module.findGlobalExportByName("mach_absolute_time"), "uint64", []);

var TOUCH_DOWN = 0, TOUCH_MOVE = 1, TOUCH_UP = 2;
var SENDER_ID = 0x8000000817319e45; // 固定のプレースホルダ値で動作した(0だと送信自体が無視される)

function makeFingerChild(type, x, y) {
    var eventMask, range, touch;
    if (type === TOUCH_DOWN) { eventMask = 3; range = 1; touch = 1; }
    else if (type === TOUCH_MOVE) { eventMask = 4; range = 1; touch = 1; }
    else { eventMask = 2; range = 0; touch = 0; }
    // index=2, identity=3 はIOS13-SimulateTouchの実装そのまま
    var child = IOHIDEventCreateDigitizerFingerEvent(ptr(0), mach_absolute_time(), 2, 3, eventMask, x, y, 0.0, 0.0, 0.0, range, touch, 0);
    IOHIDEventSetFloatValue(child, 0xb0014, 0.04); // major radius
    IOHIDEventSetFloatValue(child, 0xb0015, 0.04); // minor radius
    return child;
}

function dispatchTouch(client, type, x, y) {
    // transducerIndex=3, transducerType=99, eventMask=1 もIOS13-SimulateTouchの実装そのまま
    var parent = IOHIDEventCreateDigitizerEvent(ptr(0), mach_absolute_time(), 3, 99, 1, 0, 0.0, 0.0, 0.0, 0.0, 0.0, 0, 0, 0);
    IOHIDEventSetIntegerValue(parent, 0xb0019, 1);
    IOHIDEventSetIntegerValue(parent, 0x4, 1);
    IOHIDEventAppendEvent(parent, makeFingerChild(type, x, y), 0);
    IOHIDEventSetIntegerValue(parent, 0xb0007, 0x23);
    IOHIDEventSetIntegerValue(parent, 0xb0008, 0x1);
    IOHIDEventSetIntegerValue(parent, 0xb0009, 0x1);
    IOHIDEventSetSenderID(parent, SENDER_ID); // これが無いと(0のまま)システムに無視される
    IOHIDEventSystemClientDispatchEvent(client, parent);
}

setTimeout(function () {
    var client = IOHIDEventSystemClientCreate(ptr(0));
    var x = 0.5; // 画面中央(x/yは0.0〜1.0の正規化座標)
    var steps = [0.92, 0.80, 0.65, 0.50, 0.35, 0.20, 0.08]; // 下から上へ
    var i = 1;
    dispatchTouch(client, TOUCH_DOWN, x, steps[0]);
    function nextStep() {
        if (i < steps.length) {
            dispatchTouch(client, TOUCH_MOVE, x, steps[i]);
            i++;
            setTimeout(nextStep, 25); // 25ms間隔(人間のスワイプ相当の速度)
        } else {
            dispatchTouch(client, TOUCH_UP, x, steps[steps.length - 1]);
        }
    }
    setTimeout(nextStep, 25);
}, 150);
```

実行:
```bash
python3 run_attach.py <SpringBoardのPID> wake_screen.js 3   # まず画面オン
python3 run_attach.py <SpringBoardのPID> swipe_up.js 4      # 続けて上スワイプ
```

実機確認: この直後に`uiopen --bundleid com.apple.mobilesafari`を実行したところ、**実際にMobileSafariプロセスが起動した**(ロック中は何度やっても失敗していたのが、ここで初めて成功)。クラッシュログの増加なし、SpringBoard PIDも不変。

## 3. スクリーンショットを撮る(`_UICreateScreenUIImage`)

ウィンドウ探索や`drawViewHierarchyInRect:`/`renderInContext:`は**メインスレッド必須**で、Frida経由(別スレッド)から直接呼ぶと`abort`(運が良ければ)または実クラッシュ(後述)になる。[nekohasekai/icli](https://github.com/nekohasekai/icli)(`Sources/IcliPrivate/IcliPrivate.m`)の実装を見て解決: 実体は引数なしの単純なC関数`_UICreateScreenUIImage()`で、構造体渡しもウィンドウ取得も不要、**トップレベルから直接呼んでも安全**だった。

`take_screenshot.js`:
```javascript
var _UICreateScreenUIImage = new NativeFunction(Module.findGlobalExportByName("_UICreateScreenUIImage"), "pointer", []);
var UIImagePNGRepresentation = new NativeFunction(Module.findGlobalExportByName("UIImagePNGRepresentation"), "pointer", ["pointer"]);
var objc_getClass = new NativeFunction(Module.findGlobalExportByName("objc_getClass"), "pointer", ["pointer"]);
var sel_registerName = new NativeFunction(Module.findGlobalExportByName("sel_registerName"), "pointer", ["pointer"]);
var objc_msgSend_ptr_ptr = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "pointer", ["pointer", "pointer", "pointer"]);
var objc_msgSend_str_bool_bool = new NativeFunction(Module.findGlobalExportByName("objc_msgSend"), "bool", ["pointer", "pointer", "pointer", "int"]);
function sel(name) { return sel_registerName(Memory.allocUtf8String(name)); }

setTimeout(function () {
    var image = _UICreateScreenUIImage();
    var pngData = UIImagePNGRepresentation(image); // UIImageJPEGRepresentationはこのimageに対してなぜか常にnilを返した。PNGは問題なし
    var nsStringCls = objc_getClass(Memory.allocUtf8String("NSString"));
    var pathStr = objc_msgSend_ptr_ptr(nsStringCls, sel("stringWithUTF8String:"), Memory.allocUtf8String("/var/mobile/screenshot.png"));
    objc_msgSend_str_bool_bool(pngData, sel("writeToFile:atomically:"), pathStr, 1);
}, 150);
```

`scp`でPNGを取り出せばそのまま画面の中身が見える(時刻・通知・ロック画面の壁紙まで正確に写る)。**画面がオフの時は普通に真っ黒なPNGが返る**(バグではなく正しい挙動) — 撮る前に`wake_screen.js`で起こすこと。

### 失敗した試行(クラッシュ6回)

同じスクリーンショットを`drawViewHierarchyInRect:afterScreenUpdates:`や`-[CALayer renderInContext:]`で撮ろうとして、`dispatch_async_f` + `NativeCallback`でメインスレッドに乗せる方式を使ったところ、**内容に関係なく4/4でSpringBoardがクラッシュした**(`_dispatch_main_queue_drain`内の`EXC_BAD_ACCESS`、毎回同じ無効アドレス)。`-[UIApplication sendEvent:]`のような超高頻度に呼ばれる関数へ`Interceptor.attach`で割り込むのも1回クラッシュした(`_os_unfair_lock_corruption_abort`、フック設置の瞬間に発生 — ホットパスへのランタイムパッチは別スレッドの同時実行とレースする)。**教訓: `dispatch_async_f`/`NativeCallback`でメインスレッドに処理を乗せる方式はこの環境で信頼できない。超高頻度に呼ばれる関数へのフックも避ける。** `_UICreateScreenUIImage()`のように「そもそもどのスレッドからでも安全に呼べるよう作られた関数」を探す方が確実。

## 4. ロック画面上のシステムアラートを操作する(UIAlertActionのハンドラを直接呼ぶ)

上記のクラッシュの巻き添えで端末がSafe Modeに入り(SpringBoard tweak無効化)、「Exit Safe Mode」のアラートがロック画面の上に出た。この機種(iPhone 8 Plus、物理Homeボタン)は**上スワイプが unlock ジェスチャーとして機能しない**(Face ID機のみの挙動だった)ため、スワイプやHomeボタンHIDイベントでは解除できず、アラート自体もロック中は非対話的(タップしてもロック画面自身の「ホームボタンを押して開く」ヒントが出るだけ)。

解決策: **タッチもロック解除も介さず、アラートのボタンのハンドラブロックを直接見つけて呼び出す**。

1. `[[UIApplication sharedApplication] windows]`を全走査し、`rootViewController.presentedViewController`を辿って`UIAlertController`を探す(読み取り専用、副作用なし)
2. `.actions`配列から目的の`UIAlertAction`を`title`で特定
3. `-[UIAlertAction handler]`(公開同然のアクセサ、`@?16@0:8`=ブロックを返すだけ)でハンドラブロックを取得
4. ブロックのABI(`isa`+`flags`+`reserved`+`invoke`、`invoke`はオフセット16)から`invoke`関数ポインタを読み、`invoke(block, action)`として直接呼ぶ

```javascript
var alertVC = /* windows walk, see find_alert.js */;
var actions = objc_msgSend_ptr(alertVC, sel("actions"));
// ... actions.count 分 objectAtIndex: で回して title が一致するものを targetAction に ...
var handlerBlock = objc_msgSend_ptr(targetAction, sel("handler"));
var invokePtr = handlerBlock.add(16).readPointer(); // Block_layout: isa(8)+flags(4)+reserved(4)+invoke
var invoke = new NativeFunction(invokePtr, "void", ["pointer", "pointer"]);
invoke(handlerBlock, targetAction);
```

実行したら即座にSpringBoardがリスプリング(ハンドラ自体がそれを行うため、呼び出し元のFridaスクリプトは「script is destroyed」で終わるのが正常)。再起動後、ロック画面からSafe Modeアラートが消えていることをスクリーンショットで確認、クラッシュログの増加なし。

この手法はSafe Mode以外の「ロック画面の上に出る系システムアラート」全般に応用できる(タッチ精度やロック状態を気にせず、ボタンを押したのと同じ効果を直接起こせる)。

## 5. 本当にホーム画面まで到達させる(観察ベースのブレークスルー)

iPhone 8 Plus(物理Homeボタン、Touch ID)は**Face ID機専用の「上スワイプで解除」ジェスチャーを持たない**ため、2章のスワイプ単体や、電源ボタン+スワイプの組み合わせだけでは実際には解除されない(`isUILocked`は`1`のまま)、ということが後になって判明した。

### 遠回りした道のり: 直接メソッド呼び出しは次々ハングする

「本物の解除処理」を直接呼んで解除しようとしたが、以下は**全てハング**した(Fridaクライアント側が応答を待ち続ける。SpringBoard自体はCPU 0%で無傷、クラッシュはしない):

- `-[SBLockScreenManager unlockUIFromSource:withOptions:]`
- `-[SBLockScreenManager attemptUnlockWithMesa]`
- `-[CSCoverSheetViewController prepareForUIUnlock]` → `willUIUnlockFromSource:` → `finishUIUnlockFromSource:`(連続呼び出し)
- `-[UIApplication _simulateHomeButtonPressWithCompletion:]`(AssistiveTouchが実際に使う正規API)
- `-[CSCoverSheetViewController _setDismissed:]`(単純なBOOL setterのはずだった)
- `-[SBHomeHardwareButton emulateHomeButtonEventsIfNeeded:]`(本物の`IOHIDEventRef`を渡しても)

唯一ハングしなかったのは`-[SBHomeHardwareButton initialButtonDown:]`/`singlePressUp:`を`nil`引数で呼んだ場合だが、何も起きなかった(引数から状態を読む実装なので、`nil`だと早期リターンしていると見られる)。

**推測される原因**: これらのメソッドは内部で`dispatch_sync`等によってメインスレッドへ処理を委譲しており、Frida注入スレッドが「本物のメインスレッド」と認識されないため自己デッドロックしている可能性が高い。

### 解決策: 推測をやめて、広範囲に観察する

直接呼ぶのをやめ、**候補メソッドに片っ端から読み取り専用フックを仕掛けて、実際に発火する経路を観察する**方針に切り替えた(`observe_unlock_path.js`)。`SBHomeHardwareButton`・`CSCoverSheetViewController`・`SBLockScreenManager`・`SpringBoard`の計38メソッドに`Interceptor.attach`の`onEnter`のみ(`%orig`は常に継続、戻り値は一切触らない)でフックし、`run_persistent.py`で常駐させた。

その状態で**電源ボタン→Homeボタン→上スワイプ**の順に合成イベントを送ったところ、ログに次が記録された:

```
CSCoverSheetViewController -_setDismissed:  arg2=(desc err)
SBLockScreenManager -lockScreenViewControllerDidDismiss
```

直後に`isUILocked`を確認すると`0`になっており、スクリーンショットで実際のホーム画面(アプリアイコン・Dock)が撮影できた。日時も実際の時刻(`2026/10/14(水) 11:03:14`)と一致。

### 教訓

- 内部APIを**推測で直接呼ぶ**のはハング/クラッシュのリスクが高い
- 代わりに**広範囲の読み取り専用フックを仕掛けてから、既に安全だと分かっている入力(HIDイベント)を実際に送り、どの経路が本当に呼ばれるかを観察する**方が、遥かに安全かつ確実
- `_setDismissed:`(前章で直接呼んでハングした同じメソッド!)は、**正しい文脈(本物の解除シーケンスの一部として)で呼ばれる分には問題なく動く** — ハングするのは「文脈を無視して単独で呼んだ」ことが原因だったと考えられる
- 結局、電源ボタン単体・スワイプ単体では解除されず、**電源ボタン→Homeボタン→スワイプの3点セット**が必要だった

## 6. バッテリー切れ・再ジェイルブレイク後にハマった点(2026-10-14)

長時間の検証(複数の`run_persistent.py`を同時に張りっぱなし + 画面の点灯/消灯を繰り返す)の結果、デバイスのバッテリーが完全に切れて電源が落ちた。

### palera1nはsemi-tethered — 完全な電源断は脱獄が外れる

バッテリー切れ→充電→自動起動の後、SSHが`Host is down`から`Connection refused`に変わった(ネットワーク層は復帰したがSSHデーモンが存在しない)。これは**palera1nがA11チップ向けのsemi-tethered(準有線)脱獄であるため、完全な電源断を経ると非脱獄状態で起動する**ことが原因。SSH/frida-serverも脱獄の上に乗っている機能なので両方消える。

**復旧にはUSB接続での再脱獄が必須**(ネットワーク経由では不可能)。再脱獄後は以下を忘れずに:
- `frida-server`を`-l 0.0.0.0:27042`で再起動(環境ゴッチャの定番、再起動のたびに必要)
- SpringBoardのPIDが変わっている(今回は`39` — 起動直後は小さい番号になりやすい)

### `run_attach.py`の隠れた遅延が引き起こした「3点セットが効かない」再発

再脱獄後、本章冒頭の「電源→Home→スワイプ」の3点セットを**`run_attach.py`を3回に分けて**(`python3 run_attach.py <pid> wake_screen.js && python3 run_attach.py <pid> press_home.js && python3 run_attach.py <pid> swipe_up.js`)実行したところ、2回連続で解除に失敗した(`isUILocked`が`1`のまま)。

観察フック(`observe_unlock_path.js`)のログを見ると、スワイプが`CSCoverSheetViewController -_setDismissed:`/`lockScreenViewControllerDidDismiss`ではなく、`SBLockScreenManager -_runUnlockActionBlock: arg2=nil`→`lockUIFromSource:withOptions:`という**無効な解除試行→再ロック**のパスを辿っていた。

原因は`run_attach.py`自体の実装:

```python
script.load()
print("ATTACHED_AND_LOADED, pid=", pid)
time.sleep(duration)  # デフォルト15秒!
script.unload()
session.detach()
```

スクリプト内の実際のボタン操作(`setTimeout`で150〜300ms後に発火)はすぐ終わるが、**プロセス自体はその後も`duration`(デフォルト15秒)分スリープしてから`detach`する**。そのため`wake→home→swipe`を3つの別プロセスで逐次実行すると、各ステップの「実際のボタン操作」の間に**15秒近い間隔**が生まれてしまい、途中で画面が自動消灯/ロック状態が変化して、スワイプが本来の解除ジェスチャーとして認識されなくなっていた。

**対処**: 電源→Home→スワイプ→`isUILocked`確認までを**1つのFridaセッション内で**、実際の操作感に近い短い間隔(600ms程度)で連続実行するスクリプトに統合した(`unlock_combined.js`)。これで一発で`isUILocked`が`0`になり、スクリーンショットでもホーム画面到達を確認できた。

### 教訓

- `run_attach.py`/`run_persistent.py`のような「attach→実行→detach」ランナーを**複数回に分けて連続実行する**場合、各呼び出しの間の**実時間の空き**を見落としやすい(プリント自体はすぐ出るので気づきにくい) — タイミングがシビアな一連の操作(今回のような解除シーケンス)は、**可能な限り1セッション内で完結させる**のが安全
- `_UICreateScreenUIImage()`によるスクリーンショットは「画面が黒 = 本当に消灯」の判定に使えるが、**別プロセスを跨ぐと、スクリーンショットを撮る時点では既に状態が変わっている**ことがあるので、診断時は同一セッション内の連続スクリーンショット(`wake_then_screenshot_series.js`方式)で切り分けるのが確実
- バッテリー切れ(=完全な電源断)はsemi-tetheredジェイルブレイクでは「リスプリング」よりずっと重い事象 — USB物理アクセスが必須になる点を留意

## やってはいけないこと: SBLockScreenManagerへの直接呼び出し

同じ目的で、最初は`-[SBLockScreenManager unlockUIFromSource:withOptions:]`のような**プライベートメソッドを直接呼ぶ**アプローチを試した。結果:

- 読み取り専用のはずの`-isLockScreenVisible`を呼んだだけで`abort was called`(Frida側がスレッド違いの呼び出しを検知して安全に止めていた)
- `dispatch_async_f` + `_dispatch_main_q`で正しくメインスレッドに乗せて同じ呼び出しをやり直したところ、**今度は本当にSpringBoardが`EXC_BAD_ACCESS`(SIGSEGV)でクラッシュしてリスプリングした**(クラッシュログで`_dispatch_main_queue_drain`内での無効アドレスアクセスと確認)
- `-unlockUIFromSource:withOptions:`の直接呼び出しは、Fridaクライアント側が長時間ハングし、`frida-server`の再起動が必要になった(SpringBoard自体はCPU 0%で待機中、クラッシュはしていなかったが、原因不明の長時間ブロック)

結論: **SBLockScreenManagerのような「本来UIイベント駆動でしか呼ばれない」クラスを、合成イベント経由ではなく直接メソッド呼び出しで操作するのは危険**。IOHIDEvent経由(今回の方法)は実際のタッチ/ボタン処理と同じコードパスを通るため、内部状態の前提が崩れず安全だった。

## 参考

- [feat(ios): press volume and power buttons on simulators through the IOHID consumer-key path](https://github.com/kaeawc/auto-mobile/pull/8370)
- [pressButton: support volume_up/volume_down/power on the iOS simulator via private IOHID dispatch](https://github.com/kaeawc/auto-mobile/issues/8360)
- [xuan32546/IOS13-SimulateTouch](https://github.com/xuan32546/IOS13-SimulateTouch) — `pccontrol/Touch.xm`がデジタイザイベント合成の元ネタ
- [iolate/SimulateTouch](https://github.com/iolate/SimulateTouch) — 同種の手法の別実装
