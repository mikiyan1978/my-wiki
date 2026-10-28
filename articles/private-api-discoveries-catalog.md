# iOS 16.7.16 プライベートAPI発見カタログ

iPhone 8 Plus(iOS 16.7.16, palera1n rootful)で、これまでのtweak開発・Frida調査の中で「実機で存在確認済み」「型も確認済み」のプライベートAPIを、カテゴリ別にまとめた参照用カタログ。主に2026年のPuck Remake(Puckの独自リコーディング版)開発時の調査と、Eneko Remakeのフォアグラウンドアプリ検知調査から。

各エントリの型エンコーディング(`[B20@0:8B16]`等)はObjective-Cランタイムの`method_getTypeEncoding`が返す生の文字列で、`B`=BOOL、`@`=オブジェクト、`q`=long long、`i`=int、`f`=float、`v`=void、`^`=ポインタ、`@?`=ブロック。数字はスタックオフセット。

**既存の空っぽの`@interface`で型チェックが通っても、クラスメソッド(`+`)呼び出しはリンカが実シンボルを要求する**(詳細は[theos-tweak-troubleshooting-tips.md](theos-tweak-troubleshooting-tips.md)の#13)。各エントリに「リンク可否」を明記した:
- **直接リンク可**: `~/theos/sdks/*.sdk/System/Library/PrivateFrameworks/<Framework>.tbd`に実シンボルがあり、Makefileの`_PRIVATE_FRAMEWORKS`(または`_FRAMEWORKS`)に足すだけで`[Foo someClassMethod]`のようなブラケット構文がそのまま使える。
- **要NSClassFromString**: `.tbd`が無い/見つからない完全に内部的なクラス。`NSClassFromString(@"Foo")`で実行時にClassを取得し、`objc_msgSend`に関数ポインタとして直接キャストして呼ぶ(`.x`ファイル内なら`%c(Foo)`でも同じ)。

---

## 1. Low Power Mode(電源管理)

- **`_CDBatterySaver`** — **iOS 16.7.16ではクラス自体が存在しない**(元のPuckが使っていた旧API)。`BSBatterySaverController`/`SBBatterySaverController`/`CDBatterySaver`等、他に推測した候補名も全て不在。
- **`_PMLowPowerMode`**(`/System/Library/PrivateFrameworks/LowPowerMode.framework`)— **直接リンク可**。`_CDBatterySaver`の実質的な後継。
  - `+sharedInstance` → `id` `[@16@0:8]`
  - `-getPowerMode` → `long long` `[q16@0:8]`
  - `-setPowerMode:fromSource:` → `BOOL` `[B32@0:8q16@24]`
  - `-setPowerMode:fromSource:withCompletion:` → `void` `[v40@0:8q16@24@?32]`
  - `-setPowerMode:fromSource:withParams:` → `BOOL` `[B40@0:8q16@24@32]`
  - `-setPowerMode:fromSource:withParams:withCompletion:` → `void` `[v48@0:8q16@24@32@?40]`(未調査: `withParams:`のNSDictionaryに何のキーを渡せるか)
  - `fromSource:`に渡す`NSString *`定数(全て`extern`宣言すればリンク可能): `kPMLPMSourceControlCenter`, `kPMLPMSourceLostMode`, `kPMLPMSourceReenableBulletin`, `kPMLPMSourceSettings`, `kPMLPMSourceSiri`, `kPMLPMSourceSpringBoardAlert`, `kPMLPMSourceSystemDisable`, `kPMLPMSourceWorkouts`
  - 確認方法: `.tbd`の`objc-classes`で実在確認 → Frida上で`dlopen("/System/Library/PrivateFrameworks/LowPowerMode.framework/LowPowerMode", RTLD_NOW)` → `class_copyMethodList`で生ダンプ。クラス名は`_PMLowPowerMode`(先頭アンダースコア込みが本物の名前、`.tbd`の表記通り)。
- **`PMCoreSmartPowerNap`**(同framework内)— 存在確認・メソッド一覧ダンプ済みだが**未使用(危険と判断し見送り)**。モーションセンサー駆動のXPC常駐サービスで、コールバック登録型:
  - `-registerWithCallback:callback:`, `-registerWithIdentifier:`, `-unregisterWithIdentifier:`, `-reRegister`
  - `-syncState`, `-syncStateWithHandler:`, `-setState:`, `-state`, `-current_state`
  - `-setCSPNMotionAlarmStartThreshold:`, `-setCSPNMotionAlarmThreshold:`, `-setCSPNReentryCoolOffPeriod:`, `-setCSPNReentryDelaySeconds:`, `-setCSPNRequeryDelta:`
  - `-connection`, `-connection_interrupted`, `-setConnection:`
  - 実際の用途(「動かなくなったら何をするモード」なのか)は未解明。XPC接続の状態を誤らせるリスクがあるため、意味論を裏取りするまで封印。

## 2. Bluetooth

- **`BluetoothManager`**(`/System/Library/PrivateFrameworks/BluetoothManager.framework`)— **直接リンク可**。古くから広く使われている安定API。
  - `+sharedInstance` → `id` `[@16@0:8]`
  - `-powered` → `BOOL` `[B16@0:8]`
  - `-setPowered:` → `BOOL` `[B20@0:8B16]`
  - 他に確認できた主なメソッド: `-pairedDevices`, `-connectedDevices`, `-setDiscoverable:`, `-isDiscoverable`, `-connectDevice:`, `-disconnectDevice:`, `-cancelPairing`, `-bluetoothState`, `-localAddress`, `-deviceScanningEnabled`/`-setDeviceScanningEnabled:`

## 3. 機内モード

- **`SBAirplaneModeController`** — **要NSClassFromString**(`.tbd`なし)。
  - `+sharedInstance` → `id` `[@16@0:8]`
  - `-setInAirplaneMode:` → `void` `[v20@0:8B16]`
  - 注意: 機内モードをONにしても、ユーザーが過去に「機内モードでもBluetoothはONのままにする」を選んでいると、Bluetoothの電源は落ちない(上記`BluetoothManager`を別途叩く必要がある理由)。

## 4. おやすみモード(Do Not Disturb)

- **`DNDModeAssertionService`** — **要NSClassFromString**。
  - `+serviceForClientIdentifier:` → `id` `[@24@0:8@16]`(例: `@"com.apple.donotdisturb.control-center.module"`)
  - `-takeModeAssertionWithDetails:error:` → `id` `[@32@0:8@16^@24]`
  - `-invalidateAllActiveModeAssertionsWithError:` → `BOOL` `[B24@0:8^@16]`
- **`DNDModeAssertionDetails`** — **要NSClassFromString**。
  - `+userRequestedAssertionDetailsWithIdentifier:modeIdentifier:lifetime:` → `id` `[@40@0:8@16@24@32]`(例: `modeIdentifier:@"com.apple.donotdisturb.mode.default"`)
- 状態変更後は`SBQuietModeStatusChangedNotification`をpostしてUIに反映させる必要がある(実機で確認した慣習。postしないとCC上の表示が追従しないことがある)。

## 5. ハードウェアボタンの無効化(「ハイバネーション」系tweakの核)

全て`%hook`のみで使用(自前の`@interface`不要、クラスメソッド呼び出しもしないのでリンク問題なし)。全メソッド、iOS 16.7.16で現役稼働を確認済み:

| クラス | メソッド | 型 |
|---|---|---|
| `SBTapToWakeController` | `-tapToWakeDidRecognize:` | `[v24@0:8@16]` |
| | `-pencilToWakeDidRecognize:` | 同上 |
| `SBLiftToWakeController` | `-wakeGestureManager:didUpdateWakeGesture:orientation:` | `(id,long long,int)` |
| `SBSleepWakeHardwareButtonInteraction` | `-_performWake` / `-_performSleep` | `[v16@0:8]` |
| `SBLockHardwareButtonActions` | `-disallowsSinglePressForReason:` 等4種(Single/Double/Triple/Long) | `BOOL(id*)` |
| `SBHomeHardwareButton` | `-initialButtonDown:` / `-singlePressUp:` | `(id)` |
| `SBHomeHardwareButtonActions` | `-performLongPressActions` | `[v16@0:8]` |
| `SBRingerControl` | `-setRingerMuted:` | `(BOOL)` |
| `SBBacklightController` | `-turnOnScreenFullyWithBacklightSource:` | `(long long)` |
| `SBVolumeControl` | `-increaseVolume` / `-decreaseVolume` | `[v16@0:8]` |

iPhone 8 Plus(物理Home/Sleep/Ringerスイッチ搭載機)での確認。ノッチ以降の機種では`SBHomeHardwareButton*`系が無意味/別構成になっている可能性が高い(未確認)。

## 6. バッテリー残量・AC給電・通話状態

- **`SBUIController`** — **要NSClassFromString**。
  - `+sharedInstance` → `id`
  - `-updateBatteryState:` / `-ACPowerChanged` — フック対象(`%orig`後に残量/AC状態を読む起点)
  - `-batteryCapacityAsPercentage` → `int` `[i16@0:8]`
  - `-isOnAC` → `BOOL` `[B16@0:8]`
- **`TUCall`**(CoreTelephony)
  - `-status` → `int` `[i16@0:8]`。値は`TUCallStatus`enum: `1`=接続中(connected)、`6`=切断(disconnected)。長年安定している値なので信頼してよい。

## 7. フラッシュライト

- **`AVFlashlight`(AVFoundation内)— 直接リンク可**(AVFoundation.frameworkを普通にリンクしているだけで使える、`PRIVATE_FRAMEWORKS`指定も不要)。
  - `+hasFlashlight` → `BOOL` `[B16@0:8]`
  - `-isAvailable` → `BOOL` `[B16@0:8]`
  - `-flashlightLevel` → `float` `[f16@0:8]`
  - `-setFlashlightLevel:withError:` → `BOOL` `[B28@0:8f16^@20]`
  - `-init` → `id`

## 8. メディア再生制御

- **`SBMediaController`** — **要NSClassFromString**。
  - `+sharedInstance` → `id`
  - `-togglePlayPauseForEventSource:` → `BOOL` `[B24@0:8q16]`(引数は`long long`、`0`で呼べば動作確認済み)
  - `-_setNowPlayingApplication:` → `void` `[v24@0:8@16]`(フックして現在の「再生中アプリ」を追跡する用途)

## 9. App Switcher(バックグラウンドアプリの一括終了)

- **`SBMainSwitcherViewController`** / `-recentAppLayouts` / `-_deleteAppLayoutsMatchingBundleIdentifier:` — **iOS 16.7.16では不在**(iOS 13-14世代の旧API。元のPuckはこれに依存していた)。
- **`SBFluidSwitcherViewController`** — 現行のApp Switcher実装。`%hook`で使う分にはクラスメソッド呼び出しが無いのでリンク問題なし。AppGuardianプロジェクトの独自調査で先に発見・実証済みのAPI群をPuck Remakeでも再利用:
  - `-appLayouts` → `NSArray *` `[@16@0:8]`(画面外スクロール分も含めた全アプリのモデル一覧、`-visibleItemContainers`等の「画面内だけ」系アクセサより優れる)
  - `-_itemContainerForAppLayoutIfExists:` → `id` `[@24@0:8@16]`
  - `-_addVisibleItemContainerForAppLayout:reusingItemContainerIfExists:` → `void` `[v32@0:8@16@24]`(コンテナが未生成の場合に強制生成)
  - `-killContainer:forReason:` → `void` `[v32@0:8@16q24]`(`reason`は`long long`、`1`で呼べば動作確認済み)
- **`SBApplication`** — `-bundleIdentifier` → `NSString *` `[@16@0:8]`(appLayout/nowPlayingAppからbundle IDを取るための最終手段。`-itemForLayoutRole:`経由で辿るフォールバックも有効)。

## 10. 低電力アラート・通知

- **`SBLowPowerAlertItem`** — `+initialize`をhookして`%orig`を呼ばなければ、iOS標準の低電力バナー表示を抑制できる。
- **`BBServer`**(libbulletin)
  - `-publishBulletin:destinations:alwaysToLockScreen:` — **iOS 16.7.16では不在**(3引数版)。
  - `-publishBulletin:destinations:` — 存在する(2引数版のみ)。
  - 実運用では自前でBBServerを叩くより、このリポジトリの`BulletinRelay`(共有ライブラリ)経由で呼ぶほうが安全(SpringBoard以外のプロセスからも使えるキュー付き実装)。

## 11. SpringBoardの画面ロック/ウェイク操作

- **`SpringBoard`**(`UIApplication`のサブクラス、シングルトン)
  - `-_simulateLockButtonPress` → `void` `[v16@0:8]`(物理ロックボタンを押したのと同じ、トグル的に画面をロックする)
  - `-_simulateHomeButtonPress` — **iOS 16.7.16では不在**(旧式のウェイクAPI)。
  - `-_simulateHomeButtonPressWithCompletion:` → `void` `[v24@0:8@?16]` — 現行の置き換え。ブロック引数必須(`nil`ではなく`^{}`を渡す)。

## 12. アプリのフォアグラウンド検知(Eneko Remake調査より)

`SBFluidSwitcherViewController`が通常のアプリ起動ズームトランジション中にも`viewWillAppear`/`viewDidAppear`を受け取ってしまい、「本物のApp Switcherが開いた」と区別できない問題の調査で発見。

- **`SBApplicationProcessStateDidChange`**(NSNotification、確実に使える)— `note.object`は`SBApplication`インスタンス。`[app valueForKey:@"processState"]`で取れるオブジェクトの`-description`に`visibility: Foreground`/`Background`/`Unknown`/`Suspended`という文字列が含まれる。非SpringBoardのbundleIDが`Foreground`になった瞬間を捉える確実な信号として実機で複数回検証済み(`com.apple.Preferences`が前面に来た瞬間と、`SBIconController -viewWillDisappear:`が同時刻に発火することを確認)。
- **`SBFrontmostDisplayChangedNotification`** — 正しいタイミングで発火するが`object`/`userInfo`が常に空、実用不可。
- **`UIApplicationWillResignActiveNotification`** / **`UIApplicationDidBecomeActiveNotification`** — **SpringBoard自身のプロセス内では一切発火しない**ことを実機で確認済み(アプリ起動・通知センター開閉・コントロールセンター開閉の4パターンで観測回数ゼロ)。これらはウィンドウを持つ「アプリ自身のプロセス」内でのみ発火する——SpringBoard側で「他アプリが前面に来た」を検知する手段としては誤り。

## 13. Control Center

- **`CCUIToggleModule`**(ControlCenterUIKit)— **直接リンク可**。自前のCCトグルタイルを作る時の基底クラス。
  - `-iconGlyph`(アイコン画像)、`-selectedColor`、`-isSelected`、`-setSelected:`、`-refreshState`(override元の`[super refreshState]`を呼んでUIに反映)
- **`CCUILowPowerModule`** — Apple純正「Low Power Mode」トグルのPrincipalClass(`/System/Library/ControlCenter/Bundles/LowPowerModule.bundle/Info.plist`)。**SpringBoard起動直後のプロセスには存在しない**——CCSupport/Apple純正いずれのCCモジュールも、実際にControl Centerを一度開くまで遅延ロードされることを確認済み(`objc_getClass`で存在確認してもSpringBoard起動直後は`null`)。

---

## 調査に使った手法まとめ

1. `~/theos/sdks/*.sdk/System/Library/PrivateFrameworks/<Framework>.tbd`の`objc-classes:`/`exports:`で実在するシンボルを確認(推測より確実)。
2. 候補クラスはFridaの`objc_getClass("候補名")`で1つずつ存在確認(ワイルドカード列挙は絶対にしない、[frida-ios-crash-avoidance-rules.md](frida-ios-crash-avoidance-rules.md)参照)。
3. まだロードされていないframeworkは`dlopen(標準インストールパス, RTLD_NOW)`でその場でロードしてから`class_copyMethodList`+`method_getTypeEncoding`で生ダンプ(dyld共有キャッシュ内にしか実体が無くても、パス文字列さえ合っていればdlopenは成功する)。
4. 型が分かったら初めて正式な`@interface`を書いてコンパイル、`make package`のリンクエラーで「本当にリンク可能か」を最終確認。
