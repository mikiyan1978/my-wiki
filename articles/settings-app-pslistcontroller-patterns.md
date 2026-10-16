# Settings.app設定画面(PreferenceLoader + PSListController)構築の知見集

AppGuardian/AutoClean等の実プロジェクトで積み上げた、Preferences.framework/PSListControllerでのSettings.appペイン構築パターン集。危険なプライベートAPI1つと、それぞれ安全な代替2パターン、よくあるハマりどころを収録。

## 0. 基本的なハマりどころ

- **AltList(`com.opa334.altlist`)はこの端末でPreferenceLoaderから自動ロードされない** — `__attribute__((constructor))`内で`dlopen("/Library/Frameworks/AltList.framework/AltList", RTLD_NOW)`しないと`ATLApplicationListMultiSelectionController`が`NSClassFromString`で解決できない。これをやらないと、それを指す`PSLinkListCell`/`detail`行は**タップしても何も起きない**だけ(クラッシュなし、原因が分かりにくい)。
- **`-[PSListController specifiers]`の罠**: テーブルビューのデータソースは、`-specifiers`の戻り値ではなく**継承元の`_specifiers`ivarを直接読む**。新しく組み立てた配列を`return`するだけでivarに代入し忘れると、空白画面(ナビバーのみ、行なし、クラッシュなし)になる——このコードベースで最も頻出するバグ。必ず同じ処理の中でivarにも代入する:
  ```objc
  - (NSArray *)specifiers {
      @try {
          NSArray *result = [self buildSpecifiers];
          _specifiers = [result mutableCopy];
      } @catch (NSException *exception) {
          _specifiers = [@[errorRowSpecifier] mutableCopy];
      }
      return _specifiers;
  }
  ```
- `ATLApplicationListControllerBase`ベースの詳細画面をpopすると、Apple純正の`-[PSListController clearCache]`内部でEXC_BAD_ACCESS/SIGBUSすることがある(pop後の遅延dealloc中)。確認済みの回避策: `-[PSListController clearCache]`をグローバルにswizzleし、`ATLApplicationListControllerBase`のサブクラスに対してだけno-opにする(アイコンキャッシュのクリアだけなので無害)。

## 1. 「複数の値から1つ選ぶ」UI: 危険なAPI1つと、安全な代替2つ

**この端末でタップ時に確実にPreferences.appをクラッシュさせる**:
```xml
<dict>
    <key>cell</key><string>PSLinkListCell</string>
    <key>detail</key><string>PSListItemsController</string>
    ...
</dict>
```
`PSListItemsController`はApple純正の汎用「Titles/Valuesから1つ選ぶ」詳細コントローラーだが、この端末のこのiOSビルドではタップした瞬間にクラッシュする。`PSMultiValueSpecifier`(Settings.bundle方式の規約で、Preferences.framework/PSListControllerの世界とは別物)はセルは描画されるが**タップしても一切反応しない**(クラッシュではないが、使い物にならない)。どちらも使わない。

**安全パターンA——`UIAlertController`のアクションシート(選択肢が少数固定の場合)**: `PSButtonCell`行のアクションメソッドで`UIAlertControllerStyleActionSheet`を出し、選択肢ごとに`UIAlertAction`を1つ、ハンドラ内で`CFPreferencesSetValue`+`CFPreferencesAppSynchronize`で書き込む。PSListControllerのpush/popを一切使わないので、上記のクラッシュ系統には触れない。10個未満の選択肢で、ネイティブなドリルダウン画面が不要な場合に向く。

**安全パターンB——独自の`PSListController`サブクラス(選択肢が多い/動的/行ごとに独自の挙動が必要な場合)**: 本物の`.h`/`.m`サブクラス(独自ファイル、Makefileの`_FILES`に追加)を、
```xml
<dict>
    <key>cell</key><string>PSLinkListCell</string>
    <key>detail</key><string>YourClassName</string>
</dict>
```
で push する。選択肢を実行時にスキャンする場合(ディレクトリ内の`.caf`ファイル一覧等)や、タップ時に単に値を書くだけでなく独自動作(音をプレビュー再生する等)が必要な場合に最適。

## 2. `PSSpecifier`で行を組み立てる

```objc
PSSpecifier *row = [PSSpecifier preferenceSpecifierNamed:title target:self set:nil get:nil
                                                    detail:nil cell:PSButtonCell edit:nil];
row.buttonAction = @selector(myRowTapped:); // setProperty:forKey:@"action" ではない!
```
`PSButtonCell`の実際のアクションプロパティは型付きの`buttonAction`(SEL)であり、`loadSpecifiersFromPlistName:`が別マッピングする文字列キー`PSActionKey`(`@"action"`)とは別物——確認済みのバグ(文字列キー経由だと無反応、`.buttonAction`に直すと直った)。

- アクションセレクタは引数0個または1個——1個で定義(`- (void)myRowTapped:(PSSpecifier *)specifier`)すると、タップされた行自身の`PSSpecifier`が自動的に渡される。共有ハンドラでどの行が発火したか判別するのに使える。
- 行ごとのカスタムメタデータ: `[specifier setProperty:value forKey:@"myKey"]`/`[specifier propertyForKey:@"myKey"]` — ファイル名やバンドルIDを行に紐付けてタップハンドラ側で読み戻す標準手段。
- `[PSSpecifier groupSpecifierWithName:@"..."]`でセクション見出し/フッター。
- 値を書いた後の`[self reload]`で`-specifiers`を再度呼び出し一覧全体を再構築(チェックマークやステータステキストの更新に使う標準手段)。

## 3. 選択中の行にネイティブなチェックマークを付ける

宣言的なチェックマークプロパティは無い。標準の`UITableViewDataSource`メソッドをオーバーライドし、`super`を呼んだ後に行の`specifier`を引き直して`accessoryType`を直接設定する:
```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [super tableView:tableView cellForRowAtIndexPath:indexPath];
    PSSpecifier *specifier = [self specifierAtIndex:[self indexForIndexPath:indexPath]];
    NSString *fileName = [specifier propertyForKey:@"myKey"];
    cell.accessoryType = [fileName isEqualToString:currentlySelected]
        ? UITableViewCellAccessoryCheckmark : UITableViewCellAccessoryNone;
    return cell;
}
```
純正の「着信音」ピッカーと同じ操作感(タップで即座に再生/選択、チェックマークで現在値を表示)。

## 4. 短いシステムサウンドの再生

公開APIの`AudioToolbox`のみ、プライベートフレームワーク不要:
```objc
#import <AudioToolbox/AudioToolbox.h>
SystemSoundID soundID;
NSURL *url = [NSURL fileURLWithPath:@"/System/Library/Audio/UISounds/tweet_sent.caf"];
if (AudioServicesCreateSystemSoundID((__bridge CFURLRef)url, &soundID) == kAudioServicesNoError) {
    AudioServicesPlaySystemSound(soundID);
}
```
音源は`/System/Library/Audio/UISounds/`を`.caf`拡張子でフィルタして列挙する(ハードコードしない、iOSバージョン差に強い)。

## 5. プロセスをまたいだライブリロード(Settings変更→既に表示中のSpringBoard UI)

Settingsの`PSSwitchCell`/`PSLinkListCell`は`NSUserDefaults`/`CFPreferences`(`cfprefsd`)経由で書き込む——SpringBoard側で同じplistを生の`-dictionaryWithContentsOfFile:`で読むコードは、Settings変更直後に古いデータを見てしまう。`CFPreferencesCopyAppValue`で読むこと。

さらに、正しく読めていても次にそのコードパスが走った時にしか反映されない。SpringBoard側UI(アイコンバッジ等)をSettings変更の瞬間に即座に更新する必要がある場合の確認済みパターン:
1. Root.plistの各行に`PostNotification`で共通のDarwin通知名を設定
2. SpringBoard側tweakで、これまでに見た全てのライブUI要素を弱参照の`NSHashTable`で保持(既存のライフサイクルフック`-layoutSubviews`等から蓄積)
3. 同じDarwin通知を`CFNotificationCenterAddObserver`で監視し、発火時にハッシュテーブルを走査して各要素の状態を再計算・再適用

## 6. 既知のシングルトンアクセサが無いUI要素へのクロスプロセスアクセス

Darwin通知経由でSpringBoard側コードが、既知の`+sharedInstance`を持たないUIコントローラー(App Switcherのビューコントローラー等)を呼ぶ必要がある場合: そのコントローラー自身のライフサイクルメソッド(`-viewDidLoad`等)が確実に呼ばれるタイミングで、**弱参照**をファイルスコープのグローバル変数に捕まえておく:
```objc
id __weak gCurrentControllerInstance = nil;
// -viewDidLoad 内: gCurrentControllerInstance = self;
```
別ファイルから`extern id __weak gCurrentControllerInstance;`で参照し、使用前に必ずnilチェックと`-respondsToSelector:`チェックを行う(そのインスタンスは前回のrespring以降一度も開かれていなければ存在しない可能性が正当にある)。

## 7. プレーンな`UITableViewController`(PSListController以外)でのスライダー行

- 初回セル構成時(`cellForRowAtIndexPath:`)の`if (!subview)`ブロック内で、`cell.contentView.bounds.size.width`からフレームを計算しない——この時点ではまだテーブルビューによる実幅へのリサイズが行われていない(リサイズはこのメソッドのリターン後に起きる)。`tableView.bounds.size.width`(この時点で既に正しい)を使い、サブビューの`.frame`は毎回(初回生成時だけでなく)再計算・再代入する。
- カスタム高さの行のテキストを`cell.textLabel`に設定した上で`.frame`も手動設定しない——`UITableViewCell`は内部レイアウトパス(`cellForRowAtIndexPath:`のリターン後に実行)で組み込みの`textLabel`を再配置するため、手動追加したサブビュー(スライダー等)と衝突しうる。カスタム高さ/追加サブビューがある行では`cell.textLabel.text = nil`のままにし、別の独自タグ付き`UILabel`を使う。
- 高さをオーバーライドした行は`-tableView:heightForRowAtIndexPath:`で対応する固定高さを返す必要がある(デフォルトのまま、他の行は`UITableViewAutomaticDimension`)。

## 8. PSListControllerの行アイコン(Root.plistでは運べない)

Root.plistにはアイコン用のキーが無い(plistの値は文字列/数値/真偽値/配列/辞書/日付/データのみ)。実際の仕組みは`PSTableCell`(`PSSwitchCell`/`PSLinkListCell`等全ての基底クラス)が持つ、公開だが未文書化の`@property (nonatomic, retain) UIImage *icon;`:
```objc
- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (![cell isKindOfClass:[PSTableCell class]]) return;
    PSSpecifier *specifier = [self specifierAtIndexPath:indexPath];
    ((PSTableCell *)cell).icon = MyIconImageForSpecifier(specifier);
}
```
`#import <Preferences/PSSpecifier.h>`と`#import <Preferences/PSTableCell.h>`が必要(`PSListController.h`からは前方宣言のみで透過的に入ってこない)。

行を「どの機能か」にマッピングするには、言語非依存のキーを使う(`specifier.name`は既にローカライズ済みの表示テキストなので不可):
- スイッチ/スライダー行: `[specifier propertyForKey:PSKeyNameKey]`(生の`"key"`文字列)
- `PSLinkListCell`のナビ行: `specifier.detailControllerClass`
- `PSButtonCell`行: `specifier.buttonAction`

## 9. 配列メンバーシップで駆動するPSSwitchCell

単純な真偽値keyではなく「あるIDが配列のメンバーかどうか」をトグルしたい場合、`target:set:get:`形式が正式な仕組み:
```objc
PSSpecifier *protect = [PSSpecifier preferenceSpecifierNamed:@"Protect" target:self
    set:@selector(setProtectValue:specifier:) get:@selector(protectValue:)
    detail:nil cell:PSSwitchCell edit:nil];
```
`get`は`NSNumber`でラップした値を返し、`set`は同じ型を受け取る——`-performGetter`/`-performSetterWithValue:`というPSSpecifier自身の正規の仕組みを経由する。

行ごとに異なるインスタンスデータ(どのアプリか等)が必要な詳細画面は、静的なRoot.plistの`detail`クラスにはできない。タップされた行を知っているリスト画面自身が`[[MyDetailController alloc] initForContentSize:self.view.bounds.size]`でインスタンスを生成し、プロパティを設定してから`pushViewController:animated:`で直接pushする——Preferences.frameworkの詳細クラス解決機構を迂回する、行ごとのデータを運ぶ唯一の方法。

## 10. 手組みPSSpecifierのローカライズ

`-loadSpecifiersFromPlistName:target:`(Root.plist駆動)は`label`/`footerText`を自動的に`Root.strings`テーブルに対して解決するが、この解決は**ローダーメソッド自身の中**で行われており、`PSSpecifier`自体の機能ではない。コードで手組みした`PSSpecifier`(`+preferenceSpecifierNamed:...`)は自動では翻訳されない:
```objc
static NSString *L(NSString *key, NSString *fallback) {
    NSBundle *bundle = [NSBundle bundleForClass:[ThisController class]];
    return [bundle localizedStringForKey:key value:fallback table:@"Root"];
}
```
英語ではエラーなく動くため、非英語環境で実際にテストするまで気づきにくい。

## 11. カスタム`accessoryView`での複数フラグ表示

複数のフラグのうちどれが立っているか一目で分かる行には、小さな円形`UIImage`(`UIGraphicsImageRenderer`)を複数+手描きのシェブロン(`chevron.right`)を並べた`UIView`コンテナを`cell.accessoryView`に割り当てる——`accessoryType`ではない(これを使うと自動のディスクロージャーインジケーターは失われ、必要なら自分で描く)。`UIStackView`を`accessoryView`に直接入れるのは見た目より脆い(フレーム/サイズが保証されない)ので、手動でx座標オフセットを指定したプレーンなサブビューの方が安定する。

## 12. `-[PSListController view]`の宣言型(`UITableView *`)はこのiOSビルドでは嘘——`-table`を使う

`PSListController.h`は`view`/`table`/`tableView`という3つの一見同じ`UITableView *`プロパティを宣言しているが、**このiOSビルドでは`self.view`は実行時には`UITableView`ではない**(プレーンな`UIView`ラッパー)。確認方法: `NSStringFromClass([self.view class])`が`UIView`を返し、`isKindOfClass:[UITableView class]`も`NO`。ヘッダーの宣言型を信じてキャストしたコード(`((UITableView *)self.view).tableHeaderView = ...`)はコンパイルは通るが、実行時には何も起きない(クラッシュも警告もない)。**`self.table`(または`self.tableView`)を使う**——本物の`UITableView`であることを確認済み。非自明なコードを組む前に、必ず`NSStringFromClass`でログして検証する。

## 13. `PSListController`画面への`UIDatePicker`埋め込み

3回失敗してから動く設計にたどり着いた:

1. `cell:PSGroupCell`を通常行として使う(カスタム`cellForRowAtIndexPath:`で`UIDatePicker`入りセルに差し替える狙い)——完全な空白画面になる。`PSGroupCell`はこのフレームワークの内部行カウントではセクション見出し/フッター専用で、通常行として`cellForRowAtIndexPath:`に到達しない。
2. `cell:PSTableCell`でコンパイル——**コンパイルエラー**。`PSTableCell`はObjective-Cのクラス名(`@interface PSTableCell : UITableViewCell`)であって、`PSCellType`列挙体のメンバーではない(有効な値は`PSGroupCell, PSLinkCell, PSLinkListCell, PSListItemCell, PSTitleValueCell, PSSliderCell, PSSwitchCell, PSStaticTextCell, PSEditTextCell, PSSegmentCell, PSGiantIconCell, PSGiantCell, PSSecureEditTextCell, PSButtonCell, PSEditTextViewCell, PSSpinnerCell`)。
3. `cell:PSStaticTextCell`に修正し、`cellForRowAtIndexPath:`内の`cell.contentView`に`UIDatePicker`を追加——クラッシュしないが見た目が崩れる(上部に小さな現在値表示、その下に黒い空白の矩形、ホイールの数字が一切見えない)。ダークモード設定も効果なし(単なる色の問題ではなく、specifier駆動のカスタム高さ行+セル再利用+`UIDatePicker`自身のレイアウト/スタイルタイミングの相互作用が原因。§12の`self.view`問題も同時に存在し悪化させていた)。
4. **✅ 動いた方法: specifier行セルを完全に迂回する。** `specifiers`は`+[PSSpecifier emptyGroupSpecifier]`をちょうど1つだけ返す(空配列はコントローラー自体がクラッシュする)。`UIDatePicker`(`.time`モード、iOS 13.4+なら`preferredDatePickerStyle = .wheels`、`sizeToFit`)を、実テーブル幅(`self.table.bounds.size.width`、0なら`[UIScreen mainScreen].bounds.size.width`にフォールバック)でサイズ調整したプレーンな`UIView`コンテナに包み、`self.table.tableHeaderView`に割り当てる——`self.view.tableHeaderView`ではない(§12)。`-viewWillAppear:`と`-viewDidLayoutSubviews:`からも(`-viewDidLoad`に加えて)同一性チェック付きで再代入する(このiOSビルドのPSListController内部の再読み込み/再レイアウトパスが、初回設定後に少なくとも1回の再設定を要求することを確認済み)。ホイールが正しく表示・スクロールし、`UIControlEventValueChanged`での`CFPreferencesSetAppValue`+`CFPreferencesAppSynchronize`書き込みも再訪問を跨いで永続化することを確認。

**余談**: この端末には`/usr/bin/log`が無い(`find`で確認済み)ため、Preferences.app内からの`NSLog`/`os_log`出力はSSH経由でストリーム/取得できない。デバッグ出力は`NSFileHandle`で`/var/mobile/`配下のファイルに直接追記する方式にする(`log stream`/`log show`は使えない)。
