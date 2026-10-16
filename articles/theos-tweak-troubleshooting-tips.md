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
