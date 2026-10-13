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

## 次にやること

- [ ] 画面ロックを解除した状態でSafari等を起動し、`didAddProcess:withState:`/`executeLaunchRequest:`系が発火するか確認
- [ ] SSH経由で画面を点ける/ロックを解除する安全な方法を調査中(物理ボタン相当のHIDイベント送信など)
- [ ] `RBProcessStateChange`の`originalState`/`updatedState`の`-description`から、状態がどんな文字列で表現されるか継続確認
