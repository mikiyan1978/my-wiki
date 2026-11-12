# YTDownloaderをLiveContainerで動かす(Python+yt-dlpをアプリ内に組み込む実験)

関連: [YTDownloader TS(TrollStore版)](./ytdownloader-trollstore-guide.md) / [LiveContainer ガイド](./livecontainer-guide.md) / [UTM on iPhone](./utm-on-iphone-guide.md)

TrollStore版のYTDownloaderは、LiveContainerに入れても動きません。そこで「外部プログラムを起動せず、すべて同一プロセスの中で動かす」形に作り替えられるかを、実験アプリで確かめた記録です。端末は iPhone 14 Pro(A16)、iOS 26.6.2、LiveContainer(JITなし)。

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
| ffmpeg(WebM変換・字幕埋め込み・SponsorBlock切り出し・音声形式変換) | 代替なし。縮小または別実装 | 未対応 |
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

## 次の課題

1. 実アプリ(YTDownloader)の機能を、サブプロセス依存から、アプリ内の実行に置き換える(取得・ダウンロード・結合の流れは、実験アプリで確認済み)。
2. ffmpegに頼っていた機能(WebMの変換、字幕の埋め込み、SponsorBlockの切り出し、音声の形式変換)の代替。
3. AV1・VP9を、アプリ内で再生したい場合の、ソフトウェアデコーダ。
4. 年齢制限・ライブ・プレイリストなど、確認していない動画の種類。

---

## 注意

- 自分が権利を持つ動画、または、ダウンロードが許可されている動画に使ってください。YouTubeの利用規約や、各国の法律に従うのは、使う人の責任です。
- 実験に使った動画は、公開されている無料素材などです。
