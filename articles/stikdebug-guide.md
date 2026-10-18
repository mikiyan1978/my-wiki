# StikDebug 完全インストール・ユーザーズガイド

脱獄なしでJIT(Just-In-Timeコンパイル)を有効化できる、**オンデバイス完結型のデバッガ／JITイネーブラ**。旧名StikJIT。[idevice](https://github.com/jkcoxson/idevice)というライブラリを使い、初回のペアリングファイル準備以降は**パソコンなしで**JITの有効化・無効化・アプリ管理ができるのが最大の特徴。DolphiniOS・UTM・PojavLauncherなど、フルスピードで動かすのにJITが必須なエミュレーター系アプリを使う場合の定番ツール。

公式サイト: [stikdebug.xyz](https://stikdebug.xyz/) ／ GitHub Organization: [github.com/StikDebug](https://github.com/StikDebug) ／ 本体リポジトリ: [StikDebug/StikDebug](https://github.com/StikDebug/StikDebug)（AGPL-3.0, 2.8k★）

> ⚠️ **App Storeからは削除済み**。公式サイトまたはAltSource経由の配布のみが正規の入手方法です（後述）。

---

## 対応状況

| iOSバージョン | ステータス | 備考 |
|---|---|---|
| 1.0 〜 17.3.x | **非対応** | 接続プロトコルが異なるため |
| 17.4 〜 18.x | **完全対応** | 安定動作 |
| 26.0以降 | **対応**(制限あり) | 対応アプリ側のアップデートが必要な場合あり |

iPhone 14 Pro・iOS 26.6.2のような26系端末でも基本的に使えますが、「対応アプリの開発者側がiOS 26向けに対応済みか」に左右されるため、個々のアプリ(特にエミュレーター)ごとに動作報告を確認するのが安全です。

---

## 主な機能

- **JIT**: `get-task-allow`エンタイトルメントを持つサイドロード済みアプリに対してJITコンパイルを有効化
- **アプリ起動**: 端末にインストール済みの任意のアプリを起動
- **コンソール**: アプリ/システムのログをリアルタイム表示
- **スクリプト**: 自動化スクリプトの管理(主にiOS 26以降のJIT用)
- **アプリ有効期限**: 各アプリの失効タイミングを確認し、プロファイルのインストール/削除が可能
- **デバイス情報**: 詳細なハードウェア/デバイス情報を表示
- **プロセス**: 実行中のアプリ/プロセスを確認・強制終了
- **位置情報シミュレーター**: GPS位置を偽装

---

## 必要なもの

- iPhone / iPad(**iOS 17.4以降**。26.0以降も対応だがアプリ側の対応状況に注意)
- **SideStore / AltStore など、既にサイドロードが使える状態**(StikDebug自体のインストールとアプリのリフレッシュに必要)
- **LocalDevVPN**(ループバックVPN。SideStoreと共通で使うアプリ)
- ペアリングファイル作成用のパソコン(**初回のみ**。以下いずれか)
  - **Mac**: [iloader for Mac](https://docs.sidestore.io/docs/installation/prerequisites)をインストール
  - **Windows**: iTunesをインストール → iloaderインストーラ(exe/msi)を実行
  - **Linux**: `usbmuxd`(ディストリに同梱されている場合あり) → ディストリ向けのiloader(Debian/Ubuntu, Fedora/openSUSE等)をインストール

> StikDebugは**SideStoreの導入が前提**です。先に [SideStore 完全インストール・ユーザーズガイド](./sidestore-guide.md) を済ませておいてください。

---

## STEP 1: ペアリングファイルを用意する

ペアリングファイルは「このパソコン/ツールがこの端末と通信してよい」ことを示す、**端末固有・時限付き**のファイルです。StikDebugが正常に動作するために必須です。

1. iPhone/iPadをUSBケーブルでパソコンに接続(ワイヤレスで認識される場合はそれでも可)
2. iloaderを起動し、**「Manage Pairing File」**をクリック
3. 一覧の中から**「StikDebug」**(および他に使いたいアプリがあれば一緒に)の右側にある**「Place」**をクリック
4. 緑色で **「Pairing file placed successfully!」** と表示されれば完了

> ⚠️ **重要な注意**: 端末のOSをアップデートしたり初期化すると、ペアリングファイルは**失効**し、再度この手順が必要になります。これは**不定期なタイミングでも**起こり得ます(Apple側の仕様によるもので、開発側ではどうにもできない既知の制約)。

### オンデバイスでペアリングファイルを再生成する(StikPair)

毎回パソコンを使うのが面倒な場合、StikDebugチームが提供する **[StikPair](https://github.com/StikDebug/StikPair)**(SwiftUI製、オンデバイスでペアリングファイルを作成するアプリ)を使えば、2回目以降はパソコンなしで再ペアリングできる場合があります。ただし初回のペアリングと、信頼関係が完全に失われた場合の復旧には、結局パソコン(iloader)が必要になることが多い点に注意してください。

---

## STEP 2: StikDebugをインストールする

1. SideStoreを開き、**StikDebugのAltSourceを追加**する(SideStore公式ドキュメントまたはStikDebug公式サイトに掲載のURLを使用)
2. ソース一覧からStikDebugを探し、**「FREE」**を選択してインストール
3. 通常のサイドロードアプリと同様、**7日ごとのリフレッシュ対象**になります(SideStoreのMy Appsから管理)

---

## STEP 3: ペアリングファイルをStikDebugにインポートする

1. StikDebugを開き、**設定(Settings)**を開く
2. 「Import Pairing File」的な項目から、以下のいずれかを選択してインポート
   - iloaderで直接「Place」した場合: 自動的に認識される(端末の共有ペアリングストアに置かれるため)
   - SideStore経由で取得したものを使う場合: **「On My iPhone/iPad」→「SideStore」→ `ALTPairingFile.mobiledevicepairing`**

---

## STEP 4: TXM / Non-TXMの確認とJITスクリプト設定(iOS 26以降で重要)

1. StikDebugの**設定ページ**を開き、一番下までスクロール
2. **「TXM」** または **「Non-TXM」** のどちらかが表示されているはずです

### Non-TXMの場合
特別な追加設定は不要。STEP 5(JITの使用)に進んでください。

### TXMの場合(iOS 26以降の一部端末で表示)
以下の4ステップが追加で必要です。

1. StikDebugの**設定**で、**「Silent Audio」**と**「Always Run Scripts」**を両方ONにする
2. LiveContainer内で、JITが必要なアプリを**ダブルタップ**
3. **「Launch with JIT」**がONになっていることを確認し、**「Select」**をタップ
4. **「On My iPhone/iPad」→「StikDebug」→ scripts フォルダ**内から、対応する `.js` スクリプトを選択
   - **Flycast(iOS 26フォーク版)・UTM・DolphiniOS・ARMSX2** → `legacy.js`
   - **それ以外のアプリ** → `universal.js`

> ⚠️ 全てのアプリがiOS 26以降でJITに対応しているわけではありません。事前に各アプリの開発元で対応状況を確認してください。

---

## STEP 5: JITを有効化して使う

1. **LocalDevVPN**を開き、VPNをON(Connect)にする
2. StikDebugを開き、**「Enable JIT」**ボタンをタップ
3. 一覧からJITを有効化したい**サイドロード済みアプリ**を選択

LiveContainerと組み合わせている場合は、LocalDevVPNをONにした状態で、LiveContainer側の対象アプリ横にある**「Run」**をタップするだけでJIT付きで起動できます。

---

## トラブルシューティング

| 症状 | 原因・対処 |
|---|---|
| 「Connection dropped」またはループバック関連のエラー | iOSバージョンの対応状況を確認。ベータ版OSでは警告が出ることがある |
| Heartbeatエラー | LocalDevVPNがONになっているか、Wi-Fiに接続されているかを確認。ペアリングファイルの問題であることも多い |
| ペアリングファイル関連のエラー全般 | 端末のロックを解除し「このコンピュータを信頼」した状態で、ペアリングファイルを一から取得し直す |
| それでも解決しない | ログ・スクリーンショットを添えて[StikDebug公式Discord](https://stikdebug.xyz/)で質問 |

---

## 自分でビルドする場合(上級者向け)

StikDebugを自前のApple ID/証明書でビルドしたい場合:

```bash
git clone https://github.com/StikDebug/StikDebug.git
cd StikDebug
```

1. `StikDebug.xcodeproj` をXcodeで開く(**macOS、Xcode 16以降推奨。iOS 26以降を狙うならXcode 26以降が望ましい**)
2. StikDebugターゲット → **Signing & Capabilities** → 自分のApple IDでサインイン(無料/有料どちらでも可)
3. **Bundle Identifier**をユニークなもの(例: `com.yourname.StikDebug`)に変更
4. 接続した実機を選択し、**Cmd + R**でビルド&実行
5. 初回起動時、**設定 → 一般 → VPNとデバイス管理**で証明書を信頼

ビルド後は通常版と同じく、STEP 1以降のペアリングファイル設定が必要です。

---

## StikDebugファミリーの構成

| プロジェクト | 役割 |
|---|---|
| **StikDebug** | 本体。オンデバイスのデバッガ/JITイネーブラ兼開発者ツールボックス |
| **StikPair** | オンデバイスでペアリングファイルを作成するSwiftUIアプリ |
| **StikJIT** | 他プロセスに対してRSDトンネル経由でJITを有効化するためのiOS XCFramework(StikDebugベース、自作アプリへの組み込み用) |

---

## 出典・クレジット

- [StikDebug本体 README](https://github.com/StikDebug/StikDebug)
- [StikDebug-Guide(ペアリングファイル手順の公式ガイド)](https://github.com/StikDebug/StikDebug-Guide)
- [StikDebug公式サイト](https://stikdebug.xyz/)
- コミュニティウォークスルー「[iOS 26 Sideloading and JIT Complete Walkthrough](https://docvault.celloserenity.dev/walkthroughs/LiveContainer-iOS-26-JIT/)」(CelloSerenity, 2026年版)

ツール自体はコミュニティの有志(StephenDev氏ほか)が無償で開発・維持しているものです。本ガイドの内容は2026年10月時点のもの。手順・対応状況・配布URLは頻繁に変わるため、実行前に必ず上記リンク先の最新版を確認してください。
