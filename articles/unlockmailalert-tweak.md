# UnlockMailAlert — ロック解除の度に自動で自分撮り+メール通知するtweak

[ios-touchless-wake-unlock.md](ios-touchless-wake-unlock.md)と[ios-touchless-app-control.md](ios-touchless-app-control.md)で培った知見を、Frida/SSHなしで**端末単体だけで動く実際のJailbreak tweak**に落とし込んだもの。「ロックを解除する度に、前面カメラで撮影し、解除があった旨をメールで通知する」という簡易な盗難・不正操作検知機能。

リポジトリ: `~/iOSTweaks/UnlockMailAlert`(`com.mikiyan1978.unlockmailalert`)

## 構成(3つのdylib、1パッケージ)

Theosの`TWEAK_NAME = A B C`方式で、ターゲットプロセスが異なる3つのdylibを1つの`.deb`にまとめている。

| ターゲット | 対象プロセス | 役割 |
|---|---|---|
| `UnlockMailAlertCore` | `com.apple.springboard` | `-[SBLockScreenManager lockScreenViewControllerDidDismiss]`をフックし、解除の度にCamera.app→(待機)→Mail.appの順に起動する |
| `UnlockMailAlertCamera` | `com.apple.camera` | Camera.appの起動を検知し、前面カメラに切り替えてシャッターを押す(タップではなく実ボタンのtarget/actionを直接呼ぶ) |
| `UnlockMailAlertMail` | `com.apple.mobilemail` | 特定の件名マーカーを持つコンポーズ画面を検知し、Sendボタンのtarget/actionを直接呼んで自動送信する |

`.plist`の`Filter.Bundles`でそれぞれ別プロセスに注入されるよう振り分けている。

## 全体の流れ

```
解除(swipe等) → SBLockScreenManager -lockScreenViewControllerDidDismiss 発火
  → [UnlockMailAlertCore] LSApplicationWorkspace openApplicationWithBundleID: "com.apple.camera"
      → [UnlockMailAlertCamera] -viewDidAppear: で待機後、
          CAMFlipButtonのtarget/action(-_handleFlipButtonReleased:)を直接呼んで前面カメラに切替
          CUShutterButtonのtarget/action(-_handleShutterButtonReleased:)を直接呼んで撮影
  → (4.5秒後) [UnlockMailAlertCore] LSApplicationWorkspace openURL: mailto:...(件名に目印付き)
      → [UnlockMailAlertMail] -viewDidAppear: で件名の目印を確認し、
          Sendボタンのtarget/actionを直接呼んで自動送信
```

タイミングはすべて固定ディレイ(`dispatch_after`)で組んでいる。Camera.appの起動〜前面切替〜撮影〜ファイル書き込みに十分な余裕(0.8秒+1.3秒程度)を見込んだ後、Mail.appを起動している。

## 鍵となる実装パターン

### 1. ボタンの実体(target/action)を直接呼ぶ、合成タップは使わない

`UIButton`/`UIBarButtonItem`はどちらも`target`/`action`(`UIBarButtonItem`は両方とも**公開プロパティ**)を持つので、座標を当てずに直接呼び出せる。

```objc
UIBarButtonItem *sendItem = composeVC.navigationItem.rightBarButtonItem;
id target = sendItem.target;
SEL action = sendItem.action;
IMP imp = [target methodForSelector:action];
void (*func)(id, SEL, id) = (void (*)(id, SEL, id))imp;
func(target, action, sendItem);
```

Camera.appの`CUShutterButton`/`CAMFlipButton`は`UIControl`なので`-allTargets`/`-actionsForTarget:forControlEvent:`(`UIControlEventTouchUpInside = 1<<6 = 64`)でtarget/actionを調べる必要がある(開発時はFridaで調査、tweak本体は調査済みのセレクタ名を直接呼ぶだけで良い)。

### 2. 件名マーカーで「自分が起動したコンポーズか」を判定する

`UnlockMailAlertMail`は**全てのMailコンポーズ画面**に対して`-viewDidAppear:`が発火するので、`"[UnlockMailAlert] "`という目印を件名の先頭に必ず付け、それが無ければ何もしない。本人が手動で書いているメールを誤って自動送信してしまわないためのガード。`objc_setAssociatedObject`で「このセッションは既に処理済み」のフラグも持たせ、二重送信も防ぐ。

### 3. Logosの前方宣言だけでは`.navigationItem`等が解決できない

`%hook ClassName`で未知のプライベートクラスをフックすると、Logosは`@class ClassName;`という前方宣言しか生成しない。継承元の型(例: `UIViewController`)のプロパティ/メソッドを使いたい場合、`%hook`より前に最小限のインターフェース宣言を書く必要がある:

```objc
@interface MFMailComposeController : UIViewController
- (NSString *)subject;
@end
```

継承チェーンはFridaの`class_getSuperclass`ループで事前に確認しておく(`MFMailComposeController -> UIViewController -> NSObject`、`CAMViewfinderViewController`も同様)。

## ハマった点・やらないと決めたこと

### 写真添付は実装していない(危険なので見送り)

`MFMailComposeViewController`の公開API`addAttachmentData:mimeType:fileName:`は、実際に使われている内部クラス`MFMailComposeController`には**存在しない**(Frida上で`respondsToSelector:`が`NO`を返すことを確認済み)。コンパイル済みコードでこれを直接呼ぶと「認識されないセレクタ」で確実にクラッシュする。本当の添付用プライベートAPI(`_delivery`ivarの`MFOutgoingMessageDelivery`経由と思われる)は未解決。現状は「撮影はするが、メールには添付せず本文で通知するだけ」という安全側の実装に留めている。

### SpringBoardから直接メール送信(`MFOutgoingMessageDelivery`経由)はSafe Modeクラッシュで断念

Mail.appを起動せず`EmailCore`/`MailServices`の内部クラス(`MFMutableMessageHeaders`、`MFAccountStore`、`MFOutgoingMessageDelivery`)を使って**SpringBoardプロセスから直接**メールを送信しようとしたところ、`-setAccount:`/`-deliverSynchronouslyWithCompletion:`の呼び出しで**2回連続してSpringBoardがクラッシュしSafe Modeに突入**した(どちらもセーフモード復旧スクリプト[ios-touchless-wake-unlock.md](ios-touchless-wake-unlock.md)で復旧)。

Frida上でも同じ2つのメソッドは直接呼び出すと`abort`していたが、他のUIKit系メソッド(`send:`等)がFridaのスレッド問題で`abort`していたのと混同し、「メインスレッド経由にすれば直るはず」と判断してtweak化したのが間違いだった。実際はおそらく**SpringBoardプロセス自体にメール送信のentitlementが無い**ことが原因で、スレッドを正しくしても解決しない種類の失敗だったと考えられる。

教訓: Frida上での`abort`は「スレッドの問題」と「entitlement/権限の問題」のどちらでも同じように見えるので、安易に同一視して対策を決め打ちしない方が良い。コンパイル済みコードで試す前に、Frida上で**スレッドを変えても症状が変わらないことまで確認してから**次の手(entitlementの壁と判断する等)に進むべきだった。

### 生SMTP直接送信は保留(アプリ用パスワードが必要なため)

上記の代替として「Appleのメール機構を一切使わず、生のSMTP接続を自前で実装する」方針も検討した(`NSStream`+TLS、ポート465の暗黙TLSを想定)。これなら entitlement の壁を完全に回避できるが、認証に**アプリ専用パスワード**が必要で、これはユーザー本人が発行しSettings.appの専用設定画面から入力する必要がある(本人のメインパスワードを使うのは、2段階認証環境ではほぼ確実に失敗する上、万一保存する場合のセキュリティリスクも大きいため却下)。現状はアプリ用パスワード未発行のため保留中。

## 現状の到達点

- ロック解除の度に自動でCamera.appが起動し、前面カメラでの撮影が行われる(確認: `/var/mobile/Media/DCIM/`に新規ファイルが生成される)
- 解除の度に自動でMail.appが起動し、タップ不要で実際にメールが送信される(確認: Gmail受信箱に到達)
- 一連の流れは完全にtweak単体で完結し、SSH/Fridaは不要
- 写真はメールに添付されず、撮影の事実のみ本文に記載される(添付は次の課題)
