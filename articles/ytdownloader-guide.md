# YTDownloader 完全インストール・ユーザーズガイド

脱獄済みiOS端末向けの、YouTube動画ダウンロード単体アプリ。端末上の`yt-dlp`をネイティブUIでラップし、URL貼り付けまたはYouTubeアプリの共有メニューから直接、解像度を選んでダウンロードできる。

配布元: [mikiyan1978.github.io/sileo-repo](https://mikiyan1978.github.io/sileo-repo/)

> ⚠️ **ルートフル専用として開発・検証**: iPhone 8 Plus / iOS 16.7.16 / palera1n rootful で開発・実機検証済み。コード側はルートレス(Dopamine等、`/var/jb`配下)のパス構成にも対応させてあるが、**実際のルートレス端末での動作確認はまだ行っていない**。

---

## 必要なもの

- 脱獄済みiOS端末(Sileoが使えること)
- インターネット接続
- 空き容量 **約300MB以上**(後述のランタイム一式——これが無いとそもそもダウンロードできません)

---

## STEP 1: Sileoリポジトリを追加

1. Sileoを開き、下部の **ソース** タブへ
2. 右上の「＋」から `https://mikiyan1978.github.io/sileo-repo/` を追加

---

## STEP 2: YTDownloaderをインストール

1. 追加したリポジトリ内から **YTDownloader** を検索してインストール
2. 依存パッケージ **BulletinRelay**(ダウンロード完了通知用の共有tweak、こちらも同リポジトリに同梱)が自動で一緒にインストールされる
3. インストール後、**respring**(再起動)

> BulletinRelayは単体で開くものではなく、YTDownloaderの通知表示のために裏で動く中継tweak。特に設定操作は不要。

---

## STEP 3: フル画質ダウンロード用ランタイムの導入(実質必須)

> ⚠️ **ここが一番重要**: 当初「このステップは任意で、やらなくても360p程度は落とせる」と案内していましたが、実機検証の結果これは誤りでした。YouTube側が現在ほぼ全クライアントで新しい配信方式(SABR)を強制しており、**普通にapt/pipで手に入る程度のyt-dlpでは、画質を問わず全く何もダウンロードできません**(`ERROR: The page needs to be reloaded.`で毎回失敗)。まともに動かすには、新しいyt-dlpビルド(Python 3.11以上が必須)と、PO Token(Proof-of-Origin)を発行するローカルサーバーの両方が要ります。事実上、本ステップを飛ばすとアプリは使えないとお考えください。

これら(Python 3.13 + Node.js 22 + PO Tokenサーバー)を自前ビルドしてまとめたランタイム一式(約120MB・展開後約520MB)を配布しているので、**root権限のターミナル(SSH等)** から以下のワンライナーを実行するだけで導入できる:

```
curl -sL https://raw.githubusercontent.com/mikiyan1978/sileo-repo/main/scripts/ytdownloader_install_runtime_bundle.sh | sh
```

- ルートフル/ルートレスを自動判別し、適切な場所(`/usr/local` または `/var/jb/usr/local`)に展開する
- 導入後はアプリを再起動するだけで自動的に検知される。respring不要
- 中身: Python 3.13インタプリタ + Node.js 22(`--jitless`) + [bgutil-ytdlp-pot-provider](https://github.com/Brainicism/bgutil-ytdlp-pot-provider)

> 参考までに、多くの脱獄用パッケージリポジトリ(Procursus等)で配布されているPythonは3.9系までで、最新のyt-dlpはPython 3.11以上を必須としているため、`pip3 install yt-dlp`だけでは解決しません(`ImportError: Only Python versions 3.11 and above are supported by yt-dlp`)。

---

## トラブルシューティング

- **ダウンロードがすべて「ログを確認してください」で失敗する**: ほぼ確実にSTEP 3のランタイム未導入が原因。導入済みのはずなら、アプリを完全に再起動(タスクキル→再起動)してから再試行。
- **「Cookieの有効期限が切れています」と出る**: Cookie取り込みは非公開・年齢制限動画のためのものであり、通常の公開動画のダウンロード可否とは無関係。
- **ダウンロードが`exit -1`で失敗する**: ダウンロード中にアプリ側の停止ボタンを押すと出る正常な表示で、バグではない。

---

## 開発者

**mikiyan1978** ／ ソース: `mikiyan1978/iOSTweaks`(非公開) ／ 配布: [mikiyan1978/sileo-repo](https://github.com/mikiyan1978/sileo-repo)
