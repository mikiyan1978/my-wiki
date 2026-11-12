# iPhoneでUTMを動かす(LiveContainer + JIT / JITなし、実機検証メモ)

脱獄なしのiPhone(iPhone 14 Pro、iOS 26.6.2)で、LiveContainer内のUTMを使ってLinux(Alpine)を起動するまでの記録。**JITなし(UTM SE)** と **JITあり(UTM通常版 + StikDebug)** の両方を試した結果をまとめます。

関連: [LiveContainer ガイド](./livecontainer-guide.md) / [StikDebug ガイド](./stikdebug-guide.md)

---

## 結論(先に)

| 方式 | 起動時間(Alpine) | 必要なもの |
|---|---|---|
| UTM SE(JITなし) | 数分 | LiveContainer(JIT-less用の`.p12`証明書) |
| UTM通常版(JITあり) | **約8秒** | 上記 + StikDebug + LocalDevVPN + `legacy.js` |

- UTM SEは、命令を1つずつ解釈実行するため遅い。軽いLinuxの確認程度。
- Windows 11などの重いOSは、JITありでもメモリ(LiveContainer内の制限)やTPM要件が課題。未検証。

---

## 入手先(公式)

- UTM: [utmapp/UTM](https://github.com/utmapp/UTM/releases)(`UTM-SE.ipa` = JITなし、`UTM.ipa` = JIT必要)。ファイルサイズは公式の記載と一致することを確認する。
- Alpine Linux: [alpinelinux.org](https://alpinelinux.org/)のaarch64「Virtual」版ISO(`alpine-virt-*-aarch64.iso`)。`latest-releases.yaml`のSHA256と照合する。

---

## 1. UTM SE(JITなし)でAlpineを動かす

### つまずいた点と対処

**ISOを選べない(ファイル選択が無反応)**
- UTM SE内の「選択…」でISOをタップしても反応しない。iCloud Driveでも、アプリ自身のDocumentsでも同じ。
- 対処: ファイル選択を使わず、**Macで設定ファイル(`.utm`)を作って、中にISOを入れる**。UTMはDocuments直下の`.utm`パッケージを一覧に出す。

**`.utm`の中身**
- `config.plist`(設定、ConfigurationVersion 4)と`Data/`(ISOなどのディスクイメージ)。
- 構造の根拠はUTMのソース(`Configuration/UTMQemuConfiguration*.swift`、`QEMUConstant*.swift`)。キー名と文字列定数(`Architecture`=`aarch64`、`Target`=`virt`、`ImageType`=`CD`、`Interface`=`USB`/`VirtIO`など)をそこから読んで組み立てた。
- 最小構成: aarch64 / virt / UEFIBoot=true / メモリ512MB / CDドライブ(ISO) / ネットワーク(`virtio-net-pci`、Emulated)。

**ログインできない: `Password:`が2回出て60秒でタイムアウト**
- 症状: `root`を入れるとPasswordが2行出る。パスワードの文字が画面に出る(本来は出ない)。入力した文字の一部が別の読み取り側に取られる。
- 原因(状況証拠): ディスプレイ(VT)の`tty0`と`tty1`の**ログイン待ち受けが2つ同時に動き、入力を取り合う**。
- 対処: **ディスプレイを外し、シリアル端末だけにする**(`Display`を空配列、`Serial`に`Mode: Terminal`、`Target: Auto`を追加)。ISOの起動引数が`console=tty0 console=ttyAMA0`の両方を持つのが背景。シリアルだけにしたら一発でログインできた。

**rootのパスワード(ライブISO)**
- ISOの`alpine-baselayout-data`には`root:::0:::::`(空パスワード)とあるが、実際には空では通らなかった。原因は特定できていない。
- 回避: **`localhost.apkovl.tar.gz`をFAT形式の小さなディスクに入れ、追加ドライブ(VirtIO)としてつなぐ**。Alpineは起動時にapkovlを探して`/etc`を上書きする。中身は`etc/shadow`にrootのパスワード(`openssl passwd -6`で作ったハッシュ)を入れたもの。ライブ環境専用で、電源を切ると消える。
- FAT画像は`hdiutil create -layout NONE -fs MS-DOS -size 8m -volname APKOVL -srcfolder <dir> -format UDRW out.dmg`で作れる(中身は生のFAT)。

---

## 2. UTM通常版(JITあり)でAlpineを動かす

### 前提
- StikDebug導入済みで、ペアリングが通っていること([StikDebug ガイド](./stikdebug-guide.md)の「iOS 26.6.2での実機検証」)。
- LocalDevVPNが接続中。
- LiveContainerのUTMの設定で **「JITで起動」をオン**、「JIT起動スクリプト」に **`legacy.js`**(StikDebug公式の`StikDebug/Scripts/legacy.js`)を指定。StikDebugの **Always Run Scripts** をオン。
- LiveContainerの**マルチタスク(小窓)はオフ**にして通常起動する(小窓のUTMはwatchdogで強制終了された)。

### 「真っ黒のまま」だった時の切り分け(実際の手順)

1. **クラッシュログ**(`idevicecrashreport -e`)を見る → クラッシュ(`SIGKILL`/`0x8BADF00D`)の有無。出ないなら、デバッガに止められている可能性。
2. **UTMのデバッグログ**を有効にする(`config.plist`の`QEMU.DebugLog`=true)。`Data/debug.log`にQEMUの起動引数とイベントが残る。
3. **QEMU内部ログ**を取る。`QEMU.AdditionalArguments`に`-d`と`in_asm,cpu_reset,guest_errors,unimp`、`-D`とログのパスを足す。ゲストが命令を実行しているかが分かる。
4. 今回は「最初の命令ブロック(`0x40000000`)を変換した直後から記録が止まる」ことで、**JIT用メモリ確保の`BRK #0x69`でデバッガに止められている**と判断。`legacy.js`を設定すると直った。

### UEFIなしのカーネル直接起動(診断用に作った構成)

- ISO内の`boot/vmlinuz-virt`は**EFI zboot形式**(先頭が`MZ`、`zimg`の識別子、`gzip`)。QEMUの`-kernel`に渡すには、中のgzipを展開して**生のARM64 `Image`**にする。ヘッダのオフセット8にペイロードのオフセット、12にサイズ(リトルエンディアン)が入っている。展開後、オフセット56に`ARMd`のマジックがあれば正しい。
- `Drive`に`ImageType`=`LinuxKernel`(`Image`)、`LinuxInitrd`(`initramfs-virt`)を追加し、`UEFIBoot`=false。
- 起動引数は`QEMU.AdditionalArguments`に`-append`とその値を入れる。**値にスペースが含まれる場合は、ダブルクォートで囲んで1つの引数にする**(UTMはスペースで分割して渡す。囲まないと残りが余り、QEMUが「Could not open '...': No such file」と、ディスクとして解釈してエラーになる)。
  - 例: `-append` と `"console=ttyAMA0 modules=loop,squashfs,sd-mod,usb-storage,virtio_blk"`
- この構成で起動すると、`(none) login:`まで約8秒。ホスト名が`(none)`と表示されるのは、`/etc/hostname`がない構成のため(無害)。

---

## 3. MacからiPhoneのUTMへ直接ファイルを入れる(iCloud不要)

iCloud Driveは大きなファイル(100MB超)の同期が遅く、iPhone側に`config.plist`が届かないことがあった。代わりに、**`afcclient`(libimobiledevice)でLiveContainerのコンテナに直接コピー**できる。

```bash
# コンテナ内を見る(--documents ではなく --container がこの環境では通った)
printf 'ls Documents/Data/Application\nquit\n' | afcclient -u <UDID> --container com.kdt.livecontainer.<TeamID>

# UTMのDocumentsは次のパス
#   Documents/Data/Application/<UTMのコンテナUUID>/Documents/
# ファイルを入れる/取る
printf 'put -f ./local.iso Documents/Data/Application/<UUID>/Documents/Foo.utm/Data/local.iso\nquit\n' | afcclient ...
printf 'get -f Documents/Data/Application/<UUID>/Documents/Foo.utm/Data/debug.log ./debug.log\nquit\n' | afcclient ...
```

- 使えるコマンド: `ls [-l]` `get` `put` `mkdir` `mv` `rm` `info`。
- UTMのコンテナUUIDは、LiveContainerの`Data/Application`直下のフォルダのうち、`LCContainerInfo.plist`の時刻が取り込んだ時刻に近いもの、またはUTMのログ(`/var/mobile/Containers/Data/Application/.../Documents/Data/Application/<UUID>/`)から分かる。
- 設定ファイルの書き換え(`config.plist`)も直接できる。UTMを閉じて開き直すと反映される。
- `debug.log`の取得で、iPhone上でQEMUが何をしたかをMacから読める。

---

## 4. 注意点

- 無料Apple IDでは、LiveContainerに取り込んだアプリは**3アプリ枠を消費しない**。
- JITは、LiveContainerを起動するたびに有効化の手順が必要になることがある(LiveContainerの「JITで起動」を使うと、StikDebugが自動で動く)。
- 古いWindows(95/98/3.1)はJITなしでも動く可能性があるが、Windows 10/11はJITありでも要件・メモリの面で未検証。
- ISOやイメージは自分で用意したもののみ使う。
