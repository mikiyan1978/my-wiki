# runningboardd 調査メモ(勉強用)

iOS 16.7.16 (iPhone 8 Plus, palera1n rootful脱獄) の `runningboardd` を、Fridaで**読み取り専用**に調査するメモ。学習目的で、挙動観察から始める。

このメモは調査の進行に合わせて都度更新する。

---

## runningboarddとは

全プロセスのライフサイクル(起動・終了・バックグラウンド遷移・アサーション/権限付与)を管理するiOSのシステムデーモン。アプリをサスペンド/ジェットサムする最終判断もここが持つため、**クラッシュさせるとSpringBoardよりも被害が大きい**(最悪リブートが必要になる可能性がある)という前提で、観察のみ・戻り値は一切触らない方針で進めている。

## 環境

- SSH: `root@192.168.3.163`(自宅LAN。外出先でも、このセッションを操作しているMac自体が自宅LANに残っていれば到達できることを確認済み)
- frida-server: `/usr/sbin/frida-server -l 0.0.0.0:27042 -D` (既に起動済みだった)
- 調査プロジェクト: `~/Frida/runningboardd_recon/`(専用venv、`frida==17.19.0`)
- 既存のSpringBoard調査と同じ安全方針を継承:
  - `objc_copyClassList`の全列挙は使わず、`class_getImageName`で対象イメージに絞ってから`class_getName`
  - 存在未確認のクラス名は`objc_getClass`で先に存在確認
  - `frida.spawn()`は使わない、既存プロセスに`attach`のみ
  - `Interceptor.attach`を使ったら`script.unload()`は絶対に呼ばない(`run_persistent.py`でCtrl+Cしてもフックは残したままコントローラだけ終了)

## 1. ロード済みモジュール

`runningboardd`(PID例: 66861)が読み込んでいる関連フレームワーク:

- `/usr/libexec/runningboardd`(本体。ObjCクラスは0個 → 本体は素のC/C++実装)
- `RunningBoard.framework`
- `RunningBoardServices.framework`
- `AssertionServices.framework`
- `BaseBoard.framework`

## 2. クラス一覧(対象イメージのみに絞ったダンプ)

全13,841クラス中、上記5イメージに絞って**361クラス**を抽出(`dump_class_names.js`)。

| イメージ | クラス数 |
|---|---|
| RunningBoardServices | 127 |
| BaseBoard | 124 |
| RunningBoard | 101 |
| AssertionServices | 9 |

主な系統:
- `RBS*` — Public向けのRunningBoardServices API(`RBSAssertion`, `RBSProcessMonitor`, `RBSProcessState`, `RBSLaunchRequest` など)
- `RB*`(RBSなし) — デーモン内部実装(`RBAssertionManager`, `RBProcessManager`, `RBLaunchManager` など)
- `BS*` — BaseBoardの汎用ユーティリティ(XPC, タイマー, アサーション基盤)
- `BKS*` — 古い世代のBackBoardServices系API(`BKSProcessAssertion`など、後方互換層)

## 3. フック対象の選定と観察結果

「挙動から」学ぶ方針で、以下5つを**onEnterのみ**でフック(`observe_behavior.js`)。onLeaveは一切書かず、戻り値には絶対に触れない:

| クラス | セレクタ | 意味 |
|---|---|---|
| `RBAssertionManager` | `-processDidLaunch:` | プロセス起動イベント |
| `RBAssertionManager` | `-processDidTerminate:` | プロセス終了イベント |
| `RBAssertionManager` | `-invalidateAssertion:` | アサーション無効化 |
| `RBProcessStateChange` | `-initWithIdentity:originalState:updatedState:` | プロセスの状態遷移(旧状態→新状態) |
| `RBSAssertion` | `-acquireWithError:` | アサーション取得 |

ログは `/var/mobile/Class Dump/runningboardd/behavior_observe.log` に追記。

### 結果

- 5フックすべて正常にインストール、`runningboardd`のPID・CrashReporterログともに異常なし(安定稼働を確認)
- 観察開始直後は画面オフ・待機中だったため、最初に発火したのはバックグラウンドデーモン(`com.apple.icloud.fmfd`, `com.apple.icloud.searchpartyd`)の`role:None`↔`role:Background`遷移のみだった

### 第2弾フック(`observe_behavior2.js`、別セッションとして追加)

同じPIDに**独立した2つ目のFridaセッション**を張って追加(既存フックは解除しない方針を厳守)。「アプリ起動の実際の経路はどこか」を確かめる目的:

| クラス | セレクタ |
|---|---|
| `RBProcessManager` | `-executeLaunchRequest:withError:` |
| `RBLaunchManager` | `-executeLaunchRequest:existingProcess:requestIdentity:withError:` |
| `RBProcessMonitor` | `-didAddProcess:withState:` |
| `RBProcessMonitor` | `-didRemoveProcess:withState:` |
| `RBProcessMonitor` | `-didUpdateProcessStates:` |

### 分かったこと

`uiopen --bundleid com.apple.mobilesafari`でSafariを起動 → `killall MobileSafari`で終了、を試した結果:

1. **`processDidLaunch:`/`acquireWithError:`は一度も発火しなかった** — Safariの通常起動では使われていない(もっと内部的な経路を通る)らしい
2. **`killall`による終了は確実に捕捉できた**:
   ```
   RBAssertionManager -processDidTerminate:  arg2=[application<com.apple.mobilesafari>:47204]
   RBProcessStateChange init  role:None → role:None
   RBProcessMonitor -didUpdateProcessStates:  arg2=<RBProcessStateChangeSet: 0x82e8b13a0>
   RBProcessMonitor -didRemoveProcess:withState:  arg2=[application<com.apple.mobilesafari>:47204]
   ```
   → **`didRemoveProcess:withState:`がプロセス終了の実際の信号**だと確認できた
3. **画面ロック中の再起動リクエストは、runningboardd側のどのフックにも到達しなかった**(`executeLaunchRequest:`すら発火せず、`ps`にもプロセスが現れない)。つまりロック中のフォアグラウンド起動拒否は、**runningboarddより上位層(SpringBoard/FrontBoard)で止められている**可能性が高い。起動側(`didAddProcess:`等)を捕捉するには、画面をロック解除した状態で試す必要がある。

---

## 画面オンの方法(SSH経由、物理ボタン相当)

外出先からでもSSH経由でウェイクできるか調査。既知の手法(auto-mobile等のiOSシミュレータ操作ツールやジェイルブレイク界隈で広く使われている方式)を発見:

```
IOHIDEventSystemClientCreate(NULL)
  → IOHIDEventCreateKeyboardEvent(NULL, mach_absolute_time(), page=0x0C(Consumer), usage=0x30(Power), down=1, flags=0)
  → IOHIDEventSystemClientDispatchEvent(client, event)
  (120ms後にdown=0で同様にUPイベントも送信)
```

これは**物理的な電源ボタン押下をHIDレベルで再現する**もので、backboardd/SpringBoardは実際のボタン操作と同じに扱う。パスコードをバイパスするものではない(ロック画面を表示するだけ)。

SpringBoard(PID例: 28055)に対してFridaで一回限りの呼び出し(`wake_screen.js`、Interceptor.attach無し、呼んで即detach)として実行:
- シンボル解決(`IOHIDEventSystemClientCreate`等)は全て成功
- 呼び出し自体はエラーなく完了、SpringBoardもクラッシュせず安定継続
- ただしこの後も`uiopen`でのSafari起動は失敗したままだった → **画面を起こすだけでは足りず、実際にロック解除(パスコード/Face ID)されていないとフォアグラウンド起動はできない**らしいと判明。パスコード/Face IDのバイパスは意図的に行わない(ユーザー自身の操作が必要な領域)。

## SBLockScreenManagerへの直接呼び出しは危険(実際にクラッシュした)

ロック状態を読むだけの`isUILocked`(安全、`1`=ロック中と確認できた)に続けて、`isLockScreenVisible`を呼んだところ`abort was called`。これはFridaがスレッド違い(メインスレッド以外からの呼び出し)を安全側で止めた「幸運な回避」だった。

`dispatch_async_f` + `_dispatch_main_q`で正しくメインスレッドに乗せて同じ呼び出しをやり直したところ、**実際にSpringBoardが`EXC_BAD_ACCESS`(SIGSEGV, 無効アドレス`0x138`)でクラッシュしてリスプリングした**(`_dispatch_main_queue_drain`の中、スタックトレースで確認)。デバイス自体のリブートはなし、`runningboardd`も無事だったが、**`SBLockScreenManager`の状態系メソッドはこのiOS16.7.16の状態では呼ばない**という結論に至った。以後この方向は完全に断念し、実際のタッチ入力経路(IOHIDEventのデジタイザイベント)に切り替えた。

## 指一本触れず画面オフ→ホーム画面到達(成功)

パスコード未設定(palera1n)という前提のもと、以下の組み合わせで**物理操作ゼロ**でのアプリ起動まで到達できた:

1. 電源ボタン相当のHIDイベント(`wake_screen.js`、既出)で画面をウェイク
2. 1本指の上スワイプを、実際のタッチ入力経路と同じ`IOHIDEvent`デジタイザイベントで再現(`swipe_up.js`):
   ```
   IOHIDEventCreateDigitizerEvent(NULL, time, transducerIndex=3, transducerType=99, eventMask=1, ...)
     + IOHIDEventAppendEvent(parent, IOHIDEventCreateDigitizerFingerEvent(...))
     + IOHIDEventSetSenderID(parent, <固定のseederID値>)
     → IOHIDEventSystemClientDispatchEvent(client, parent)
   ```
   DOWN(y=0.92) → MOVE×6(y=0.80→0.08、25ms間隔) → UPの順で1本指スワイプを合成。実装は[xuan32546/IOS13-SimulateTouch](https://github.com/xuan32546/IOS13-SimulateTouch)の`pccontrol/Touch.xm`を参考にした。
3. `uiopen --bundleid com.apple.mobilesafari`で起動 → **実際にMobileSafariプロセスが起動した**(今まで何度も失敗していたのが、ここで初めて成功)

クラッシュログ増加なし、SpringBoard PIDも不変で安定。`SBLockScreenManager`への直接メソッド呼び出し(危険、上述)を迂回し、**本物の入力経路を模倣する方が安全で確実**という結論。

ただし今回のSafari起動でも、既存の`runningboardd`観察フック(`didAddProcess:withState:`/`executeLaunchRequest:`等)はどれも発火しなかった — ロック状態とは無関係に、Safariの通常起動がそもそも別の経路を通っている可能性が高い(未解明、継続課題)。

## 次にやること

- [ ] Safari起動の実際の経路(`didAddProcess:`等が発火しない理由)を別角度から特定する
- [ ] `RBProcessStateChange`の`originalState`/`updatedState`の`-description`から、状態がどんな文字列で表現されるか継続確認
