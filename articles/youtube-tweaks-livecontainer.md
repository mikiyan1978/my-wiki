# YouTube Tweaks 導入(LiveContainer 用)

LiveContainer の中の YouTube(uYou)向けの Tweak を、Sileo のように「ソースから一覧を取得 → 追加・更新・取り除く」で管理する小さなアプリです。非脱獄の iPhone で、LiveContainer を使っているときに使います(実機の iPhone 14 Pro、iOS 26 で確認)。

## 入れ方
1. LiveContainer のソースに、`https://mikiyan1978.github.io/sileo-repo/lc-source.json` を追加します。
2. 「YouTube Tweaks 導入」を入れます(IPA を直接取り込むなら、[Releases の youtubetweaks-v1.0](https://github.com/mikiyan1978/sileo-repo/releases/tag/youtubetweaks-v1.0))。
3. LiveContainer でこのアプリの「マルチタスクで起動」を**オフ**にして、通常モードで開きます。
4. 一覧から Tweak を選んで「追加する」を押し、LiveContainer を完全に終了して、開き直します。

## 今ある Tweak
| Tweak | 内容 |
|---|---|
| Border Progress | 再生画面の縁を、再生の進み具合に合わせて線がなぞる(Android の ReVanced パッチのような表示)。YouTube の設定の「Border Progress」で、オン/オフ、太さ、開始位置(左上・上辺の中央)、色(14 種類のプリセットと、カスタム 2 色)、グラデーションの向き(斜め・横・縦・線に沿って一周)を変えられる |
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
