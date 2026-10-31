# YTDownloader 完全インストール・ユーザーズガイド

脱獄済みiOS端末向けの、YouTube動画ダウンロード単体アプリ。端末上の`yt-dlp`をネイティブUIでラップし、URL貼り付けまたはYouTubeアプリの共有メニューから直接、解像度を選んでダウンロードできる。

配布元: [mikiyan1978.github.io/sileo-repo](https://mikiyan1978.github.io/sileo-repo/)

> ⚠️ **ルートフル専用として開発・検証**: iPhone 8 Plus / iOS 16.7.16 / palera1n rootful で開発・実機検証済み。コード側はルートレス(Dopamine等、`/var/jb`配下)のパス構成にも対応させてあるが、**実際のルートレス端末での動作確認はまだ行っていない**。

---

## 必要なもの

- 脱獄済みiOS端末(Sileoが使えること)
- インターネット接続
- 画質制限なしでダウンロードしたい場合は、空き容量 **約300MB以上**(後述のランタイム一式)

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

## STEP 3: yt-dlp本体を導入

YTDownloaderはあくまで`yt-dlp`のラッパーであり、`yt-dlp`自体は同梱されていない。SSH、またはZeus/NewTermのような端末内ターミナルアプリから:

```
pip3 install yt-dlp
```

アプリ初回起動時、`yt-dlp`が見つからない場合はアプリ内に正確なコマンドが表示されるので、それに従ってもよい。

> ⚠️ **ここが一番の注意点**: yt-dlpの最新版は **Python 3.11以上を必須**としている。多くの脱獄用パッケージリポジトリ(Procursus等)で配布されているPythonは **3.9系**までで、そのままでは最新のyt-dlpが起動すらしない(`ImportError: Only Python versions 3.11 and above are supported by yt-dlp`)。この状態でも360p程度までのダウンロードは可能だが、それ以上の画質は後述のSTEP 4が必須。

---

## STEP 4(推奨): フル画質ダウンロード用ランタイムの導入

YouTube側のPO Token(Proof-of-Origin)要件により、360pを超える画質でのダウンロードには、単にyt-dlpがあるだけでは足りず、**PO Tokenを発行するローカルサーバー**が別途必要になる。これを動かすには新しめのPython(3.13)とNode.js 22が要る。

これらを自前ビルドしてまとめたランタイム一式(約120MB・展開後約520MB)を配布しているので、以下のワンライナーを **root権限のターミナル(SSH等)** から実行するだけで導入できる:

```
curl -sL https://raw.githubusercontent.com/mikiyan1978/sileo-repo/main/scripts/ytdownloader_install_runtime_bundle.sh | sh
```

- ルートフル/ルートレスを自動判別し、適切な場所(`/usr/local` または `/var/jb/usr/local`)に展開する
- 導入後はアプリを再起動するだけで自動的に検知される。respring不要
- 中身: Python 3.13インタプリタ + Node.js 22(`--jitless`) + [bgutil-ytdlp-pot-provider](https://github.com/Brainicism/bgutil-ytdlp-pot-provider)

導入しなかった場合、アプリは自動的に360p上限のフォールバック動作になる(エラーにはならない)。

---

## トラブルシューティング

- **「Cookieの有効期限が切れています」と出る/画質が上がらない**: Cookie取り込みは非公開・年齢制限動画のためのものであり、通常の公開動画の画質上限とは無関係。STEP 4のPO Tokenサーバーが本体。
- **ダウンロードが`exit -1`で失敗する**: ダウンロード中にアプリ側の停止ボタンを押すと出る正常な表示で、バグではない。
- **全く起動しない/画質選択が360p止まり**: STEP 3のPythonバージョン要件を再確認。`python3 --version`が3.11未満ならSTEP 4が必須。

---

## 開発者

**mikiyan1978** ／ ソース: `mikiyan1978/iOSTweaks`(非公開) ／ 配布: [mikiyan1978/sileo-repo](https://github.com/mikiyan1978/sileo-repo)
