# YTDownloaderをLiveContainerで動かす(Python+yt-dlpをアプリ内に組み込む)

関連: [YTDownloader TS(TrollStore版)](./ytdownloader-trollstore-guide.md) / [LiveContainer ガイド](./livecontainer-guide.md) / [UTM on iPhone](./utm-on-iphone-guide.md)

TrollStore版のYTDownloaderは、LiveContainerに入れても動きません。そこで「外部プログラムを起動せず、すべて同一プロセスの中で動かす」形に作り替えられるかを、実験アプリで確かめ、その後、実アプリ(YTDownloader)に組み込んで **LiveContainer 版**として配布した記録です。端末は iPhone 14 Pro(A16)、iOS 26.6.2、LiveContainer(JITなし)。

---

## どちらの版を入れる?

アプリは同じソースから作っていますが、端末の環境によって入れる版が違います。LiveContainer 版の詳細は [YTDownloaderをLiveContainerで動かす](./ytdownloader-livecontainer-port.md) を参照してください。

| あなたの環境 | 入れる版 | 入れ方 |
|---|---|---|
| **TrollStore が使える** | **TrollStore 版**(`.tipa`) | TrollStore で開いてインストール |
| TrollStore が使えない。**LiveContainer** が使える | **LiveContainer 版**(`.ipa`) | LiveContainer の「ソース」に追加するか、「+」から IPA を取り込む |

### 端末の iOS バージョンで見る

| 端末の iOS | TrollStore 版 | LiveContainer 版 |
|---|---|---|
| 13 以下 | 使えない | 使えない |
| 14.0 〜 15.x | 使える | 使えない(LC 版は iOS 16.0 以降) |
| 16.0 〜 16.6.1、16.7 RC(20H18)、17.0 | 使える | 使える |
| 16.7(RC 以外)、17.0.1 以降(18、26 を含む) | 使えない | 使える |

- TrollStore 自体が対応する iOS は、公式の記載では「14.0 beta 2 〜 16.6.1、16.7 RC(20H18)、17.0」です。16.7(RC 以外)と 17.0.1 以降は、今後も対応されない見込みと書かれています([TrollStore の README](https://github.com/opa334/TrollStore))。
- LiveContainer 自体は iOS/iPadOS 15 以降で、AltStore 2.0 以降または SideStore 0.6.0 以降で入れます([LiveContainer の README](https://github.com/LiveContainer/LiveContainer))。このアプリの LC 版は、最低 iOS 16.0 に設定しています。
- アプリ本体の最低バージョンは、TrollStore 版が iOS 14.0、LC 版が iOS 16.0 です。

### 機能の違い

| | TrollStore 版 | LiveContainer 版 |
|---|---|---|
| 最高画質(最大2160p)の保存 | できる | できる(2160p は AV1 が多く、A16 など AV1 非対応の端末では、アプリ内で再生できない) |
| 音声のみ | MP3 / M4A / Opus / FLAC / WAV | M4A または Opus(`.mka`)。設定で選ぶ |
| 字幕の埋め込み、SponsorBlock | できる | できない(項目は非表示) |
| 区間だけのダウンロード | できる | 未対応(ffmpeg が必要なため) |
| 映像と音声の結合 | ffmpeg | AVFoundation(H.264 / AV1 の mp4 + m4a)。VP9 + Opus は自作の Matroska 書き出しで `.mkv` / `.webm` に結合 |
| 動画を渡す方法 | 共有メニュー、リンクのコピー | 共有メニュー用ショートカット、またはリンクをコピーして貼り付け |
| yt-dlp の更新 | アプリ内で更新 | アプリ内で更新(設定の「ダウンロードエンジン」。「同梱の版に戻す」も可) |
| アプリの更新案内 | あり | あり(起動時、12時間に1回まで) |
| 初回の準備 | ランタイムをダウンロード(約120MB、展開後は約520MB) | なし(アプリに同梱、アプリは約50MB) |

迷ったら、TrollStore が使えるなら TrollStore 版を選んでください(機能が多い)。両方入れても干渉しません(bundle ID が別です)。入れた版が環境に合っていないときは、アプリを開くと案内が出ます。

LiveContainer 版の確認範囲は、実機(iPhone 14 Pro、iOS 26.6.2、JIT なし)で数本の動画です。年齢制限・ライブ・メンバー限定は未確認です。

## 実アプリへの組み込みと配布

実験で確かめた方式を、YTDownloader 本体のソースに組み込みました(同じソースを `-DYTD_LC=1` でコンパイルすると LiveContainer 版になります)。ソースは `mikiyan1978/iOSTweaks` の `YTDownloaderTrollStore/`、LC 版の部品は `LiveContainer/` にあります。

### 入れ方
- **LiveContainer の「ソース」に追加:** `https://mikiyan1978.github.io/sileo-repo/lc-source.json`(AltStore 形式)。一覧から YTDownloader LC をインストールします。
- **IPA を直接取り込む:** [GitHub Releases の ytdownloader-lc-v1.3.1](https://github.com/mikiyan1978/sileo-repo/releases/tag/ytdownloader-lc-v1.3.1) から IPA(署名なし、約14MB)を取り、LiveContainer の「+」から取り込みます。LiveContainer が自分の証明書で署名します。

### 動作を確認できたこと(iPhone 14 Pro、iOS 26.6.2、JIT なし)
- 動画の情報取得、画質の一覧、ダウンロード、映像と音声の結合、ライブラリでの再生。
- 「互換性優先」をオンにすると、H.264 に絞られて再生できる。
- ライブラリのサムネイルと再生時間の表示(下記)。

### LC 版での置き換え・制限
| 項目 | 内容 |
|---|---|
| サムネイル・再生時間 | ffprobe/ffmpeg が無いので、yt-dlp が保存した webp/png をそのまま使い、無ければ AVFoundation で1フレームを取り出す。再生時間と寸法も AVFoundation で読む |
| Safari の Cookie | 読めない(LiveContainer の中のアプリは、他のアプリの領域にアクセスできない)。cookies.txt をファイルから読み込む方式のみ |
| yt-dlp の更新 | 設定の「ダウンロードエンジン」で最新の yt-dlp(zipapp)を取得して差し替える。SHA-256 を検証し、読み込みに失敗したら同梱の版へ自動で戻す |
| MKV / WebM | AVFoundation は開けないので、結合は自作の EBML 書き出し(`mkvmux.py`)。再生は VLC など(共有時は APFS のクローンで、容量を増やさず元のファイル名のまま渡す) |
| 音声のみ | DASH の m4a は再生時間が2倍に見えるため、書き出し(パススルー)で作り直す。Opus は `.mka` |
| アプリの更新案内 | 起動時に、公開中の版と比べて案内を出す |
| 完了・失敗の通知 | LiveContainer の中のアプリには、iOS が通知を許可しない(許可を求めても、確認が出ずに `UNErrorDomain Code=1` で失敗する。実機で確認)。代わりに、画面の上のバナーと振動(アプリを開いているとき)、ライブラリのタブの件数の印(閉じている間に終わった分)で知らせる |
| 動画を渡す方法 | 共有メニューからは直接開けないので、**ショートカット**を使う(下記)。または、YouTube でリンクをコピーし、アプリの「URLを貼り付けて開始」を押す |
| 初回起動 | 「準備完了」の説明画面を表示。設定の「ヘルプ」から、使い方ガイドと説明画面をいつでも開き直せる |
| 版の食い違い | TrollStore 版を LiveContainer の中で起動、または LC 版を LiveContainer の外で起動すると、アプリが案内を出す(LiveContainer が設定する環境変数 `LC_HOME_PATH` で判定) |

### 共有メニューから渡すショートカット
LiveContainer の起動 URL を組み立てるショートカット「YTDownloader LCへ送る」を作りました(実機の iPhone 14 Pro で確認)。入手は [iOSTweaks の ShareShortcut](https://github.com/mikiyan1978/iOSTweaks/tree/main/YTDownloaderTrollStore/LiveContainer/ShareShortcut) から `YTDownloader LCへ送る.shortcut` を iPhone で開き、「ショートカットを追加」を押します。使い方は、YouTube で動画の共有 → 「YTDownloader LCへ送る」を選ぶだけで、LiveContainer 経由でアプリが開き、画質の選択画面に進みます。

仕組み: 動画 URL を URL エンコードし、`ytdownloaderlc://open?url=…` を作って Base64 にし、`livecontainer://livecontainer-launch?bundle-name=com.mikiyan1978.ytdownloader.lc.app&open-url=<Base64>` を開きます。アプリ側は、もともと `ytdownloaderlc://` を受け取る処理を持っていたので、変更はありません。共有シートに出ないときは、ショートカットの詳細で「共有シートに表示」をオンにします。

### なぜ共有メニューから直接は開けないのか
LiveContainer 自身は共有シートの拡張機能を持っており、共有した URL を LiveContainer に渡せます。ところが、LiveContainer は `https`/`http` の URL を、ゲストアプリの URL スキームでは探さず、自分の Web 表示で開く作りです(ソースを読んで確認)。ゲストアプリを URL で指定して開くには `livecontainer://livecontainer-launch?bundle-name=…&open-url=<Base64>` を、ショートカットなどで組み立てる必要があり、利用者の手間が大きいので採用しませんでした。ゲストアプリの拡張機能も、LiveContainer の中では登録できません。

---

---

## 結論

| 項目 | 結果(実機で確認) |
|---|---|
| Python 3.13 をアプリ内で動かす | **できる**(JITなしのLiveContainerで) |
| yt-dlp でYouTubeの情報取得 | **できる**(最大2160pのURLまで。JSの実行環境は不要だった) |
| 映像・音声を別々にダウンロード | **できる**(約59MBを約5.6MB/秒、完全なサイズで完了) |
| 映像+音声を1つのmp4に結合 | **できる**(AVFoundation、再エンコードなし、約59MB+14MBで3秒弱) |
| 最高解像度(2160p)の保存 | **できる**。ただし2160pはAV1で、**A16ではデコードできず、アプリ内では再生できない** |
| H.264 の再生 | **できる**(720p〜1080p、動画による。ハードウェアデコードで650fps相当) |

確認した動画は数本だけです。年齢制限・ライブ・プレイリスト・メンバー限定は未確認で、YouTube側の仕様は時期で変わります。

---

## なぜ、そのままでは動かないのか

TrollStore版は、子プロセスとして次を `posix_spawn` で起動する作りです。TrollStore専用の権限(`platform-application`、`com.apple.private.security.no-sandbox`)があって初めて成り立ちます。

| 起動している実体 | 役割 |
|---|---|
| Python 3.13 + yt-dlp | 情報取得・ダウンロード |
| Node 22(bgutilサーバー) | PO tokenの生成 |
| aria2c | 並列ダウンロード |
| ffmpeg / ffprobe | 結合・音声抽出・字幕・SponsorBlock |

LiveContainerの中のアプリは、他のプログラムを起動できません(Pythonの`subprocess`も`OSError`でブロックされる)。そのため、全部をアプリの中(同一プロセス)に置き換える必要があります。

---

## 置き換えの設計

| 今 | LiveContainer版 | 状態 |
|---|---|---|
| Python + yt-dlp | PythonをアプリにC APIで組み込む。yt-dlpは純Python | 実機で動作 |
| aria2c | 廃止。yt-dlp内蔵のダウンロードで足りた | 実機で動作 |
| ffmpeg(結合) | AVFoundation(`AVMutableComposition` + `AVAssetExportSession`、パススルー) | 実機で動作 |
| ffmpeg(WebM変換・字幕埋め込み・SponsorBlock切り出し・音声形式変換) | 代替なし。LC 版では対応しない(音声のみは M4A のまま保存) | 未対応(LC 版の仕様) |
| Node + bgutil(PO token) | 不要だった(下記) | 確認した範囲では不要 |

---

## 実験アプリの作り方(要点)

### 入手するもの(いずれも公式)
- **Python本体**: [BeeWare Python-Apple-support](https://github.com/beeware/Python-Apple-support/releases) の `Python-3.13-iOS-support.<ビルド>.tar.gz`(`Python.xcframework`が入っている)。ダウンロード後にSHA256をリリースの値と照合する。
- **yt-dlp**: PyPIの純Python wheel(`py3-none-any`)。展開してアプリに入れるだけ。
- **certifi**: iOSのPythonは既定のCA証明書パスを持たないので、PyPIのwheelを展開し、`SSL_CERT_FILE`で`cacert.pem`を指す。

### アプリの構成(ビルドはXcodeのclang直接でよい。Xcodeプロジェクトは不要)
```
PoC.app/
  PoC                      # 実行ファイル(ObjC、Pythonを起動する)
  Frameworks/Python.framework
  Frameworks/<モジュール>.framework   # 拡張モジュール(.so)をframework化したもの
  python/lib/python3.13/ ...          # 標準ライブラリ + lib-dynload
  app_packages/ yt_dlp, certifi
  app/ *.py                           # 自分のスクリプト
```

### 肝になった点
1. **拡張モジュール(`.so`)は、framework化して`.fwork`で解決する**。公式の`install_dylib`(`Python.xcframework/build/utils.sh`)と同じ手順: `.so`を`Frameworks/<名前>.framework/<名前>`へ移し、元の場所に`.fwork`(中身は移した先の相対パス)を置く。
2. **`.fwork`の解決は`dirname(sys.executable)`が基準**。`PyConfig.executable`を、アプリ本体の実行ファイルのパスに設定する(これを忘れるとimportに失敗する)。
3. 初期化は`PyPreConfig`/`PyConfig`の**Isolated**を使い、`home`と`module_search_paths`を明示する。Pythonスレッドは**スタックを大きめ(16MB)**にして別スレッドで動かす。
4. **LiveContainerは、アプリに入れるときに、自分の証明書で全体を再署名する**ので、未署名のIPAのまま取り込める。

### アプリ内蔵のPythonモジュール(`PyImport_AppendInittab`)
ffmpegの代わりに、ObjCで書いた関数をPythonから呼べるようにした(`ctypes`は使わない)。
- `ytdhost.merge(video_mp4, audio_m4a, out_mp4)`: AVFoundationでパススルー結合。
- `ytdhost.probe(path)`: ファイルを開き、先頭60フレームを実際にデコードして、成否と速度を返す。
- `ytdhost.hwdecode()`: H.264/HEVC/AV1/VP9のハードウェアデコード可否(`VTIsHardwareDecodeSupported`)。

リンクするframework: `UIKit Foundation CoreGraphics AVFoundation CoreMedia VideoToolbox CoreVideo`。

---

## 詰まった点と対処

### 1. HLS(m3u8)のフォーマットを選ぶと結合できない
`ext`が`mp4`でも、`protocol`が`m3u8_native`のものは、中身が**MPEG-TS**(先頭が`0x47`)で、AVFoundationでmp4として結合できない(`no video track`)。**`protocol == "https"` だけを候補にする**。

### 2. ファイル名に`.part`を含めない
ダウンロード先を`<id>.v.part.mp4`のようにすると、yt-dlpの一時ファイル(`.part`)と衝突しうる。一時ファイルの名前に`.part`を入れない。

### 3. 映像と音声は別ファイルで来る
2160p・1080p・720p(H.264)は、映像だけのDASH(fMP4)と、音声だけ(m4a)で別々に来る。1本にしたければ結合が必須(下の「結合」)。

### 4. JSの実行環境(deno/node)は、確認した範囲では不要
yt-dlpは警告(`No supported JavaScript runtime could be found`)を出すが、既定のクライアントで最大2160pのURLが取れ、ダウンロードも最後まで通った(8MBを1回で取るような直接の範囲指定は途中で打ち切られるが、yt-dlpの分割ダウンロードは問題なし)。将来、動画や時期によっては必要になる可能性がある。

### 5. 結合(AVFoundationのパススルー)
- 映像のトラックと音声のトラックを`AVMutableComposition`に入れ、`AVAssetExportPresetPassthrough`、`AVFileTypeMPEG4`で書き出す。出力は通常のmp4(`ftyp`→`moov`→`mdat`)で、映像と音声の両トラックが入る。
- **音声はAAC(m4a)のみ**(WebMのOpusは結合できない)。
- **途中までのファイル(切り取ったもの)は、`Invalid sample`で結合できない**。結合の検証は完全なファイルで行う。

---

## 画質の上限と再生の可否(実測)

### 取得できる最大
JSなしでも、既定のクライアントで**1080p・1440p・2160pのURL**が取れた(4Kと称する動画12本で確認)。ただし、1440pと2160pは、多くが**AV1とVP9**(H.264はなし)。

### 端末のデコード(iPhone 14 Pro / A16)
| 形式 | ハードウェアデコード | 実際の結果 |
|---|---|---|
| H.264(avc1) | あり | **デコード成功**(720p〜1080p、650fps相当) |
| HEVC | あり | (YouTubeからは来ない) |
| **AV1**(mp4) | **なし** | ファイルは開けるが、`Cannot Decode`(デコーダがない) |
| **VP9**(mp4) | なし | `NO video track`(開けない) |
| **VP9**(webm) | なし | `NO video track`(AVFoundationが対応しない) |

### 画質の選択方針(案)
- **標準: H.264の最高画質**(動画により720p〜1080p)。アプリ内で再生できる。
- **オプション: 最高解像度(AV1の2160pなど)**。保存はできるが、**iPhone 14 Proでは再生できない**。AV1に対応した他のプレーヤーで再生できるかは、未確認。
- 1080p以上を、アプリ内で再生したい場合は、ソフトウェアデコーダ(ffmpegのdav1dなど)をアプリに組み込む必要があり、大きな作業になる。

---

## 実験の運用のコツ

- **LiveContainer内のPythonスクリプトは、再取り込みなしで差し替えられる**。実行ファイルを変えない限り、`Documents/Applications/<アプリ>.app/app/*.py`を、USB経由で直接上書きすればよい(UTMの記事の「MacからiPhoneのUTMへ直接ファイルを入れる」の`afcclient`と同じ方法)。実行ファイルを変えたときだけ、IPAを入れ直す。
- **ログはアプリが`Documents`に書く**ので、`afcclient`でMacから読む。LiveContainerの`Documents/Data/Application/<UUID>/Documents/`の下に作られる。再取り込みしても、データのUUIDが変わることがあるので、更新日時が新しいフォルダを探す。
- **ログは追記**にしていると、画面には前回の分も続けて出る。終わりは一番下の`=== done ===`。
- **まずiOSシミュレータで検証する**と速い。`ios-arm64_x86_64-simulator`のスライス(Intel Macでは`x86_64`)で、ネットワークも使える。ただし、デコードやJITなしの挙動は、実機と違う(ハードウェアデコードの可否は特に)。
- `afcclient`は、ファイル名に**非ASCII文字(例: 長いダッシュ`–`)**を含むと、`get`/`mv`ができない。保存するファイル名は、ASCIIに寄せるほうが、道具の面では扱いやすい。

---

## 今後の課題

1. ~~使えない機能の項目を隠す~~(LC 1.2 で対応済み)。
2. ~~yt-dlp をアプリの入れ替えなしで更新する~~(LC 1.3.0 で対応済み。実際の GitHub からの取得は、新しい yt-dlp が出るまで未確認)。
3. AV1・VP9 を、アプリ内で再生したい場合の、ソフトウェアデコーダ。
4. 年齢制限・ライブ・プレイリスト・メンバー限定など、確認していない動画の種類。
5. iOS 15 での動作確認(LC 版の最低バージョンは、確認できていないので iOS 16.0 にしている)。

---

## 注意

- 自分が権利を持つ動画、または、ダウンロードが許可されている動画に使ってください。YouTubeの利用規約や、各国の法律に従うのは、使う人の責任です。
- 実験に使った動画は、公開されている無料素材などです。
