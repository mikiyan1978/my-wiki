# Theos Tweak開発でハマりがちなポイントと豆知識

iPhone 8 Plus(iOS 16.7.16, palera1n rootful)でいくつものtweakを作ってきた中で、何度も踏んだ地雷と、知っていれば一瞬で解決した豆知識をまとめた。技術メモというより「次に同じ穴に落ちないための」実践ガイド。

## 1. `%hook`したプライベートクラスの前方宣言問題

Logosで`%hook SomePrivateClass`すると、コンパイラには`@class SomePrivateClass;`という前方宣言しか渡らない。これだと継承元(`UIViewController`等)のプロパティやメソッドにアクセスしようとした瞬間、こんなエラーで止まる:

```
error: property 'navigationItem' cannot be found in forward class object 'MFMailComposeController'
```

対策は、`%hook`ブロックより前に最小限のインターフェース宣言を書くだけ:

```objc
@interface MFMailComposeController : UIViewController
- (NSString *)subject;
@end
```

継承元が何かはFridaで`class_getSuperclass`を辿れば一瞬で分かる。これを知らずに「このクラスhookできない」と諦めかけたことが何度かあった。

## 2. ボタンの「中身」は、実は公開APIで覗ける

座標を当ててタップを合成するより先に試すべきなのが、`UIButton`/`UIBarButtonItem`の`target`/`action`を直接読むこと。どちらも**普通の公開プロパティ**:

```objc
UIBarButtonItem *sendItem = composeVC.navigationItem.rightBarButtonItem;
id target = sendItem.target;
SEL action = sendItem.action;
IMP imp = [target methodForSelector:action];
void (*func)(id, SEL, id) = (void (*)(id, SEL, id))imp;
func(target, action, sendItem);
```

タップ座標を合成するよりずっと確実(座標がちょっとズレたり、OSのバージョンでレイアウトが変わったりすると即座に壊れる)。`UIControl`系(ボタン全般)は`-allTargets`/`-actionsForTarget:forControlEvent:`で同じことができる。

## 3. 1つのパッケージに複数プロセス向けのdylibをまとめる

Theosの`TWEAK_NAME`には複数の名前を空白区切りで書ける。それぞれに別の`.plist`(Filter)を用意すれば、1つの`.deb`で複数のプロセスに別々のコードを注入できる:

```make
TWEAK_NAME = MyTweakCore MyTweakMail MyTweakCamera

MyTweakCore_FILES = MyTweakCore.xm
MyTweakCore_FRAMEWORKS = UIKit
# MyTweakCore.plist: { Filter = { Bundles = ( "com.apple.springboard" ); }; }

MyTweakMail_FILES = MyTweakMail.xm
MyTweakMail_FRAMEWORKS = UIKit MessageUI
# MyTweakMail.plist: { Filter = { Bundles = ( "com.apple.mobilemail" ); }; }
```

「SpringBoardでイベントを検知→別アプリを起動して別の処理をさせる」系の機能は、だいたいこの形になる。

## 4. `abort was called`を見たら「スレッドの問題」と決めつけない

Fridaで試している最中にプライベートAPIを呼んで`Error: abort was called`が出ると、つい「メインスレッドじゃないからだ」と思って`performSelectorOnMainThread:`に逃げたくなるが、**原因は2種類ある**:

- 本当にメインスレッドを要求しているだけ(UIKit系に多い)
- 呼び出しているプロセス自体にその操作のentitlement(権限)が無い

後者はスレッドをいくら正しくしても直らない。実際に「SpringBoardから直接メール送信」を試みて、メインスレッド経由でもSafe Modeクラッシュを2回連続で引き起こした実例がある——原因は十中八九entitlementの壁で、Mail.app自身(正しい権限を持つプロセス)を経由する方式に戻してようやく安定した。**スレッドを変えても症状が変わらないことまで確認してから、次の仮説に進む。**

## 5. `dispatch_async_f` + `NativeCallback`はFridaだけの地雷

Frida上での実験で、GCDのコールバックトランポリン(`dispatch_async_f`)とFridaの`NativeCallback`を組み合わせると、このデバイス/iOSバージョンの組み合わせでは同じ無効アドレスで繰り返しクラッシュした。**ただしこれはFrida特有の問題。** 実際にコンパイルしたtweakの中で普通に`dispatch_async(dispatch_get_main_queue(), ^{ ... })`を使う分には、本物のGCD+本物のObjective-Cブロックなので全く問題ない。Frida上の実験で詰まったからといって、同じ処理を実tweakに移植する時まで怖がる必要はない。

## 6. PSListControllerの行が表示されない時、まず疑うべき1行

Settings.appの設定画面で、`-specifiers`をオーバーライドしたのに画面が真っ白(ナビバーだけ、行が1つも出ない)になったら、9割方これが原因:

```objc
- (NSArray *)specifiers {
    NSArray *result = [self buildMySpecifiers];
    return result; // ← これだけだと表示されない!
}
```

テーブルのデータソースは`-specifiers`の戻り値ではなく、継承元の`_specifiers`という**ivar**を直接読む。必ず明示的に代入する:

```objc
- (NSArray *)specifiers {
    NSArray *result = [self buildMySpecifiers];
    _specifiers = [result mutableCopy]; // これが無いと詰む
    return _specifiers;
}
```

## 7. `self.view`が嘘をつくことがある

`PSListController`は`view`/`table`/`tableView`という3つの似たプロパティを持つが、ビルドによっては`self.view`が実行時には`UITableView`ではない(ただの`UIView`ラッパー)。ヘッダー的には同じ型に見えるのでキャストは普通にコンパイルが通り、**実行時に何も起きない**(クラッシュも警告もない)という一番厄介なバグになる。非自明なことをする前に`NSStringFromClass([self.view class])`を一度ログして確認する癖をつけると良い。怪しかったら`self.table`を使う。

## 8. 「車輪の再発明」の前に5分だけ検索する

プライベートなミューテーション系APIをゼロから逆解析しようとして2回クラッシュさせた後、たった5分のGitHub検索で、同じ問題をとっくに解決している既存tweak(しかも数年前のもの)が見つかったことがある。しかもその解決策は「ミューテーションAPIを呼ぶ」のではなく「**クエリメソッドの戻り値を書き換えるだけ**」という、根本的に安全な別解だった。プライベートAPIを深追いする前に、同じジャンルのtweakのソースを一度は検索する価値がある。

## 9. クラッシュしたらまずログを読む、推測しない

`/var/mobile/Library/Logs/CrashReporter/*.ips`は`.ips`拡張子だが中身はJSON。Pythonの`json.loads`で読めば、例外種別(`EXC_BAD_ACCESS`/SIGSEGVなど)、フォルトアドレス、シンボル化済みのスタックトレースが一発で分かる。「たぶんこれが原因だろう」で次の手を打つ前に、まずこのログを読む方が結局早い。

## 10. Respring後はPIDが変わる、uptimeは変わらない

`sbreload`/`launchctl kickstart -k`等でSpringBoardを再起動すると、新しいPIDが振られる(Frida再アタッチが必要)が、`uptime`は変わらない(デバイス自体は再起動していない)。逆に`uptime`がリセットされていたら、それは本当の電源断/フルリブート——半有線(semi-tethered)系のjailbreakなら、USB再接続での再ジェイルブレイクが必要になる、もっと重い事態のサイン。この2つを見分けるだけで「軽微な respring」か「物理アクセスが要る重大事態」かが一瞬で判別できる。

## 11. ROOTFUL=1を忘れると、tweakだけでなく他のtweakまで壊れる

`ROOTFUL ?= 0`をデフォルトにしているMakefileで`ROOTFUL=1`を付け忘れてビルドすると、`THEOS_PACKAGE_SCHEME = rootless`になり、rootfulデバイスなのに`/var/jb/...`以下にインストールされてしまう。これ自体は単にそのtweakが動かないだけで済みそうに見えるが、実際には**他の無関係なtweakまで巻き添えで壊れる**ことがある。実例: `[NSFileManager fileExistsAtPath:@"/var/jb"]`だけでrootless/rootfulを判定している別のtweak(アプリ管理画面)が、この誤インストールで作られた`/var/jb`ディレクトリの存在だけを見て「この端末はrootless」と誤判定し、パッケージ一覧が全部空になった。

直し方は該当パッケージを`dpkg --remove`して`/var/jb`を消し、`ROOTFUL=1`を付けて再ビルドするだけだが、**「このtweakは壊れていないのに様子がおかしい」時は、直前に別のtweakを誤った設定でインストールしていないかをまず疑う**。

## 12. CydiaSubstrate + clang modulesが衝突して"cyclic dependency"で全滅する

クリーンビルド(`make clean && make package`)で、それまで問題なかったプロジェクトが急に

```
cyclic dependency in module 'Darwin': Darwin -> Foundation -> CoreFoundation -> Darwin
```

で止まることがある。CydiaSubstrate.frameworkのmodulemapがclang modulesと相性が悪いのが原因。Makefileに1行足すだけで直る:

```make
MyTweak_USE_MODULES = 0
```

## 13. `+sharedInstance`のようなクラスメソッド呼び出しだけ、リンクエラーになる

プライベートクラスに自分で`@interface`を書いて`%hook`以外の場所で普通に`[Foo sharedInstance]`のようなブラケット呼び出しを書くと、コンパイルは通るのに**リンクで**こう落ちることがある:

```
ld: Undefined symbols:
  _OBJC_CLASS_$_Foo, referenced from: ...
```

ポイントは「インスタンスメソッド呼び出し」と「クラスメソッド呼び出し」でリンカの要求が違うこと。`id`型の変数に対して`-someInstanceMethod`を呼ぶだけなら、実行時にobjc_msgSendがセレクタで解決するのでリンク時のClassシンボルは不要——だが`[Foo someClassMethod]`や`[[Foo alloc] init]`のように**クラスそのもの**を参照する式は、リンカが`_OBJC_CLASS_$_Foo`という実体シンボルを要求する。独自に書いた`@interface`は型チェックを通すだけで、このシンボルは生成してくれない。

対策は2つ:
- そのプライベートクラスが属するframeworkの`.tbd`が`~/theos/sdks/*.sdk/System/Library/PrivateFrameworks/`以下に存在するなら、Makefileの`_PRIVATE_FRAMEWORKS`に足して普通にリンクする(次項参照)。
- `.tbd`が無い/見つからない完全に内部的なクラス(SpringBoard内のSB*系クラスなど)なら、`NSClassFromString(@"Foo")`で実行時にClassを取得し、`objc_msgSend`に関数ポインタとして直接キャストして呼ぶ(Logosの`.x`ファイル内なら`%c(Foo)`マクロで同じことができる)。

## 14. 「本当に存在するprivate API」を当て推量する前に`.tbd`を見る

プライベートAPIの実在するクラス名・メソッド名を知りたい時、SNSやブログの情報は古いiOSバージョンのものだったりして当てにならない。もっと確実なのは2段構え:

1. まず`~/theos/sdks/*.sdk/System/Library/PrivateFrameworks/`以下を`grep -l`で検索し、`<Framework>.tbd`の`objc-classes:`を見る。これはApple自身がそのSDKバージョン向けに吐き出した**実在するシンボルのリスト**なので、ここに載っているクラス名は確実にリンク可能。
2. `.tbd`にはクラス名・ivar名はあってもメソッドのシグネチャまでは載っていない。実際の引数/戻り値の型を知るには、該当frameworkがまだロードされていないプロセスでもFridaから`dlopen(path, RTLD_NOW)`すればその場でロードでき、`class_copyMethodList`+`method_getTypeEncoding`でメソッド一覧と型を生でダンプできる(標準インストールパス文字列をそのまま渡せば、dyld共有キャッシュ内にしか実体が無くてもdlopenは成功する——`ls`や`scp`でそのパスにファイルが見当たらなくても気にしなくていい)。

この2段構えで、「このバージョンのiOSで本当に使えるAPIか」を当て推量なしに確認してからコードを書ける。

## 15. Filter plistを忘れると"You are missing a filter property list"

`INSTALL_TARGET_PROCESSES = SpringBoard`をMakefileに書いても、それだけでは足りない。Tweak側に`<TweakName>.plist`(または`Filter.plist`)を置いて、実際にどのバンドルIDに注入するかを明示する必要がある:

```
{ Filter = { Bundles = ( "com.apple.springboard" ); }; }
```

これが無いと`make package`の最終ステージ(stage)で初めてエラーになる——コンパイル・リンクは全部通った後なので、原因が分かりにくい。

## 16. `-Werror`配下だと`keyWindow`のような軽微な非推奨APIも即ビルドエラーになる

`UIApplication.keyWindow`のような非推奨(deprecated)だが実用上問題ないAPIが、プロジェクトの警告設定次第ではただの警告ではなく**ビルドエラー**になることがある:

```
error: 'keyWindow' is deprecated: ... [-Werror,-Wdeprecated-declarations]
```

その1箇所だけを許可したい時は、ファイル全体やMakefileの警告設定を緩めるのではなく、該当行だけをpragmaで囲むのが一番安全:

```objc
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
UIViewController *presenter = [UIApplication sharedApplication].keyWindow.rootViewController;
#pragma clang diagnostic pop
```

## 17. 別プロセス(SpringBoard以外)へのFridaアタッチは、PIDの"鮮度"を疑う

SpringBoard以外のアプリ(Preferences.appなど)にFridaで`device.attach(pid)`しようとして`frida.TransportError: timeout was reached`が繰り返し出る時、frida-server自体やネットワークは正常(SpringBoardには同時に問題なくアタッチできる)でも、**そのPIDが長時間バックグラウンドでサスペンドされ続けている**と接続が通らないことがある。`kill -9`でそのプロセスを一度殺し、`uiopen`等で再度フォアグラウンドに呼び出して新しいPIDを取り直してからアタッチすると直ることが多い。

また、`_UICreateScreenUIImage()`によるスクリーンショット撮影はSpringBoardプロセスから呼ぶ前提の実装で、他アプリのプロセス内から呼ぶと`Error: abort was called`になる。他アプリの画面を確認したい時はスクリーンショットではなく、見たいクラスの該当メソッド(例: `PSListController`の`-specifiers`)を直接呼んで戻り値を検証する方が、確実で速い。

## 18. `objc_msgSend`をNativeFunctionとして使い回すと引数個数エラーになる

Frida上で`objc_msgSend`を`NativeFunction`化する時、`["pointer","pointer"]`(self, cmd)のような2引数版を1つ作って、それを3引数・4引数の呼び出しにも使い回すと

```
Error: bad argument count
```

になる。FridaのNativeFunctionは宣言した引数の型と数に厳密なので、引数の数(self, cmd, arg1, arg2, ...)ごとに別々の`NativeFunction`インスタンスを用意する必要がある。

## 19. 似た機能を持つ既存tweakを置き換える時は`Conflicts:`を書く

ハードウェアボタンの無効化やApp Switcherフックなど、同じ種類のフックを行う独自tweakを作って元のtweakと**共存**させてしまうと、お互いの内部状態(「今まさに自分がアクティブか」等のフラグ)が食い違って、片方だけが復帰してもう片方はロックしたまま、のような中途半端な壊れ方をすることがある。元のtweakを置き換える・リコードする時は、`control`ファイルに

```
Conflicts: com.original.tweak.bundleid
```

を書いておく。`dpkg -i`が自動で「先に元のtweakを消してから」を強制してくれるので、共存によるフックの競合がそもそも起こり得なくなる。

---

## 秘技奥義編

ここから先は「知らないと一生ハマる、知っていれば一瞬」の類の、もう一段深い小技集。

### 奥義1: 全クラス列挙は「やった瞬間に死ぬ」

Fridaでこのデバイスの未知のprivateクラスを漁りたくなった時、一番やってはいけないのが`ApiResolver("objc")`のワイルドカード検索や、`objc_copyClassList`で取れる全クラスを`class_getName`で回すような**全列挙**。iOS 16.7.16のSwiftランタイムは、あるジェネリックなSwiftクラスにぶつかると内部のデマングラーが`abort()`を呼ぶ既知の地雷があり、これはJS側で`try/catch`しても**キャッチできない**(プロセスごと落ちる)。対象がSpringBoardならそのままrespring行き。

秘訣は、見たいクラス名の**候補**を`objc_getClass("候補名")`で1つずつ当てる(存在しなければ単に`null`が返るだけで絶対に落ちない)。本物のクラスダンプが欲しい時も、「全クラスを無差別に舐める」のではなく「存在が確認できたクラスだけ`class_copyMethodList`する」の2段構えに必ず分ける。

### 奥義2: `frida.spawn()`は使うな、既に起きているプロセスに忍び寄れ

このデバイス+frida-serverの組み合わせでは、`frida.spawn()`で起動したアプリはiOSのwatchdogに「5秒以内に正常終了しなかった」と見なされて強制終了される。**新規spawnは使わず**、人間(または`uiopen`)に先にアプリを起動させてから`ps aux`でPIDを取り、`device.attach(pid)`で後から忍び寄るのが唯一安定する流儀。

### 奥義3: `script.unload()`も使うな、フックは外さず生かしたまま放置が正義

一度`Interceptor.attach`したフックを安全に外す方法は、実はこのデバイスには存在しないに等しい。`script.unload()`を呼ぶと、別スレッドがちょうどフック済み関数の中にいるタイミングでSIGBUSを引く実例が複数回確認されている。対策は「外さない」——フックしたまま対象プロセスの寿命が尽きるまで放置し、追加の観測が必要になったら**別の独立したセッションを同じPIDに新しく貼る**(フックは積み重なるだけで、古いものを剥がす必要はない)。

### 奥義4: frida-serverは起動していても「聞いていない」ことがある

`ps aux | grep frida-server`でプロセスが生きているのを確認しても、Mac側から`device_manager.add_remote_device()`で延々タイムアウトすることがある。原因はlaunchd管理のfrida-serverがデフォルトで**loopback(127.0.0.1)にしかbindしない**こと。リモート接続を受けるには明示的にbindアドレスを指定して立て直す必要がある:

```bash
launchctl unload /Library/LaunchDaemons/re.frida.server.plist
pkill frida-server
nohup /usr/sbin/frida-server -l 0.0.0.0:27042 -D > /var/mobile/frida_server.log 2>&1 &
```

これはrespringでは消えないが、**端末の再起動で毎回元に戻る**ので、セッションの頭で必ず確認する癖をつけると無駄なタイムアウト地獄を避けられる。

### 奥義5: 他のtweakが`%new`で足したメソッドは、お前のtweakからも見える

同じプロセス(SpringBoard)に複数のtweakが注入されている時、Objective-Cランタイムは**プロセス全体で1つ**しかない。つまり、あるtweakがLogosの`%new`で既存クラスにメソッドを足すと、その瞬間から**他の全てのtweak(自分のtweakも含む)から**、まるで元から生えていたかのようにそのメソッドを呼べるようになる。実際、SpringBoardのメソッド一覧をFridaでダンプした時に見覚えのない`puckWake`/`puckLightWake`のようなメソッド名が混ざっていて二度見した実例がある——正体は、過去に注入された**別の(今は入れ替えた)tweakが足した`%new`メソッドの残骸**だった。

この性質は諸刃の剣: 自分のtweakを他のtweakに依存させずに独立させたいなら、相手の`%new`メソッドをあてにした実装はしない(相手がアンインストールされた瞬間に消える)。逆に「このtweakが入っていれば、その`%new`メソッドを呼んで機能を借りる」という疎結合連携(`NSClassFromString`+`respondsToSelector:`+`objc_msgSend`)は、公式の共有ライブラリを用意するほどではない軽い連携に便利。
