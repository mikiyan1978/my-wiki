# SpringBoard: Force Touch(通知プレビュー)とアイコン非表示/「ホーム画面から取り除く」の内部実装

Frida観察だけで調べた、ホーム画面アイコンまわりの2つの機能の内部構造。いずれも**読み取り専用の観察が先、ミューテーション系APIは危険**という教訓に行き着いた。

## 1. Force Touch(長押し/強押しで通知プレビューを出す機能)

(2026-09-26, iPhone 8 Plus, SpringBoard PID例42842)

「ホーム画面アイコンを長押し/3D Touchで通知プレビューを出す」機能は、標準の`UIContextMenuInteraction`APIの薄いラッパーに過ぎず、大規模な専用サブシステムではない。

- **`SBIconView`**(アイコン1つにつき1インスタンス、`UIContextMenuInteractionDelegate`):
  - `contextMenuInteraction:configuration:highlightPreviewForItemWithIdentifier:` — 押下中のハイライト表示
  - `_contextMenuInteraction:previewForIconWithConfigurationOptions:highlighted:` — プレビュー内容(通知プレビュー含む)を実際に構築。プレビュー表示中に何度も再入される(ライブ更新されているように見える)
  - `contextMenuInteraction:willPerformPreviewActionForMenuWithConfiguration:animator:` — プレビューをタップして開いた時に発火
  - `gestureRecognizerShouldBegin:` — タッチ確定ゲート、ジェスチャー調停中に何度も呼ばれる
- **`SBIconController`**(全アイコンを所有):
  - `_forceTouchControllerWillPresent:` / `_iconForceTouchControllerWillPresent:` — プレビュー表示のトリガー
  - `dismissAppIconForceTouchControllers:`
  - ivar `_appIconForceTouchWindows`(`NSSet`) — 現在表示中のプレビューウィンドウ集合を保持するだけ。専用コントローラークラスは見つからず、状態はこのSetとウィンドウ自体に存在
- **`SBHomeScreenViewController`**: `_appIconForceTouchControllerWillPresentNotification:` / `...DidDismissNotification:` — 純粋な`NSNotificationCenter`リスナーで、回転を抑制しウィンドウレベルのアサーションを取得/解放するだけ(独自の表示ロジックは持たない)

観察された呼び出し順序(押下→プレビュー表示→指を離す):
```
contextMenuInteraction:configuration:highlightPreviewForItemWithIdentifier:
  _contextMenuInteraction:previewForIconWithConfigurationOptions:highlighted:
gestureRecognizerShouldBegin:  (繰り返し)
_forceTouchControllerWillPresent:
_appIconForceTouchControllerWillPresentNotification:
  (直後に) _contextMenuInteraction:previewForIconWithConfigurationOptions:highlighted:  (再入)
--- 指を離す ---
_appIconForceTouchControllerDidDismissNotification:
```

## 2. アイコン非表示/「ホーム画面から取り除く」の内部実装

(2026-09-27, SpringBoard PID例10320。Peekaboo的な「非表示にしたアイコンの隙間を自前のレイアウト操作なしで埋めたい」という目的で調査)

### 2つの異なる可視性メカニズムが共存している

1. **ページ/フォルダ単位の「このリスト全体を隠す」フラグ** — `SBIconListModel`:
   `isHidden`/`setHidden:`/`setHidden:byUser:hiddenDate:`、`removeIcon:`(`v24@0:8@16`)、`removeIcon:options:`(`v32@0:8@16Q24`、optionsのビット意味は未解読、観測値は常に`0`)など。
2. **タグベースの一括可視性システム**(Screen Time/MDMのアプリ制限と同じ仕組みを共有していると思われる) — `SBIconModel`の`isIconVisible:`/`adjustIconsToShow:iconsToHide:afterChangeToVisibleIconTags:hiddenIconTags:`、`SBApplicationInfo`の`hasHiddenTag`/`visibilityOverride`、`SBIconController`の`_visibleTags`/`_hiddenTags`(`applicationRestrictionController:didUpdateVisibleTags:hiddenTags:`経由、これはScreen Time/MDM専用と判明)。

### 実機観察で判明したこと

ユーザーが実際に長押し→「Appを削除」→「ホーム画面から取り除く」を実行する間、読み取り専用で上記メソッドを観察(`observe_remove_from_home.js`)した結果:

```
ENTER -[SBIconController isHideSupportedForIcon:inLocation:]
ENTER -[SBIconListModel removeIcon:options:]   icon=..., options=0x0
```

`adjustIconsToShow:iconsToHide:...`(タグベース側)は**一度も呼ばれなかった** — 「ホーム画面から取り除く」はリスト単位の`removeIcon:options:`(options=0)で実装されており、タグベースの仕組みはScreen Time/MDM専用と確定した。

### ⚠️ 危険: 実際に呼び出してSpringBoardを2回クラッシュさせた

観察だけでなく実際に`removeIcon:options:0`→`addIconToHomeScreen:`を呼んでみたところ:

1. Frida直接呼び出し(メインスレッド外)→ `addIconToHomeScreen:`で`abort was called`(Frida側が安全に停止、クラッシュなし)
2. `performSelectorOnMainThread:waitUntilDone:YES`で正しくメインスレッドに乗せて再試行 → **今度は本当にSpringBoardが`EXC_BAD_ACCESS`でクラッシュ**(クラッシュログ2本で確認、`__NSThreadPerformPerform`経由でメインスレッド上で実行中だったことも確認)。ただし`removeIcon:options:0`自体は`isIconVisible`に何の効果も与えなかった(ペアリング自体が間違っている可能性も残る)。

**結論: `removeIcon:options:`/`addIconToHomeScreen:`を実運用のtweakから呼ばない。** クラッシュ原因が「APIの前提条件違反」か「Fridaでのテスト特有のアーティファクト」か切り分けできないまま、2回のクラッシュは十分なシグナル。

### ✅ 実際に採用した安全な解決策: ミューテーションではなくクエリをフックする

同種の問題を解決済みの既存tweak(**Hid3x**, github.com/ipadkid358/Hid3x)を先に探すべきだった、という反省のもと発見。そのTweak.xの全体:

```objc
%hook SBIconModel
// requires a respring, only called on SpringBoard load
- (BOOL)isIconVisible:(SBApplicationIcon *)icon {
    if ([hideList containsObject:icon.applicationBundleID]) return NO;
    return %orig;
}
%end
```

`removeIcon:`や`addIconToHomeScreen:`のようなミューテーション系APIを一切呼ばない。**レイアウトエンジンが「このアイコンにグリッドの枠を割り当てるか」を判断する際に必ず参照する、唯一の真実のソース(`isIconVisible:`というクエリメソッド)の戻り値を書き換えるだけ** — SpringBoard自身の、既に正しく動くレイアウトコードがそのアイコンを自然にスキップしてくれる。クラッシュリスクはゼロ(ただの戻り値上書き、MacDock/SnowFall/Peekabooの他の箇所と同じ安全なパターン)。Peekaboo(`~/iOSTweaks/Peekaboo/SpringBoard.xm`)に実装済み。

**トレードオフ**: `isIconVisible:`はレイアウト計算時にのみ参照される(常時ポーリングではない)ため、新規に非表示にしたアイコンの隙間は**次のレイアウト再計算(respring/回転など)まで埋まらない**。Peekabooは即座の見た目の非表示には既存の`SBIconView.hidden`トグルを併用し、隙間埋めだけこの制約を受け入れている。

**教訓: プライベートなミューテーション系APIをゼロから逆解析する前に、同じ問題を解決済みの既存tweakが無いか5分だけ検索する。** 今回は検索が後回しになり、クラッシュ2回の後にようやく見つかった。

### respring無しで隙間を閉じる試み(失敗)

`-[SBIconController _updateVisibleIconsAfterOcclusionChange]`(「今見えているアイコンを再計算してリレイアウトする」トリガーではと仮説)を試したが、クラッシュもせず効果もなし(「オクルージョン」はウィジェット等による一時的な遮蔽の話で、グリッドメンバーシップ/レイアウトとは無関係と判明)。Screen Time機能自体はrespring無しでライブに非表示/復元ができることが分かっているので、`applicationRestrictionController:didUpdateVisibleTags:hiddenTags:`を読み取り専用で観察するのが次の有望な手がかり(未着手)。

## 安全な代替手段(既に使われている、ミューテーション一切なし)

SpringBoardの実モデルに一切触れない: アイコン自身のビュー(`SBIconView.hidden`/`.userInteractionEnabled`)だけをトグルする。隙間は残るが完全に安全(クラッシュリスクゼロ、持続化状態に一切触れない、即座に元に戻せる)。
