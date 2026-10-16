# iOS + Fridaでのクラッシュ回避ルール集(早見表)

iPhone 8 Plus(iOS 16.7.16, palera1n rootful)でのFrida調査・tweak開発を通じて積み上がった、どのプロジェクトにも共通する安全ルール。新しいFridaスクリプトを書く前に必ず目を通す。個別の発見の詳細は各記事([springboard-icon-visibility-force-touch.md](springboard-icon-visibility-force-touch.md)、[springboard-pip-internals.md](springboard-pip-internals.md)、[fastfreeze-background-app-killing.md](fastfreeze-background-app-killing.md)、[ios-touchless-wake-unlock.md](ios-touchless-wake-unlock.md)、[headless-app-launch-fbssystemservice.md](headless-app-launch-fbssystemservice.md))を参照。

## ハードルール(どのスクリプトにも適用)

1. **全ObjCクラスのワイルドカード列挙は絶対にしない**(`ApiResolver("objc")`の`*`、`objc_copyClassList`の全走査)。iOS 16.7.16のSwiftランタイムは、一部のジェネリックSwiftクラスで内部デマングラーのアサーションに当たり`abort()`(SIGABRT、JSから捕捉不可)する。対象プロセス全体がクラッシュする(SpringBoardならrespring)。安全な代替: `objc_getClass("候補名")`で1つずつ存在確認(純粋なルックアップなので、存在しない名前でも絶対にクラッシュしない、nullが返るだけ)、存在確認できたクラスにだけ`class_copyMethodList`/`class_copyIvarList`を使う。
2. **Frida 17.xにはグローバルな`ObjC`ブリッジが無い。** 生のObjCランタイムC APIを`NativeFunction`+`Module.findGlobalExportByName`で呼ぶ: `objc_getClass`, `sel_registerName`, `sel_getName`, `class_getInstanceMethod`, `class_getSuperclass`, `class_getName`, `method_getName`, `method_getTypeEncoding`, `class_copyMethodList`, `class_copyIvarList`, `ivar_getName`, `ivar_getTypeEncoding`, `object_getClass`, `free`。`Module.findExportByName(module, name)`(2引数版)はFrida 17で削除済み——`Module.findGlobalExportByName(name)`か`Process.getModuleByName(module).findExportByName(name)`を使う。
3. **この端末では`frida.spawn()`を絶対に使わない。** 新規spawnされたアプリはiOSウォッチドッグに約5秒で強制終了される("failed to terminate gracefully after 5.0s")——このjailbreak/frida-serverの組み合わせではspawn時のゲーティングが不安定。必ずユーザーに手動でアプリを起動してもらい、PIDを`ps aux | grep -i <name>`で見つけて`device.attach(pid)`する。
4. **`script.unload()`を絶対に呼ばない。** 別スレッドがフック中の関数呼び出し中にインターセプターをアンロードすると、SIGBUS(KERN_PROTECTION_FAILURE)でクラッシュしたことがある(SpringBoard・Spotify両方で確認)。安全なパターン: 一度attachしたら、プロセスの寿命が尽きるまでフックを張ったままにする(`run_persistent.py`——フォアグラウンド常駐、`signal.pause()`、Ctrl+Cで`session.detach()`のみ、unloadは呼ばない)。追加の計装が必要になったら、同じPIDに**2つ目の独立したセッション**をattachする(1つ目をリロードしない)。`run_attach.py`(N秒後に`unload`+`detach`するタイマー式)は**`Interceptor.attach`を一切呼ばない純粋な読み取り専用の一発ものスクリプトにのみ**安全——クラス/メソッド/ivarの一発ダンプ等。`Interceptor.attach`を含むスクリプトは必ず`run_persistent.py`を使う。
5. SpringBoard特有の注意: **読み取り専用の観察フックのみ**(enter/leaveでログするだけ、`retval.replace(...)`や自前のライブAPI呼び出しは絶対にしない)——OS自身のUIが実際の操作で渡す本物の引数値を既に観察済みで、意味論に確信がある場合を除く。間違った書き込みはSpringBoardをクラッシュさせるだけでなく(軽微)、**永続化されたホーム画面アイコンレイアウトを破壊しうる**(一時的なビューフレーム崩れよりずっと復旧が困難)。モデルを変更するプライベートメソッドを勝手読みで呼ばない——先に観察、確信が持ててから変更する。
6. Mac側の`run_in_background`なBashコントローラーは、対象デバイス側プロセスが元気に動き続けていても、コーディングセッション自身のシェルコンテキストがクリーンアップされる際に刈り取られることがある。`&`単体ではなく`nohup ... & disown`を使う(macOSには`setsid`が無い)。
7. クラスダンプの出力先はデバイス上の`/var/mobile/Class Dump/<AppName>/`(バンドルIDではなく表示名)、`/tmp/`は使わない(再起動で消える)。対象アプリは通常`mobile`で動くので先に`mkdir -p`+`chown mobile:mobile`。
8. **特定アプリ向けの新しいtweakに着手する前に**、`/var/mobile/Class Dump/<AppName>/`とこの種の記事を先に確認してから、プライベートなクラス/セレクタ名を記憶や公開class-dumpリポジトリから推測する。

## 「呼ぶ前に確認する」系の教訓

- **公開class-dumpや既存の公開tweakは、実機の現在のOSバージョンと全く違う(古い)リビジョンのプライベートフレームワークを前提にしていることがある**——同じクラス名でも別のメソッドセット。`%hook`で存在しないセレクタをフックしてもLogosは無音で失敗する(クラッシュも警告もない)。新しいtweakの土台にする前に、必ず対象デバイス上でそのクラスの現在のメソッド一覧を自分でダンプする。
- **ミューテーション系(状態を変更する)プライベートAPIを推測で直接呼ばない。** 先に広範囲の読み取り専用フックを仕掛け、既に安全と分かっている入力(HIDイベント等)を実際に送り、どの経路が本当に呼ばれるかを観察する方がはるかに安全。
- **Frida上での`abort was called`は「スレッドの問題」とは限らない。** メインスレッドで呼べば直ることもあれば(UIKit系の多く)、呼び出し元のプロセスにそもそもその操作のentitlementが無いことが原因のこともある(例: SpringBoardから直接メール送信を試みて2回Safe Modeクラッシュした件、[unlockmailalert-tweak.md](unlockmailalert-tweak.md)参照)。スレッドを変えても症状が変わらないことまで確認してから、次の仮説(entitlementの壁等)に進む。
- **同じ問題を解決済みの既存tweakが無いか、ゼロから逆解析する前に5分だけ検索する。**(Hid3xの例、[springboard-icon-visibility-force-touch.md](springboard-icon-visibility-force-touch.md)参照)
- **アクセサ名は中身を保証しない。** `currentProcess`/`frontmostX`のような名前のメソッドが実際に「今画面に見えているアプリ」を返すとは限らない([fastfreeze-background-app-killing.md](fastfreeze-background-app-killing.md)の`-currentProcess`誤判定の例)。
- **ミューテーション操作の検証は、否定ケース1回では不十分。** 持続的な肯定ケース(本当に数分間フォアグラウンドのまま、等)も試してから信頼する。

## Logosでプライベートクラスをフックする時のコンパイルエラー対策

`%hook ClassName`は未知のプライベートクラスに対して`@class ClassName;`という前方宣言しか生成しない。継承元の型(`UIViewController`等)のプロパティ/メソッドを使いたい場合、`%hook`より前に最小限のインターフェース宣言を書く必要がある:
```objc
@interface SomePrivateClass : UIViewController
- (NSString *)someMethod;
@end
```
継承チェーンは`class_getSuperclass`をループで辿って事前にFridaで確認しておく。

## メインスレッドへの安全な乗せ方

- **`dispatch_async_f` + `NativeCallback`は、このFrida 17.19.0 + arm64 + iOS 16.7.16の組み合わせで破損/危険と判明済み**(同一の無効アドレスで繰り返しSpringBoardをクラッシュさせた実績あり)。**Frida上の実験では絶対に使わない。**
- Frida上の実験で、1引数のセレクタをメインスレッドに安全に乗せるには`-[NSObject performSelectorOnMainThread:withObject:waitUntilDone:]`。多引数のセレクタは`NSInvocation`を組み立てて、それ自体の`invoke`(引数0個)を`performSelectorOnMainThread:withObject:waitUntilDone:`経由で呼ぶ。
- **ただしこれはFrida特有の回避策であり、実際にコンパイルされたtweakコードでは不要なことが多い。** 本物の`dispatch_async(dispatch_get_main_queue(), ^{ ... })`と本物のObjective-Cブロックは、GCDの生コールバックトランポリンを経由するFridaの`NativeCallback`とは全く別物で、通常通り安全に動く([unlockmailalert-tweak.md](unlockmailalert-tweak.md)で実証済み)。
