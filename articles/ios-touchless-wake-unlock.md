# iOSをSSH経由だけで触れずに起こしてホームまで到達させる(Frida + IOHIDEvent)

ジェイルブレイク済みiPhoneに、**物理的なタッチや画面確認を一切せず**(=SSHだけ)画面をオンにし、ロック画面を上スワイプでホーム画面まで進め、そのままアプリを起動する手順。[`runningboardd-recon`](runningboardd-recon.md)の調査中、外出先からSafariを起動したくて必要になった。

**前提条件**: パスコード/Face IDを設定していないこと(palera1nではデフォルトでそうなっている)。パスコードがある場合、電源ボタンのシミュレーションで画面を起こすところまでは安全にできるが、**ロック解除そのものは認証が必要な領域なので意図的に行わない**。

---

## 結論だけ先に

1. 電源ボタン押下をHIDレベルで再現 → 画面がオンになる
2. 1本指の上スワイプをHIDレベルで再現 → ロック画面が解除されホーム画面に到達
3. 普通に`uiopen`等でアプリを起動すればそのまま立ち上がる

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
