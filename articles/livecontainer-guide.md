# LiveContainer 完全インストール・ユーザーズガイド

脱獄不要で、無料Apple IDの「3アプリまで」制限を実質回避しながら複数のiOSアプリをサイドロードできるツール。SideStoreと組み合わせて使うのが定番。

公式サイト: [livecontainer.site](https://livecontainer.site) ／ GitHub: `LiveContainer/LiveContainer`

> ⚠️ **2026年10月時点の重要な注意**: GitHub上の公式リポジトリ `LiveContainer/LiveContainer` に、通常のWebアクセスおよびAPI経由でアクセスできない状態を確認しています（検索結果には存在するが、直接アクセスは404）。削除ではなく一時的な非表示/制限の可能性がありますが、**配布元は必ず公式サイトか信頼できるコミュニティ(SideStore公式Discordなど)経由で確認し、素性不明の「野良プリビルドIPA」はインストールしないでください**。一部のフォークには、コンテナ内のデータを読み取れてしまう不正なビルドが混ざっているとの指摘もあります。

> ⚠️ **iOS 26.4以降の既知の問題**: SideStoreとLiveContainerが使うアプリ更新(リフレッシュ)の仕組みが、**iOS 26.4のアップデートで一部壊れている**との報告があります。本ガイドの手順は iOS 15.0〜26.3 で動作確認されたもの。**iOS 26.6.2のような26.4以降のバージョンでは、途中のリフレッシュ手順でつまずく可能性があります**。実行前に最新の対応状況を[SideStore Discord](https://dis.sidestore.io)で確認することを強く推奨します。

---

## LiveContainerとは

iOSアプリを独立した「コンテナ」内で動かすランチャーアプリ。普通にサイドロードすると1アプリ=1スロット消費しますが、LiveContainer自体が1スロットを使い、その中にいくつでもIPAをインポートして起動できます。「LiveContainer + SideStore」という統合版を使うと、SideStore機能を保持しつつ2つの空きスロットも確保できます。

---

## 必要なもの

### パソコン側(初回のみ)
- iloader
- iTunes(**Windowsのみ**必要)

### iPhone/iPad側
- LocalDevVPN
- パスコード設定済み、iOS 15.0以降

> この手順は、既存のSideStore / LiveContainerがインストールされていない状態を前提としています。既にある場合は、事前にアンインストールしてから進めてください。

---

## STEP 1: iloaderでSideStoreを導入

1. **iloader** を起動
2. 「Add Account +」から **Apple Accountでサインイン**
3. iPhone/iPadをパソコンにUSB接続(「Refresh Devices」が必要な場合あり)
4. 「Devices」欄で対象端末を選択
5. 「Installers」欄で **「SideStore (Stable)」** を選択してインストール

---

## STEP 2: SideStoreのセットアップ

1. **設定** → **一般** → **VPNとデバイス管理**
2. iloaderで使ったApple Accountのメールアドレスをタップ → **デベロッパを信頼**
3. 設定トップに戻り **プライバシーとセキュリティ** を開く
4. **セキュリティ** 欄の **Developer Mode** をON(案内に従う。iOS 16.0以降)
5. 再起動後、**LocalDevVPN** を開いて **Connect**
6. 「許可」をタップしてパスコード入力
7. **SideStore** を開き「My Apps」→ SideStoreの横の **「7 DAYS」** をタップ
8. iloaderで使ったのと**同じApple Accountでログイン**

ここまでで通常のSideStoreとして機能します(本体含め3スロット中、あと2つ追加インストール可)。LiveContainerと組み合わせてさらに拡張したい場合は続行してください。

---

## STEP 3: LiveContainer + SideStore に切り替える

SideStore機能を保持したまま、LiveContainer内で無制限にアプリを動かせるようにします。

1. iPhone/iPad上で **「LiveContainer + SideStore」の最新IPA** をダウンロード(Stable推奨。Nightlyは最新機能があるが不安定なので初心者非推奨)
2. SideStoreの「My Apps」で左上の **「＋」** をタップし、ダウンロードしたIPAを選択してインストール
3. 「Keep App Extensions (Use Main Profile)」を選んでインストール完了を待つ → **LiveContainer** を開く
4. 左上の **SideStoreアイコン** をタップ → ペアリングファイルの選択を求められたら、**「On My iPhone/iPad」→「SideStore」→ `ALTPairingFile.mobiledevicepairing`** を選択
5. 「My Apps」の **「7 DAYS」(LiveContainer横)** をタップ
6. 同じApple Accountでログインし、リフレッシュ完了を待つ
7. **単体のSideStoreアプリを削除**(LiveContainer内蔵のもので代替される)
8. LiveContainerを一度閉じて再度開く → **設定** → **「Import Certificate From SideStore」** → OK
9. **「JIT-Less Mode Diagnose」** → **「Test JIT-Less Mode」** を実行し、テストが通ることを確認

これで、LiveContainer内には**制限なくアプリを追加**できるようになります。SideStoreで直接入れたアプリのみが3アプリ制限にカウントされ、LiveContainer内のアプリはカウントされません。

---

## 日常の運用

- サイドロードしたアプリは**7日ごとに要リフレッシュ**(期限切れで使えなくなる)
- LocalDevVPNをONにしておけばバックグラウンド自動更新される
- Shortcutsアプリのオートメーションでリフレッシュ自体を自動化することも可能(定期実行のショートカットを組む)
- LiveContainer(Nightly)ではアプリ内から直接AltStore形式のソースを追加することも可能

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| LiveContainer+SideStore特有の不具合 | 通常のSideStoreサポートではなく、**LiveContainerのGitHub Issues**か、SideStore Discordの **#support-forum** チャンネルへ(通常のSideStoreチャンネルでは対応不可と明記されている) |
| iOS 26.4以降でリフレッシュがうまくいかない | 既知の問題。最新の回避策はSideStore Discordで確認を |
| 証明書エラーで再インストールを求められる | iloaderで既存証明書をRevoke(失効)してから再試行 |

---

## 出典・クレジット

本ガイドは以下の一次情報を基に作成しています:
- [SideStore公式ドキュメント](https://docs.sidestore.io)
- [LiveContainer公式サイト](https://livecontainer.site)
- コミュニティガイド「[Installing LiveContainer+SideStore from start to finish](https://gist.github.com/sinceohsix/688637ac04695d1ff38f844acc8ba7f3)」(r/sideloaded, 2026年3月30日版、★225)

ツール自体はコミュニティの有志が無償で開発・維持しているものです。不具合報告や要望は各GitHub Issueへ、敬意を持って投稿しましょう。

手順・対応状況は頻繁に変わるため、実行前に必ず上記リンク先の最新版を確認してください。
