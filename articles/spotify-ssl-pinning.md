# Spotify: SSL Pinning調査メモ

(2026-09-26。フルの調査内容と動くバイパススクリプトは`~/Frida/spotify_ssl_bypass/README.md`。)

## バイナリ構成

Spotify本体は`Security.framework`/`CFNetwork.framework`のみをリンクしており、独自にバンドルされたTLSスタック(BoringSSL等)は無い。

## 見つかったピン留め実装は2箇所のみ

- `ARTSRSecurityPolicy` / `ARTSRPinningSecurityPolicy`(Ably SDK、SocketRocketフォークのWebSocketトランスポート)
- `GCKX509TrustVerifier`(Google Cast、優先度低、未フック)

メインのHTTPS API通信(`SPTDataLoader`経由)は標準のCFNetwork/NSURLSession検証パスに乗っており、そちらには独自ピン留めは見つからなかった。

## バイパススクリプト(`ssl_bypass.js`)のフック対象

- C関数`SecTrustEvaluate`/`SecTrustEvaluateWithError`を強制成功させる
- `-[ARTSRSecurityPolicy evaluateServerTrust:forDomain:]`と`-[ARTSRPinningSecurityPolicy evaluateServerTrust:forDomain:]`を強制`YES`にする

## 未検証のまま残っている点

前回のセッション時点で`verify_logging.js`(ログイン画面操作・検索操作90秒間)では`SecTrustEvaluate[WithError]`の呼び出しが**一度もカウントされなかった**。考えられる理由は2つ:

- その90秒間に新規TLSハンドシェイクが発生しなかっただけ
- Spotifyの通信が`Network.framework`/QUIC経由で、証明書検証がプロセス内ではなくシステムデーモン(`nesessionmanager`)側で行われている

本当にバイパスが効いているかの確認には、mitmproxyによるMITM検証(端末のWi-FiプロキシをMac上のmitmproxyに向け、そのCA証明書をインストール・信頼した上で平文APIリクエストが見えるか確認)が必要——**これはまだ実施していない**。「バイパスが効いている」と主張する前に、この検証を済ませること。
