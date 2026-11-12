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
- **Windows 11 ARM64も、JITあり(UTM通常版)でインストールしてデスクトップまで到達できた**(iPhone 14 Pro、後述の「5. Windows 11 ARMを入れる」)。ただし非常に重く、メモリ不足でLiveContainerごと落ちることもあった。

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
- 古いWindows(95/98/3.1)はJITなしでも動く可能性がある。Windows 11はJITありで動作を確認したが、実用性は未検証(速度・メモリ)。
- ISOやイメージは自分で用意したもののみ使う。

---

## 5. Windows 11 ARMを入れる(iPhone 14 Pro、JITあり)

結果: **インストール完了、ローカルアカウントで初期設定を終え、デスクトップが表示された**(所要は約4時間強)。以下は、実際に詰まった点と対処。

### 準備
- ISO: [Microsoft公式](https://www.microsoft.com/software-download/windows11arm64)から「Windows 11 Arm64」のISO(約8.1GiB)を入手(エディション・言語の選択と利用条件への同意は自分で行う)。
- iPhoneの空きは約50GB以上あると安心(ISO 8.1GiB + 仮想ディスク)。
- 仮想ディスクは**qcow2(使った分だけ増える形式)**で作る。64GiBのqcow2でも、インストール後の実ファイルは約11.8GiBだった。Macに`qemu-img`がなくても、ヘッダ(マジック`QFI\xfb`、v3、クラスタ64KiB、L1表、refcount表)を仕様どおり書けば手で作れる(今回は最小の空qcow2を自作し、QEMUが開けた)。

### 設定(`config.plist`)
| 項目 | 値 |
|---|---|
| 起動 | UEFI(`UEFIBoot`=true)、JIT |
| CPU | `neoverse-n1`(`max`でも起動したが重い)、**2コア** |
| メモリ | 2048MBで完走。1536MB+JITキャッシュ256MBに下げたが、設定は**次回起動から有効** |
| ディスク | qcow2を**NVMe**接続(Windowsの標準ドライバで認識される) |
| ISO | USBのCD |
| ネットワーク | `e1000e`(ただしWindows側にドライバがなく、OOBEで「ネットワークに接続しましょう」が出た) |
| TPM | なし(iPhoneのUTMでは付けられない) |

### 詰まった点 1: 画面デバイスは `ramfb` ではなく `virtio-ramfb`
- `ramfb`: UEFIの途中で止まり、画面もシリアルも無表示(Alpineでも同じ)。
- 切り分けは、動く構成(Alpine)を基準に、Windowsと同じ構成から**部品を1つずつ戻す**。「画面を外した版」が動き、次に画面デバイスだけ変えた版を試して、`virtio-ramfb`でUEFIが通り、画面にも映った。

### 詰まった点 2: UEFIシェルで止まる → 手動でCDから起動
- 起動順序で何も起動できず、`Shell>`に落ちた(`Press any key to boot from CD or DVD...`を見逃した)。
- `FS0:` → `cd efi\boot` → `bootaa64.efi` で起動し、表示が出たらすぐキーを押す。
- **インストール途中の再起動では、このメッセージが出ても、キーを押さない**(押すとインストーラが最初からやり直しになる)。

### 詰まった点 3: 「メモリが不足しています!」
- UTMが警告を出したが、すぐには強制終了されなかった。**OKを押して待つ**と、セットアップ画面まで進んだ。

### 詰まった点 4: 要件チェック(TPM・メモリ・セキュアブート)
- 「このPCは現在Windows 11のシステム要件を満たしていません」と出た。
- 回避は、セットアップ中に`Shift + F10`でコマンドプロンプトを開き、`HKLM\SYSTEM\Setup\LabConfig`に`BypassTPMCheck`、`BypassSecureBootCheck`、`BypassRAMCheck`(DWORD=1)を追加してから「戻る」で進める方法が一般的。今回は、この画面を通過できたことを画面で確認した(実際に入力したコマンドの詳細は、画面操作中のため記録していない)。
- UTMの画面キーボードのバーに`F9`/`F10`/`F11`と`Pg Up`/`Pg Dn`が出る。`Shift`を押しながら`F10`を押す。

### 詰まった点 5: 初期設定(OOBE)のエラー
- `OOBEREGION`、`OOBESETTINGS`の「問題が発生しました」が出た。ボタン(やり直す/スキップ)が、画面の小ささで見切れて押せないことがあった。
- 対処: `Shift + F10`でコマンドプロンプトを開き、`shutdown /r /t 0`で**再起動**すると、エラーが消えて先に進んだ。再起動の確認画面(「強制的に再起動」)が出たら、それを選ぶ。

### 詰まった点 6: ネットワークがなく先に進めない
- 「ネットワークに接続しましょう」で「次へ」が押せない(アダプタのドライバがない)。
- 対処: `Shift + F10`で`start ms-cxh:localonly`を実行すると、**ローカルアカウントの作成画面**が出た(このビルド26300で動作を確認)。
- ユーザー名・パスワードは自分で決める。画面が小さくて確定ボタンが見えないときは、`Pg Dn`でスクロールしたり、画面を縮小する。
- 続くプライバシー設定は、内容を読んで自分で選ぶ。

### 詰まった点 7: LiveContainerが途中で落ちた(メモリ不足の疑い)
- OOBEの途中で、LiveContainerごと落ちた(クラッシュレポートやJetsamの記録は取れず、原因はメモリ不足と断定できない)。
- 影響: OOBEの入力はやり直しになったが、**qcow2は壊れず、Windowsは再起動できた**。
- 対策: iPhoneを再起動してメモリを空ける、他のアプリを閉じる、充電しながら画面ロックを外して行う。

### 進み具合の測り方(長時間のインストール)
- qcow2の実サイズを、`afcclient`の`ls -l`で定期的に確認する(約1.5〜6MB/秒で増えた)。
- インストールは2段階: 第1段階(ファイルのコピー・展開、約1時間半)→再起動→第2段階(機能・ドライバ、約2時間以上)→再起動→OOBE。
- 画面の進行率は均等ではなく、後半は遅くなる。

### バックアップと検証
- **仮想マシンを、Windowsのシャットダウンで止めてから**、`afcclient`でqcow2をMacへコピーする(約12GiB、数分)。動作中のファイルはコピーしない。
- Macの空きが足りないと危険(約12GiBの空きでは足りず、不要なファイルを消して約20GiBにしてから実施)。
- 取得後、qcow2の内部を調べると、整合性を確認できる: ヘッダの`incompatible_features`が0(ダーティ=異常終了の印がない)、L1/L2表を辿って実データのクラスタ数を数え、**最大オフセットがファイルサイズ以内**であること。ゼロ扱いのクラスタ(エントリの最下位ビット)は、データ領域に数えない。
- 「ファイルサイズが長時間変わらない」のは、Windowsがデータを解放(`discard=unmap`)して領域を再利用しているためと思われる(推測)。

### 注意
- 検証用のWindowsです。要件を回避した環境のため、更新で不具合が出る可能性があります。
- プロダクトキーなし(未認証)のまま使うと機能に制限があります。ライセンスは自分の責任で扱ってください。
