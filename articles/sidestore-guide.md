# SideStore 完全インストール・ユーザーズガイド

脱獄不要でiOSにアプリをサイドロードできる、**AltStoreの派生(フォーク)**。最大の特徴は「**初回セットアップ時だけ**パソコンが必要で、その後はWi-Fiだけで7日ごとの自動更新が完結する」こと。

公式: [sidestore.io](https://sidestore.io) ／ ドキュメント: [docs.sidestore.io](https://docs.sidestore.io) ／ [GitHub](https://github.com/SideStore/SideStore)

---

## 必要なもの

- パスコード設定済みの iPhone / iPad / iPod touch(**iOS/iPadOS 15.0以降**)
- パソコン(Windows 8以降 / macOS High Sierra以降 / 最新のLinux / 未登録のChromebook)— **初回インストール時のみ**
- Apple Account(Apple ID)
- **Wi-Fi接続**(モバイル回線では不可。初回セットアップ・7日ごとの更新とも必須)
- 多少の時間的余裕(トラブルシューティング込みで)

---

## STEP 1: 端末側の事前準備(LocalDevVPN)

SideStoreは端末内で完結する専用VPNを使って内部サービスと通信する仕組み。

1. App Store または AltStore PALのソースから **LocalDevVPN** アプリをインストール
2. アプリを開き「Connect」をタップ
3. 「VPN構成を許可しますか」と聞かれたら **「許可」** → パスコード入力

> このVPNは、SideStoreでアプリのインストール・更新・リフレッシュを行うたびに**毎回ONにしておく必要**があります。

---

## STEP 2: パソコンで iloader を使ってSideStoreを導入

1. お使いのOS(macOS / Windows / Linux / Chromebook)に対応した **iloader** をダウンロード・インストール
2. iPhoneをUSBケーブルでパソコンに接続し、「このコンピュータを信頼しますか」が出たら信頼してパスコード入力
3. iloaderを起動
4. **Apple Accountでサインイン**(端末に紐づくApple IDである必要はない。大文字小文字を区別するので注意)
5. 接続した端末を選択
6. **「Install SideStore (Stable)」を選択**

> iloader自体の不具合に関する質問は、SideStore公式ではなく **idevice Discord** サーバーのサポート範囲です。

---

## STEP 3: iPhone側で信頼・Developer Mode有効化

お使いのiOSバージョンによって手順が少し異なります。

### iOS/iPadOS 15.0〜17.7.7 共通の流れ
1. **設定** → **一般** → **VPNとデバイス管理**
2. 「Developer App」欄にある、自分のApple Account名のプロファイルを選択
3. **「"〇〇"を信頼」** → **「許可して再起動」**
4. パスコードを入力して確定
5. **設定** → **プライバシーとセキュリティ** を開き、一番下までスクロール
6. **「Developer Mode」をON** → 端末が再起動する

### iOS/iPadOS 18.0以降
基本的な流れは同じですが、画面文言が一部異なる場合があります(「VPNとデバイス管理」の位置や確認ダイアログの表記など)。表示に従って進めてください。

---

## STEP 4: SideStoreの初回セットアップ完了

1. **LocalDevVPN** を開いて「Connect」(VPNをON)
2. **SideStore** を開く
3. iloaderで使ったのと同じ **Apple Accountでサインイン**
4. 「My Apps(マイアプリ)」タブを開く
5. SideStoreの右にある **「7 DAYS」のカウンターをタップ** → 手動リフレッシュが走り、セットアップが完了する
   - 「署名証明書を失効/新規作成しますか」と聞かれたら **「Yes」または「Refresh Now」**
6. 成功すると、通知と共にホーム画面に戻される。数秒でSideStoreが再度使えるようになる

> ⚠️ 端末をアップデート・初期化すると、ペアリングファイルが失効することがあります(Appleの仕様により不定期に起きることも)。その場合は同じ手順でペアリングをやり直してください。

---

## 日常の使い方

### アプリを追加する
- 「ソース(Sources)」からIPAを配布しているリポジトリを追加するか、直接 `.ipa` ファイルを開いてサイドロード
- 公式コミュニティソース: `https://community-apps.sidestore.io/sidecommunity.json`

### 7日ごとの更新
- LocalDevVPNをONにしておけば、SideStoreがバックグラウンドで自動リフレッシュ
- 自動更新が効かない場合は「My Apps」で各アプリの日数カウンターを手動タップ

### JITが必要なアプリ(エミュレータ等)を使う場合
| iOSバージョン | 方法 |
|---|---|
| iOS/iPadOS 16以下 | 「My Apps」でアプリを長押し →「Enable JIT」 |
| iOS/iPadOS 17.4以上 | SideStore 0.6.2以降、または **StikDebug**(旧StikJIT)を使用 |

---

## 無料Apple IDの制限と回避策

| 制限 | 内容 |
|---|---|
| 同時インストール数 | SideStore本体を含め**最大3アプリ**まで |
| 週あたりの新規インストール(App ID)数 | **週10個**まで |
| 証明書の有効期限 | **7日**(自動/手動リフレッシュが必要) |

**回避策:**
- **Apple Developer Program($99/年)**に加入 → 3アプリ制限が外れ、有効期限も365日に延長
- **LiveContainer** を使う(コンテナ内でアプリを動かし、実質的にApp ID枠を消費しない)
- iOS 18 db5 / 18.0.1以下限定で **SparseRestore** エクスプロイトを使い3アプリ制限を回避(App ID週10個制限は回避不可)

---

## よくあるトラブル

| 症状 | 原因・対処 |
|---|---|
| ログインできない・エラーが出る | Anisetteサーバーの一時的なダウンが主な原因。設定アプリ内SideStoreの「Anisette URL」を変更して公式サーバーに切り替える |
| アカウントがロックされる | 古い/共有されすぎているAnisetteサーバーが原因のことが多い。公式サーバーか自前のanisette-v3-serverを使う |
| 初回インストール後、動作がおかしい | 「はじめに一度SideStore自身をリフレッシュしてから他のアプリをサイドロードする」のが推奨。iloaderは別のgroupIDでインストールするため |
| iPhone/iPadにMac/MacBookが身に覚えなく追加される | Anisetteサーバーがログイン支援のために自動生成する**実在しない仮想Mac**。個人情報は送信されないため無害 |
| アプリ更新に「pacer test」と表示される | メジャーバージョン間の互換性テスト用の名残。リリース後も出る場合はSideStoreを再インストールするかバグ報告 |

---

## AltStore/Sideloadlyからの移行

既存のAltStoreやSideloadlyでサイドロードしたアプリは、**元のアプリを削除せずに**同じ(または更新版の)IPAをSideStoreでサイドロードすれば、データを保持したままSideStoreの管理下に移行できる。

---

## まとめ: AltStoreとの違い

| | AltStore (Classic) | SideStore |
|---|---|---|
| 初回セットアップ | PC(AltServer)が必要 | PC(iloader)が必要 |
| 7日ごとの更新 | **PCがWi-Fi上で起動している必要あり** | **PC不要**(LocalDevVPN + Wi-Fiのみ) |
| JIT有効化 | 別途ツールが必要な場合あり | アプリ内 or StikDebug連携 |

日常的にPCをつけっぱなしにしない運用なら、SideStoreの方が圧倒的に扱いやすい。

---

出典: [SideStore公式ドキュメント](https://docs.sidestore.io)（Prerequisites / Install / FAQページ、2026年10月時点の内容）。手順は更新される可能性があるため、実行前に公式ドキュメントの最新版も確認してください。
