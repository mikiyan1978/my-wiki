# FastFreeze.xm — バックグラウンドに回ったアプリを即座に強制終了するtweak、4連敗の記録

(2026-10-01, iPhone 8 Plus。商用tweak「Adytum」の"Fast Freeze"機能(バックグラウンド音楽再生のような正当なentitlementを持つ最も厳しいケース=Spotifyを対象に、独自実装で挑戦。`~/iOSTweaks/SwitcherGrid/FastFreeze.xm`。1セッションで4回失敗してから動く設計にたどり着いた、OS側の壁の記録。)

## 失敗1: シーン遷移フックでは検知できない

`StayForeground.xm`が使っている`%hook FBScene -updateSettings:withTransitionContext:completion:`フックを流用し、`-deactivationReasons`の0↔非0遷移を監視しようとした。2つのバグが重なっていた:

- `%orig`呼び出しの前後比較が無意味(`settings`オブジェクトは`%orig`では変化しない。ビルドごとに「前回呼ばれた時の値」を辞書で保持して比較する必要があった)
- それを直しても、Spotifyでは`-deactivationReasons`が常に`0x0`のまま——**「audio」バックグラウンドモードを宣言しているアプリに対しては、iOSはそもそも非アクティブ化の脅威を与える必要がないため、シーン遷移/非アクティブ化フックでバックグラウンド化を検知すること自体ができない。**

**教訓**: 正当なバックグラウンド実行entitlement(audio, location, VoIP等)を持つアプリに対しては、この種のフックポイントは使えない。そして皮肉なことに「バックグラウンドで強制終了したい」という機能が最も欲しがる対象こそ、まさにそのentitlementを持つアプリである。

## 失敗2: ポーリング+SIGSTOP/SIGCONTは、OSのウォッチドッグに殺される

1Hzタイマーで「今フォアグラウンドか」をポーリングし、`kill(pid, SIGSTOP)`/`SIGCONT`で凍結/再開する方式に切り替え。`SIGSTOP`自体は実際に機能した(`result==0`、プロセスは本当に停止)が、iOSの**「プロセス起動ウォッチドッグ」**が無応答許容時間を使い切ったプロセスを強制終了し、バックグラウンド音声継続のために再起動してしまう——クラッシュログに文字通り`"process-launch watchdog transgression: ... exhausted real (wall clock) time allowance of 20.00 seconds"`という行が出る。ポーリングは次のtickで新しいPIDをまた凍結し、無限ループに。

ユーザー自身が確認したところ、**商用のAdytumの実際の"Fast Freeze"機能もSpotify相手に全く同じ失敗をする**——OSレベルの壁であって、どちらの実装のバグでもない。Adytumの文字列を調べても音声/バックグラウンドモードの除外ロジックは一切見つからなかった(商用競合も同じ問題を黙って負けている)。

**教訓**: 正当にレスポンシブさを維持しようとしているアプリに対してSIGSTOPは使えない。FrontBoard(`FBProcessManager`/`FBApplicationProcess`)もRunningBoard系クラス(`RBSProcessHandle`/`RBSProcessMonitor`)も読み取り専用のクエリ(`-currentState`/`-fb_canTaskSuspend`)しか公開しておらず、実際のサスペンドは`runningboardd`へのXPC経由——SpringBoard自身が呼べる単純なObjective-Cメソッドではない。

## 失敗3: SIGKILLに切り替えたが、再起動ループ(原因は別)

`SIGSTOP`の代わりに本物の`kill(pid, SIGKILL)`(KillGuard.xmで既に実績のある手法)に変更。デプロイすると**即座に再起動ループ**(数秒間隔で3つの異なるPIDが連続でkillされる)。

tweak全体を介さず、SSH経由で手動の`kill -9 <pid>`を1回だけ送って切り分けたところ、**それだけでは再起動しなかった**——つまり「OSが音声継続のために勝手に再起動する」仮説は誤りで、**自分自身のポーリングが毎tick「フォアグラウンドでない→kill」を再評価し続けていた**のが原因だった。

**教訓**: 「OSが勝手に再起動する」と決めつける前に、自分のポーリング/リトライロジックを完全に外した状態で生のsyscallを単体テストする。説明のつかないループは、しばしば自分自身のコードが(自分が起こした状態に対して)再試行しているだけ。**修正**: バンドルIDごとのクールダウン(30秒)——kill後、本当にフォアグラウンドに戻ってくるまで同じバンドルIDは再killしない。これにより将来どんな原因のループも自己制限的になる。

## 失敗4: フォアグラウンド判定自体が間違っていた

クールダウンを入れても、「本当にフォアグラウンドで使用中なのに殺された」という報告。当時のフォアグラウンド判定は`+[FBProcessManager sharedInstance] -currentProcess] -bundleIdentifier`との比較だった。

**`-currentProcess`は名前に反して「今画面に見えている最前面アプリ」を意味するとは限らない**——push通知やウィジェット更新など、システムの他の一時的な活動が一瞬だけ別プロセスを指すことがあり、1秒ポーリングがその瞬間に当たると、画面には見えている(=正しい)アプリが殺されてしまう。

**修正**: 「今誰が最前面か」というグローバルな問いから、「あなた(対象アプリ)はフォアグラウンドか」という個別の問いに切り替えた——`-[FBProcessManager applicationProcessesForBundleIdentifier:]`で対象バンドルID自身の`FBApplicationProcess`を取得し、その`-[FBProcess isForeground]`を直接問い合わせる。2分間の実使用で誤killゼロ、その後の本物のバックグラウンド化で正しく1回killされることを確認。

**教訓**: `currentProcess`/`frontmostX`のような名前のアクセサが実際に何を返すかは、名前からは保証されない。ミューテーション操作の検証は「離れている」という否定ケース1回では不十分——「本当に数分間フォアグラウンドのまま」という持続的な肯定ケースも、未確認のクエリメソッドを信頼する前に必ず試す。

## 最終的に動いた設計

```
1Hzタイマー → FastFreezeBundleIDs の各エントリについて:
    FFIsForeground(bundleID)   // 自分自身のFBApplicationProcessで-isForeground
    フォアグラウンドならクールダウンのタイムスタンプをクリア
    そうでなければ FFKillIfRunning(bundleID)   // kill(pid, SIGKILL)、直近30秒以内のkillならスキップ
```

PID検索はKillGuard.xmと同じ`sysctl(KERN_PROC_ALL)`スキャン(`CFBundleExecutable`名で一致)を移植。

## おまけ: Settings.appのピッカーとファイル直書きのキャッシュ不整合

同プロジェクトの別ファイル(IconMenu.xm)で、3D Touchの「保護」/「Fast Freeze」トグルが`com.mikiyan1978.appguardian.plist`に`writeToFile:atomically:`で直接書き込んでいたが、同じドメインをAltList系のSettings.appピッカーは`NSUserDefaults`/`CFPreferences`(`cfprefsd`のプロセスごとキャッシュ)経由で読んでいたため、トグル直後もピッカー側は古い状態を表示し続けた。**修正**: `CFPreferencesSetValue` + `CFPreferencesAppSynchronize`(`kCFPreferencesCurrentUser`、`/var/mobile/Library/Preferences/`配下なので)で書くようにし、ディスク上のファイルは変わらず直接読み取り勢もそのまま動きつつ、`cfprefsd`のキャッシュも正しく無効化されるようにした。

## 商用競合Adytym(`com.kernelrw.adytum`)の解析メモ

`otool`/`nm`/`strings`のみで解析(デコンパイラ無し、正確な呼び出し順序は未追跡)。

- MobileSubstrateのtweakで、フックは全て`NSClassFromString`で実行時解決(未定義の`_OBJC_CLASS_$_*`シンボルがゼロ) — `nm -u`からターゲットクラスを読み取れない設計
- `FBProcessManager`/`FBApplicationProcess`/`FBScene`関連のセレクタ文字列、`dispatch_source_set_timer`のインポート(ポーリング型、遷移フック型ではない)を確認
- `SKConfigSnapshot`クラスや`SK*`系のC関数群は、一見StoreKit関連に見えたが、実際はベンダー独自の内部名前空間(OSバージョン互換チェックや設定プロファイルのエクスポート/インポート用)——**クラスプレフィックスから特定のAppleフレームワークを推測しない、裏付けなしに**
- `SBSApplicationShortcutCustomImageIcon`という第4の`SBSApplicationShortcutItem`アイコンサブクラスを発見(`-initWithImagePNGData:`で事前レンダリング画像を渡せる) — `SBSApplicationShortcutSystemIcon`にはtint/color APIが一切無いため、Adytumはこれで3D Touchメニューにカラーアイコンを実現していた。同じ手法をIconMenu.xmに移植(SFシンボルを`UIImageSymbolConfiguration`でレンダリング→PNG化→`SBSApplicationShortcutCustomImageIcon`でラップ)
- audio/バックグラウンドモード/ウォッチドッグを意識した文字列は見つからず——商用製品もこの失敗モードを特別扱いしていない
