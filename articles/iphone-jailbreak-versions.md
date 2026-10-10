# iPhone 脱獄対応iOS一覧（8 Plus 〜 18系）

調査日：2026年9月28日 ／ 最新ツール：Dopamine 3.0.10（2026-09-23）

## 早見表

| 機種 | チップ | 初期iOS | 脱獄できる上限 | 主なツール |
|---|---|---|---|---|
| iPhone 8 / 8 Plus / X | A11 | 11.0 | 16.7.16（端末の最終iOS） | Dopamine / palera1n |
| iPhone XS / XS Max / XR | A12 | 12.0 | 18.7.1 | Dopamine |
| iPhone 11 / 11 Pro / 11 Pro Max / SE(第2世代) | A13 | 13.0 | 26.0.1 | Dopamine |
| iPhone 12 mini / 12 / 12 Pro / 12 Pro Max | A14 | 14.1 | 17.3.1 | Dopamine |
| iPhone 13 系 / SE(第3世代) / 14 / 14 Plus | A15 | 15.0（14/14 Plusは16.0） | 17.3.1 ※ | Dopamine 3.0 |
| iPhone 14 Pro / 14 Pro Max | A16 | 16.0 | 17.3.1 | Dopamine |
| iPhone 15 / 15 Plus | A16 | 17.0 | 17.3.1 | Dopamine |
| iPhone 15 Pro / 15 Pro Max | A17 Pro | 17.0 | 17.3.1 | Dopamine |
| iPhone 16 / 16 Plus / 16 Pro / 16 Pro Max / 16e | A18 系 | 18.0（16eは18.3） | なし | — |
| iPhone 17 / Air / 17 Pro / 17 Pro Max | A19 系 | 26.0 | なし | — |
| iPhone 18 Pro / 18 Pro Max（2026年9月発売） | A20 Pro | iOS 27 系 | なし | — |

※ iPhone 13 系は、公式ガイドの機種別ページと Dopamine 3.0 の発表内容が食い違っています。詳細は下の A15 の項を参照してください。

---

## チップ別の詳細（iOSバージョン範囲ごと）

### A11：iPhone 8 / 8 Plus / X

iOS 16.7.x が最終。iPhone 8 Plus 専用ページは取得できなかったため、同じ A11 の iPhone X の表を適用しています。

| iOS | 状況 | 手段 |
|---|---|---|
| 11.0 〜 11.4.1 | 可 | unc0ver |
| 12.0 〜 13.7 | 可 | Amethyst |
| 14.0 〜 14.8.1 | 可 | Taurine |
| 15.0 〜 16.6.1 | 可 | Dopamine（TrollStore 経由） |
| 16.7 〜 16.7.16 | 可 | Dopamine |
| 15.0 以降（全範囲） | 別手段 | palera1n（checkm8・PC 必須・再起動ごとに再実行。A11 はパスコード無効化が必要） |

### A12：iPhone XS / XS Max / XR

iOS 26 非対応（最終は iOS 18.7.10）。

| iOS | 状況 | 手段 |
|---|---|---|
| 12.0 〜 13.7 | 可 | Amethyst |
| 14.0 〜 14.8.1 | 可 | Taurine |
| 15.0 〜 16.6.1 | 可 | Dopamine（TrollStore 経由） |
| 16.7 〜 16.7.2 | 可 | Dopamine |
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 18.7.1 | 可 | Dopamine |
| 18.7.2 〜 18.7.10 | 不可 | ツールなし |

### A13：iPhone 11 / 11 Pro / 11 Pro Max / SE（第2世代）

iOS 26 に対応する最古のチップ世代。表は iPhone 11 のページによります。

| iOS | 状況 | 手段 |
|---|---|---|
| 13.0 〜 13.7 | 可 | Amethyst |
| 14.0 〜 14.8.1 | 可 | Taurine |
| 15.0 〜 16.6.1 | 可 | Dopamine（TrollStore 経由） |
| 16.7 〜 16.7.2 | 可 | Dopamine |
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 18.7.1 | 可 | Dopamine |
| 18.7.2 〜 18.7.8 | 不可 | ツールなし |
| 26.0 〜 26.0.1 | 可 | Dopamine 3.0（momentarius） |
| 26.1 〜 27.0 | 不可 | ツールなし |

### A14：iPhone 12 mini / 12 / 12 Pro / 12 Pro Max

初期 iOS 14.1。iOS 26 まで更新できる端末ですが、脱獄は 17.3.1 までです。

| iOS | 状況 | 手段 |
|---|---|---|
| 14.1 〜 14.8.1 | 可 | Taurine |
| 15.0 〜 16.6.1 | 可 | Dopamine（TrollStore 経由） |
| 16.7 〜 16.7.2 | 可 | Dopamine |
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 17.3.1 | 可 | Dopamine 3.0（Titan） |
| 17.4 〜 26.6.2 | 不可 | ツールなし（SPTM バイパスなし） |

### A15：iPhone 13 mini / 13 / 13 Pro / 13 Pro Max / SE（第3世代）／ iPhone 14 / 14 Plus

初期 iOS：13 系 15.0、SE 第3世代 15.4、14 / 14 Plus 16.0。

| iOS | 状況 | 手段 |
|---|---|---|
| 15.0 〜 16.5 | 可 | Dopamine（TrollStore 経由） |
| 16.5.1 〜 17.3.1 | 可（要確認） | Dopamine 3.0（Titan） |
| 17.4 以降 | 不可 | ツールなし |

Dopamine 3.0 の発表では、A15〜A17 と M2 は iOS 16.5.1〜17.3.1 に対応とされています。一方、iOS Guide の iPhone 13 ページは、16.7〜16.7.2 と 17.0.1 以降を「N/A」、16.5.1〜16.6.1 と 17.0 を nathanlr としています。ページが Dopamine 3.0 に追随していない可能性があるため、実機で試す前に Dopamine の公式配布ページで対応を確認してください。

### A16：iPhone 14 Pro / 14 Pro Max

初期 iOS 16.0。

| iOS | 状況 | 手段 |
|---|---|---|
| 16.0 〜 16.6.1 | 可 | Dopamine（TrollStore 経由） |
| 16.7 〜 16.7.2 | 可 | Dopamine |
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 17.3.1 | 可 | Dopamine |
| 17.4 〜 27.0 | 不可 | ツールなし |

### A16：iPhone 15 / 15 Plus

初期 iOS 17.0。

| iOS | 状況 | 手段 |
|---|---|---|
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 17.3.1 | 可 | Dopamine |
| 17.4 〜 27.0 | 不可 | ツールなし |

A16 の 15 / 15 Plus は、A16 の 14 Pro と同じ対応範囲（Titan）です。専用ページの取得はしていません。

### A17 Pro：iPhone 15 Pro / 15 Pro Max

初期 iOS 17.0。

| iOS | 状況 | 手段 |
|---|---|---|
| 17.0 | 可 | Dopamine（TrollStore 経由） |
| 17.0.1 〜 17.3.1 | 可 | Dopamine |
| 17.4 〜 27.0 | 不可 | ツールなし |

### A18 系：iPhone 16 / 16 Plus / 16 Pro / 16 Pro Max / 16e

初期 iOS 18.0（16e は 18.3）。

| iOS | 状況 | 手段 |
|---|---|---|
| 18.0 〜 26.x 全て | 不可 | 公開された脱獄ツールなし |

### A19 系：iPhone 17 / Air / 17 Pro / 17 Pro Max

初期 iOS 26.0。

| iOS | 状況 | 手段 |
|---|---|---|
| 26.0 以降 全て | 不可 | 公開された脱獄ツールなし |

### A20 系：iPhone 18 Pro / 18 Pro Max（2026年9月発売）

iPhone 18 / 18e は2027年春発売の予定で、未発売です。iPhone Duo（折りたたみ）も同時発表と報じられていますが、機種別の脱獄情報は確認できていません。

| iOS | 状況 | 手段 |
|---|---|---|
| 全バージョン | 不可 | 公開された脱獄ツールなし（iOS 27 は研究段階） |

---

## 共通の注意点

- Apple が署名を止めた古い iOS には通常戻せません。脱獄したい端末は、対応範囲を超える更新をしないでください。
- Dopamine は「semi-untethered」方式で、再起動のたびにアプリから再脱獄します。rootless のため tweak は rootless 対応版が必要です。
- palera1n が使う checkm8 は A11 以前だけが対象で、A12 以降には使えません。
- A12 / A13 向けの BootROM 脆弱性「usbliter8」（2026年6月公開）は概念実証で、単独の脱獄ツールではありません。
- TrollStore が使えるのは iOS 14.0〜16.6.1 と 17.0 までで、iOS 18 以降は非対応です。
- 表中の「TrollStore 経由」は、該当 iOS で TrollStore を使って Dopamine を入れる手順を指します（iOS Guide の表記）。
- 「iOS 18.7.x でオンライン脱獄できる」とうたうサイトは、本ページの公式情報と食い違います。有料の「脱獄」を含め、信用しないでください。

---

出典：iOS Guide（機種別 Firmware Selection：iPhone X / XS / XR / 11 / 12 / 13 / 14 Pro / 15 Pro）、 The Apple Wiki - Dopamine、 iClarified Jailbreak Guide、 iClarified - Dopamine 3.0、 MacRumors - iPhone 18 Pro。

機種ごとの専用ページを直接確認していない箇所（iPhone 8 Plus、15 / 15 Plus、SE 各世代、16e）は、同じチップの機種の表から補っています。脱獄の可否は変わり得るため、実行前に最新情報を確認してください。
