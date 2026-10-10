# Husk — iPhoneでAndroidアプリ・ゲームを動かす

**脱獄不要**でAndroidのAPK/ゲームをiPhone上で実行できるオープンソースアプリ。[github.com/Leviidev/Husk](https://github.com/Leviidev/Husk)（GPL-2.0、2026年10月時点で★1.1k、開発中）。

> 2026年10月9日にX上で正式版リリースが話題になったが、「脱獄」という単語は文脈(脱獄コミュニティでの話題という意味)であり、**Husk自体はJIT(後述)さえ確保できれば脱獄済み端末は不要**。

---

## できること

APKを追加してタップするだけで全画面起動。内部的には2つの動作方式を持つ。

### 1. Translation Layer(推奨・高速)
ゲーム本体の64bit ARMネイティブコードを**そのままiPhoneのCPU上で直接実行**し、Android側のAPI(Cライブラリ、Java呼び出し、OpenGL ES→ANGLE、Vulkan→MoltenVK、サウンド、タッチ、コントローラー)をHuskが肩代わりする方式。Androidそのものは起動しないため数秒で起動し、コードはエミュレートされない。

**対応エンジン:**
- Unity
- Unreal Engine 4(Vulkan経由)
- cocos2d-x
- Godot 3 / 4(GLES2, GLES3, Compatibilityレンダラー)
- SDL2 / SDL3(LÖVE製ゲーム含む)
- GameActivity(Minecraft系)/ NativeActivity
- RockstarのSAN(GTA: San Andreas用独自エンジン)

動作実績: GTA: San Andreas, Minecraft Dungeons, Beach Buggy Racing 2, Geometry Dash(Geode mod読み込み対応)など。

### 2. Emulation(低速・互換性重視)
フル機能のAndroid(LineageOS)をQEMUでエミュレートして丸ごと起動。Translation Layerが対応していないアプリ向け。iOSはアプリにハイパーバイザーを与えないためCPUエミュレーションとなり、動作はかなり重い。

### 制限事項
- APKは**64bit(arm64-v8a)ネイティブコード必須**。32bitのみのAPKはTranslation Layer非対応
- `.apk` に加え `.xapk` `.apkm` `.apks`(分割バンドル)も追加可能
- Javaのみで書かれたネイティブエンジン無しのアプリはTranslation Layer非対応 → Emulationで動かす
- Google Playサービスは「インストール済み・未サインイン」として見える程度。Play Games連携・クラウドセーブ・課金は不可
- 一度に起動できるゲームは1つ。切り替えはアプリ内「Close Husk to Play」から
- セーブデータはゲームページから `.zip` でバックアップ/復元可能(別端末への移行も可)

---

## インストール方法

### 方法A: サイドローダーのソース経由(推奨・自動アップデート)
SideStore、AltStoreなどに以下のソースURLを追加する。

```
https://raw.githubusercontent.com/Leviidev/Husk/main/altsource.json
```

手順: サイドローダーアプリ内 **Sources › ＋ › 上記URLを貼り付け** → Huskが表示されるのでインストール。以後のアップデートもソース経由で通知される。

### 方法B: IPAを直接ダウンロード
[Releases](https://github.com/Leviidev/Husk/releases) から `Husk.ipa` を取得し、SideStore / AltStore / **TrollStore** のいずれかでインストール。IPAは共通の1ファイルで、TrollStoreはHusk本来のentitlementsを維持したまま、他のサイドローダーは自分の署名で再署名する。

### 動作要件
- **iOS 16.0以降**

---

## JIT(実行時コンパイル)の確保方法

Translation Layer・Emulationの両方とも動作に**JIT**が必要で、iOSではJITは通常デバッガ経由でのみ取得できる。Huskはアプリ内(設定 › JIT & Sideload)で複数の取得方法をガイドしてくれる。

| 方法 | 条件 |
|---|---|
| 内蔵StikJIT | iOS 26以降。Husk自身がJITを有効化。iOS 27ではPC不要でSettingsからペアリング可能 |
| StikDebug | Huskと並行してインストールして使用 |
| TrollStore または Dopamine脱獄(「Allow JIT in Apps」) | **iOS 26より前のバージョンのみ**有効。iOS 26以降はデバッガ経由以外のJIT取得手段が塞がれている |

> 本Wikiの[iPhone 脱獄対応iOS一覧](#/view/iphone-jailbreak-versions)の通り、A18系(iPhone 16系)以降は脱獄ツール自体が存在しないため、これらの端末でHuskを使う場合は内蔵StikJITかStikDebugでのJIT取得が事実上唯一の選択肢になる。

---

## ソースからビルドする場合(開発者向け)

Mac + Xcode環境が必要。ビルドスクリプトはmeson, ninja, pkg-config, python3, xcodegen, qemu-img、Rust(aarch64-apple-iosターゲット)を呼び出す。

```bash
./scripts/ci_build.sh
```

QEMU・ANGLE・virglrenderer・MoltenVK・on-deviceペアリングライブラリをクリーンな状態から取得してクロスコンパイルし、`build/Husk.ipa` を生成する。初回ビルドは数時間かかるが、2回目以降は既存のビルド成果物を再利用する。アプリ部分のみの再ビルドは:

```bash
./scripts/package_ipa.sh
```

で `~/Desktop/Husk.ipa` に出力される。

---

## ライセンス・App Store配布不可の理由

GPL-2.0-or-later。QEMU(GPLv2)をリンクしているため配布バイナリはGPLv2の結合著作物となりソース公開義務がある。Translation Layer部分もHusk独自コードとして同ライセンス。`get-task-allow` とランタイムでのデバッガアタッチが必須のため、**App Store配布は不可能**(サイドロード専用)。

---

## コミュニティ
- Discord: https://discord.gg/xApMHWAzkh
- Issues/PR: [GitHub](https://github.com/Leviidev/Husk)

---

出典: [Leviidev/Husk README](https://github.com/Leviidev/Husk)（2026年10月時点の内容）。開発中プロジェクトのため、仕様・対応端末は変わる可能性があります。導入前に公式READMEの最新版を確認してください。
