# crate — iPhone で Linux デスクトップを動かす(LiveContainer + StikDebug で動作確認)

関連: [LiveContainer ガイド](./livecontainer-guide.md) / [StikDebug ガイド](./stikdebug-guide.md) / [SideStore ガイド](./sidestore-guide.md) / [Husk(Android)](./husk-android-on-iphone.md)

**crate** は、iPhone / iPad の上で本物の Linux の仮想マシンを動かすオープンソースのアプリです([oneamendez/crate-ios](https://github.com/oneamendez/crate-ios)。エンジンは [oneamendez/crate](https://github.com/oneamendez/crate))。クラウドの VM やリモートデスクトップではなく、端末の中だけで動き、オフラインでも使えます。2026年11月15日に X で紹介された投稿をきっかけに試しました。

---

## 何ができるか(README の記載)

| ディストリビューション | デスクトップ |
|---|---|
| Ubuntu 26.04 LTS | GNOME |
| Debian 13 | KDE Plasma または Xfce |
| Arch Linux ARM | KDE Plasma または Xfce |

- エンジンは Rust 製で、arm64 から arm64 への変換により、ゲストの命令の多くをそのまま実行します。
- ゲストの OpenGL は、virglrenderer・ANGLE・MoltenVK を通して、iPhone の GPU で動きます。
- 要件: **JIT が必要**(App Store では配れない)。アプリの最低 iOS は 18.0。

## このアプリでの確認結果

端末は iPhone 14 Pro(iOS 26.6.2)、**LiveContainer の中**で動かしました(v0.1.3)。

| 項目 | 結果 |
|---|---|
| LiveContainer への取り込み | できた(署名なし IPA を「+」から取り込み) |
| 内蔵 StikJIT | **使えない**(アプリの拡張機能を起動できないため。アプリ自身が設定画面でそう案内する) |
| StikDebug での JIT | **できた**(設定の Method を StikDebug にして、Turn On JIT) |
| メモリ | 「Increased limit: Yes」、上限 4GB |
| Debian 13 + Xfce | インストール(ダウンロード 701MB)、起動、ログインまで成功 |
| ターミナル | コマンドを実行できた(`ls` など)。画面下に Esc / Ctrl / Tab / 矢印のキー列が付く |

## 手順

1. [Releases](https://github.com/oneamendez/crate-ios/releases) から `crate-unsigned.ipa` を取り、`gh release view` の SHA-256 と一致するか確認する。
2. LiveContainer の「+」から IPA を取り込む(LiveContainer が自分の証明書で署名する)。
3. LiveContainer 側で、JIT を有効にして起動する設定にしておく。
4. crate を開き、Settings → JIT の Method を **StikDebug** にする。「Turn on JIT when crate opens」をオンにすると、起動のたびに自動で行われる。
5. Install タブで OS を選ぶ(軽さを優先するなら **Debian 13 Xfce**。KDE や GNOME は重い)。アカウント名とパスワードを決める。
6. Home の「Start」を押す。JIT の丸が灰色のときは、先に「JIT」を押す(StikDebug に切り替わり、`legacy.js` が実行されて戻る)。

## 注意

- 公式の想定は、Mac で署名して直接入れる方法です。LiveContainer の中で動かすのは、公式の手順ではありません。
- アプリを開き直すたびに、JIT の有効化(StikDebug への切り替え)が必要です。
- 確認したのは、Debian 13 + Xfce の起動とターミナルのみです。速さ、GPU の描画、ほかの OS、長時間の使用は、確認していません。
- iPhone 8 Plus(iOS 16.7.16)は、最低 iOS 18.0 を満たさないため、対象外です。
