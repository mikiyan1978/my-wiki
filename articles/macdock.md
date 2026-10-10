# MacDock — 自作Tweak

**mikiyan1978**(自分)が開発した、脱獄iOS向けの**macOS風フローティングウィンドウマネージャー**Tweak。任意のアプリをウィンドウとして浮かせ、複数同時表示できる。

[Sileoリポジトリで配布中](https://mikiyan1978.github.io/sileo-repo/)（`com.mikiyan1978.macdock`）

---

## 機能

- 任意のアプリをリサイズ可能なウィンドウとしてフロート表示
- 複数ウィンドウを同時展開
- 画面端へのスナップ(吸着)
- ウィンドウの位置・サイズを記憶
- 信号機ボタン(閉じる／最小化／最大化／復元)
- ピンチでリサイズ
- ドラッグで移動
- ライブプレビュー付き設定画面

---

## 使い方

### ウィンドウを開く
アプリ内の ⧉ ボタンをタップするか、アプリセレクターパネルからアプリを選択する。

### 信号機ボタン(ヘッダーバー左上)
| 色 | 動作 |
|---|---|
| 🔴 赤 | ウィンドウを閉じる(アプリも終了) |
| 🟡 黄 | 0.6倍に縮小 / 復元 |
| 🔵 青 | ウィンドウ解除してフルスクリーン化 |

### ヘッダーバーのジェスチャー
| 操作 | 動作 |
|---|---|
| シングルタップ | ボタンの表示/非表示切り替え |
| ダブルタップ | 0.6倍に縮小 / 復元 |
| 長押し(0.7秒) | ウィンドウを閉じる |
| ドラッグ | ウィンドウを移動 |

### リサイズ
2本指ピンチ、または右下のハンドルをドラッグ。最小サイズは120pt。

### 画面端スナップ
端から70pt以内でリリースすると自動吸着(上下左右対応、Dynamic Island考慮済み)。

### ウィンドウ記憶
アプリごとに位置・サイズを記憶し、次回起動時に復元。

### 設定
設定アプリ → MacDock から、ON/OFF・エッジスナップ・位置記憶・ドラッグ時の透明度をリスプリング不要でライブ反映。

---

## 動作要件

- iOS 15.0 以降
- Rootless脱獄(Dopamine推奨)
- MobileSubstrate / Substitute

```
Package: com.mikiyan1978.macdock
Depends: mobilesubstrate, firmware (>= 15.0)
Version: 1.0.0
Section: Tweaks
Tag: role::developer, compatible_ios16
```

---

## 実装構成(Theos/Logos)

| ファイル | 役割 |
|---|---|
| `MacDockCore.xm` | コア機能・フック |
| `MacDockHooks.xm` | SpringBoard/アプリへのフック |
| `MacDockWindow.xm` | ウィンドウ管理ロジック |
| `MacDockFloatingWindowView.xm` | フローティングウィンドウのView実装 |
| `MacDockAppButton.xm` | アプリ内の⧉ボタン(別dylib: `MacDockAppButton.dylib`) |
| `MacDockView.h/.m` | ビューのベースクラス |
| `MacDockPref/` | 設定Bundle(設定アプリ連携) |
| `headers/MacDockPrivateAPI.h` | 使用するプライベートAPIヘッダ |

ビルド成果物は `MacDock.dylib` と `MacDockAppButton.dylib` の2本立て。

---

## Changelog

### v1.0.0
- 初回リリース
- フローティングウィンドウ管理
- アプリセレクターパネル
- スペーステーマの設定UI
- Sileoリポジトリ対応
- 起動時のホーム画面シーン検出の不具合修正
- デバッグログ・ダンプの削除

---

## 開発メモ

2026年10月、X上で [Huy Nguyen (@Little_34306)](https://x.com/Little_34306) が投稿した、macOS風の信号機ボタン付きフローティングウィンドウのデモ([投稿](https://x.com/Little_34306/status/2108380205320745444))を見て「自分のMacDockにそっくり」と反応。UIコンセプト(traffic light風ウィンドウ、設定リストのフロート表示)が近いことを確認した。
