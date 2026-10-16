# 画面遷移ゼロでキルされたアプリを完全にバックグラウンド起動する方法

(2026-10-15, SpringBoard PID例35123→48726(respring後)。目的: ユーザーがスワイプキルしたアプリを完全に無表示で再起動し、そのアプリ自身の起動時ロジック(BGBeaconのビープタイマー)を「こっそり起こす」トリガーとして使う。スクリプトは`~/Frida/background_launch/`配下。)

## 確認済みの仕組み(最後まで動作確認済み)

- `+[FBSSystemService sharedService]`でシングルトン取得。`FBSSystemService`/`FBSOpenApplicationOptions`/`FBSOpenApplicationService`は全て`objc_getClass`で存在確認済み(ワイルドカード列挙はしない)
- `-[FBSSystemService openApplication:options:withResult:]` :: `v40@0:8@16@24@?32` — void、`NSString*`のバンドルID、`NSDictionary*`のオプション辞書、completionブロックを取る。**completionブロックに`NULL`/`nil`を渡してもクラッシュ/例外にならない**(fire-and-forget呼び出しとして安全)
- 鍵となるオプションは、推測ではなく**エクスポートされたシンボルを直接解決して**見つけた: `Module.findGlobalExportByName("FBSOpenApplicationOptionKeyActivateSuspended")`が指す`NSString*`の実際の値は`"__ActivateSuspended"`。`@{FBSOpenApplicationOptionKeyActivateSuspended: @YES}`をオプション辞書として渡すことで、起動が完全にヘッドレスになる。未検証の兄弟キーも同じ方法で解決済み: `FBSOpenApplicationOptionKeyActivateForEvent`(`"__ActivateForEvent"`)、`FBSOpenApplicationOptionKeyUnlockDevice`(`"__UnlockDevice"`)
- `FrontBoardServices`/`FrontBoard`/`BaseBoard`/`BackBoardServices`/`RunningBoardServices`は全てdyldの共有キャッシュにマージ済みでディスク上に単体バイナリが存在しない(他の調査でも繰り返し出てくる罠) — シンボルは必ず`Module.findGlobalExportByName`、クラスは`objc_getClass`で都度解決する
- 他の調査知見通り、このようなプライベートAPI呼び出しは**SpringBoard自身のメインスレッド**で行う必要がある(`performSelectorOnMainThread:withObject:waitUntilDone:`、動的登録した使い捨て`NSObject`サブクラス経由)。**今回新たに判明した罠**: この使い捨てクラスには毎回ユニークな名前を付けること(例: `Date.now()`をサフィックスに) — 同じプロセス内で以前のスクリプト実行時に既に登録済みの名前で`objc_allocateClassPair`を呼ぶと**サイレントに`NULL`が返り**、続く`class_addMethod`/`objc_registerClassPair`がその`NULL`に対して実行されてスクリプトが`Error: access violation accessing 0x0`でクラッシュする(SpringBoard自体は無事、スクリプト内の安全な失敗)

## 実際に観察された挙動(BGBeacon、1Hzビープタイマーを持つ自作テストアプリで検証)

1. **無改造のBGBeaconで最初に試行**: 画面には一切変化なし(ユーザー本人が実画面を見て確認)。ログは`didFinishLaunching`→`startBeeping`→ちょうど`beep #1`の1行だけ出て、その後何も出ない。`applicationDidEnterBackground`は一度もログされなかった(「普通にバックグラウンド化して停止した」という説明ではない)。プロセス自体は`ps`上でずっと生きていた。**結論: `ActivateSuspended`はプロセスを起動し、本物の`UIApplicationMain`/`didFinishLaunchingWithOptions`を実行させるが、その直後(ランループ1ティック分程度)でほぼ即座に凍結(サスペンド)する**、明確なライフサイクルコールバックでの説明なしに。
2. **修正**: `application:didFinishLaunchingWithOptions:`の中に、ごく普通の`[application beginBackgroundTaskWithExpirationHandler:...]`呼び出しを追加しただけで解決。実機で確認、画面には一切変化がないまま音声が実際に鳴り続け、ビープタイマーが約10秒間で10回連続発火した(通常の約30秒のバックグラウンドタスク猶予に収まるはずだが、厳密な上限までは未検証)。これは普通のApp Storeアプリが使うのと全く同じ公開APIで、プライベートAPIが必要なのは起動トリガー側だけ。

## 実用レシピ: 「キルされたアプリをこっそり起こして起動時ロジックだけ走らせる」

1. 対象アプリの`didFinishLaunchingWithOptions`内に`beginBackgroundTaskWithExpirationHandler:`呼び出しが必要(これが無いと、ランループ1ティック分だけ実行されてすぐ再サスペンドする)
2. SpringBoard自身のプロセス(実際に注入されたtweak、外部CLIではない——`FBSSystemService`はLaunchServices上のアイデンティティを持たないプロセスからの呼び出しを拒否する)から、メインスレッド上で`[[FBSSystemService sharedService] openApplication:@"バンドルID" options:@{FBSOpenApplicationOptionKeyActivateSuspended: @YES} withResult:nil]`を呼ぶ
3. ユーザーがApp Switcherでスワイプキルした後でも機能する(完全にキルされたアプリに対してテスト済み、単なるバックグラウンド化ではない)
4. 実運用のtweakでは、手順2をDarwin通知ハンドラから起動する想定(どのプロセス——rootのLaunchDaemonも含め——からでも`notify_post`できるが、実際の`openApplication:`呼び出し自体はSpringBoardの中で行う必要がある)。このセッションでは使い捨てFridaスクリプトでの検証のみで、実tweakとしてはまだ組んでいない。

## 無関係だが重要な運用上の教訓

`run_persistent.py`のPythonコントローラプロセスを(Ctrl+Cで`session.detach()`させず)生の`kill <pid>`で強制終了すると、SpringBoard側に注入されたFridaエージェントが壊れた状態になり、以降の`device.attach(pid)`が(同じPIDに限らず、新規の全ての試行で)ハングして`frida.TransportError: timeout was reached`になることがある。SSH経由の`sbreload`はこの時PID不変のまま何もエラーを返さず失敗する(同じ詰まったチャンネルの影響を受けている可能性)。**`killall SpringBoard`は確実に機能する**(PID変化を確認、直後のattachも即座に成功)。コントローラプロセスを行儀悪く終了させた後にattachが原因不明にタイムアウトし始めたら、スクリプト自体のデバッグに時間を使う前に、まず`killall SpringBoard`(その前にユーザーの画面が正常に反応しているか確認)を試す。
