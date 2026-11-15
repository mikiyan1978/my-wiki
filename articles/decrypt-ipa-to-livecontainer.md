# 脱獄機で復号した IPA を、LiveContainer に入れる(Spotify の例)

脱獄できる iPhone(iPhone 8 Plus、iOS 16.7.16)で、App Store のアプリを復号し、非脱獄の iPhone(iPhone 14 Pro、iOS 26.6.2)の LiveContainer で動かす手順です。Spotify(9.1.98)で、**起動、ログイン、再生、Tweak の動作**まで、実機で確認しました。自分の端末で、自分が入れたアプリを試すための手順です。復号した IPA を、配布や公開しないでください。

## 前提
- 脱獄機に、`frida-server` が動いている(Frida 17 系)。Mac に、同じ系統の Python の `frida`。
- 脱獄機に、復号したいアプリが入っていて、起動できる。
- 非脱獄の端末に、LiveContainer が入っている。

## 手順
1. **暗号化の範囲を調べる。** 本体の先頭を取り出して、`otool -l` で `LC_ENCRYPTION_INFO_64` を見る。`cryptid 1` なら暗号化されている。`cryptoff` と `cryptsize` が、暗号化されている範囲。Spotify は、本体の 4096 バイトだけが対象だった(フレームワークは暗号化なし)。拡張機能(`PlugIns/*.appex`)も、暗号化されている。
2. **アプリ本体を、Mac にコピーする。** 脱獄機の `/var/containers/Bundle/Application/<UUID>/<アプリ>.app` を、`scp -r` でコピーする。
3. **動いているプロセスから、復号済みの領域を取り出す。** アプリを、脱獄機で手動で起動し(Frida の `spawn` は使わない。後述)、PID を調べて、`attach` する。`Process.getModuleByName(<本体>).base.add(cryptoff).readByteArray(cryptsize)` で取り出す。メモリ上では、すでに復号されている。
4. **本体のファイルに書き戻す。** `cryptoff` の位置に、取り出した内容を書き、`LC_ENCRYPTION_INFO_64` の `cryptid` を 0 にする(`otool -l` で確認)。
5. **IPA にする。** `Payload/<アプリ>.app` に入れて zip する。拡張機能は、LiveContainer の中では使えないので、`PlugIns` を外す。
6. **LiveContainer に入れる。** IPA を、非脱獄の端末に送り、LiveContainer の「+」から取り込む。署名は、LiveContainer が行う。

## 注意点
- **Frida の `spawn` は使わない。** この端末では、起動したアプリが、数秒で OS に終了される。手動で起動して、`attach` する。`script.unload()` も使わない(`detach` だけにする)。
- **frida-server を、外から接続できるようにする。** 標準の起動は、`0.0.0.0` で待ち受けない。`launchctl unload` してから `frida-server -l 0.0.0.0:27042 -D` で起動する。Mac 側の `frida` は、サーバーと同じ系統にする(少し古い Python 版でも、つながった)。
- 本体が `arm64` だけでも、`arm64e` の端末(iPhone 14 Pro)で動いた。
- 動く保証はない。アプリが、再署名や、権限の違いを検知すると、起動しない。権限に依存する機能(プッシュ通知など)は、動かないことがある。

## Tweak を入れる
Theos で作った Tweak は、`CydiaSubstrate` の参照の書き換えが要る。[LC Tweaks 導入](youtube-tweaks-livecontainer) の「Substrate 互換ライブラリ」を見る。Spotify の Tweak は、導入アプリから入れられる。
