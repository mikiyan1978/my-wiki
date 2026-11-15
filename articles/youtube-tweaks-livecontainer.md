# LC Tweaks 導入(LiveContainer 用)

LiveContainer の中のアプリ(YouTube、Spotify など)向けの Tweak を、Sileo のように「ソースから一覧を取得 → 追加・更新・取り除く」で管理する小さなアプリです。非脱獄の iPhone で、LiveContainer を使っているときに使います(実機の iPhone 14 Pro、iOS 26 で確認)。

## 入れ方
1. LiveContainer のソースに、`https://mikiyan1978.github.io/sileo-repo/lc-source.json` を追加します。
2. 「LC Tweaks 導入」(旧名「YouTube Tweaks 導入」)を入れます(IPA を直接取り込むなら、[Releases の youtubetweaks-v1.1](https://github.com/mikiyan1978/sileo-repo/releases/tag/youtubetweaks-v1.1))。
3. LiveContainer でこのアプリの「マルチタスクで起動」を**オフ**にして、通常モードで開きます。
4. 一覧から Tweak を選んで「追加する」を押し、LiveContainer を完全に終了して、開き直します。

## 今ある Tweak
| Tweak | 内容 |
|---|---|
| Border Progress | 再生画面の縁を、再生の進み具合に合わせて線がなぞる(Android の ReVanced パッチのような表示)。YouTube の設定の「Border Progress」で、オン/オフ、太さ、開始位置(左上・上辺の中央)、色(14 種類のプリセットと、カスタム 2 色)、グラデーションの向き(斜め・横・縦・線に沿って一周)を変えられる |
| Spotify Tweak | Spotify の見た目と機能のカスタマイズ(ジャケットの回転、背景画像、歌詞の表示、ジェスチャーなど)。対象は、復号済みの Spotify の IPA を入れた LiveContainer([復号の手順](decrypt-ipa-to-livecontainer))。CydiaSubstrate 互換ライブラリが要る(下記) |
| 共有シートから YTDownloader LC へ | YouTube の共有シートに「YTDownloader LC」を足す。YTDownloader LC が必要([YTDownloader LC](ytdownloader-livecontainer-port)) |

## アプリの機能
- ソースの `index.json` から一覧を取得し、各 Tweak の最新の版、説明、状態(未追加・追加済み・更新あり)を表示する。
- 行を押すと、更新内容を見て、追加・更新・入れ直し・取り除きができる。
- 右上の「+」で、自分のソース(https の URL)を足せる。
- 入れる前に、ダウンロードした dylib の SHA-256 を、ソースの記録と照合する。一致しなければ入れない。
- Tweak 本体はアプリに同梱しないので、Tweak の追加や更新に、アプリの入れ替えは要らない。

## 仕組み
- アプリは、dylib を `$LC_HOME_PATH/Documents/Tweaks/<フォルダ>/` に置き、YouTube 側の `Documents/Applications/<アプリ>/LCAppInfo.plist` の `LCTweakFolder` にフォルダ名を書く。`LC_HOME_PATH` は、LiveContainer が設定する、本体の home の場所。
- LiveContainer は、**アプリごとに Tweak フォルダを 1 つしか選べない**ので、すでに選ばれているフォルダがあれば、そこに入れる(なければ `YouTubeTweaks` を作る)。入れた版は、dylib の横の隠しファイル `.ytt-<id>` に記録する。
- Tweak は、LiveContainer がアプリの起動時に署名して読み込む。ad-hoc 署名の dylib で動く。

## 対象アプリと Substrate 互換ライブラリ(1.1 から)
- Tweak ごとに、対象アプリの bundle ID(`targets`)を指定する。アプリは、その bundle ID のアプリにだけ、Tweak を入れる(YouTube は `com.google.ios.youtube`、Spotify は `com.spotify.client`)。
- Theos の Logos で書いた Tweak は、`CydiaSubstrate` を使う。非脱獄では、脱獄機にしかない場所を参照できないので、(1) dylib の参照を `@rpath/CydiaSubstrate.framework/CydiaSubstrate` に書き換え、(2) `@loader_path` を rpath に足し、(3) 互換ライブラリ(`CydiaSubstrate.framework`)を、dylib の横に置く。uYou の IPA に入っていた互換ライブラリで、実機(iPhone 14 Pro)で動いた。
- 互換ライブラリは、出どころのライセンスが確認できないため、配信の場所には置かない。`needsSubstrate` の Tweak は、アプリが、端末の LiveContainer に入っているアプリ(uYou を入れた YouTube など)の `Frameworks/CydiaSubstrate.framework` を、Tweak の横にコピーする。
- 互換ライブラリは arm64 のみ。Spotify の本体も arm64 だけなので、Tweak も arm64 の薄い版にする(arm64e を含めると、dyld が arm64e のスライスを選び、互換ライブラリと食い違って、読み込みに失敗するおそれがある。これは仕組みからの推測で、実機では、arm64 の薄い版だけを試した)。
- 一覧の場所は `lc-tweaks/index.json`。旧版のアプリ 1.0 は、対象アプリを知らないため、`youtube-tweaks/index.json`(YouTube の 2 つだけ)を、そのまま使う。

## 制限
- **通常モード(マルチタスクをオフ)でしか、入れられない。** マルチタスク表示のアプリは、別プロセス(LiveProcess)で動き、LiveContainer 本体のフォルダに触れない(実機で「アクセス権がない」を確認)。
- SpringBoard には、できない。LiveContainer の中のアプリや Tweak は、その中のアプリにしか入らない。
- 入れたあとは、LiveContainer を完全に終了して、開き直す必要がある。

## Tweak を足す・更新する(作者向け)
ソースは [iOSTweaks の YouTubeTweaks](https://github.com/mikiyan1978/iOSTweaks/tree/main/YouTubeTweaks)。`Tweaks/<名前>/` に `Tweak.m`、`build.sh`、`tweak.json`(`id`、`name`、`description`、`version`、`changelog`、`dylib`)を置き、`scripts/publish.py` で、dylib と `index.json` を作って、`sileo-repo` の `youtube-tweaks/` に置く。版を上げると、アプリに「更新あり」と出る。

## 調べて分かったこと
- フックできる YouTube 側のクラス(実機のバイナリで存在を確認): `YTPlayerViewController`(`playerView`、`currentVideoMediaTime`、`currentVideoTotalMediaTime`、`isPlayingAd`)。
- 設定画面に項目を足す方法は、YouPiP などと同じ。4 文字のコード(`ytbp` など)を独自のカテゴリーにして、`YTSettingsGroupData orderedCategories`、`YTAppSettingsPresentationData settingsCategoryOrder`、`YTSettingsSectionItemManager updateSectionForCategory:withEntry:` を差し替える。
- LiveContainer の Tweak の読み込みは、Tweak 同士が競合しない(同じフォルダに複数の dylib を置ける)。
- 設定画面が無い環境(PreferenceLoader が使えない LiveContainer)では、設定値が未設定になる。`integerForKey:` は、未設定だと 0 を返すので、0 に意味がある設定(Spotify Tweak の「ホーム背景の位置」の「左寄せ」など)は、未設定のときの既定値を、コードで明示する必要がある(実際に、ホーム背景が左にずれた)。
