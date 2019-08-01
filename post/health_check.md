+++
date = "2019-08-01T21:39:29+09:00"
draft = true
title = "ブロックチェーン関連プロダクトのヘルスチェックの実装をみる"
+++

[ゼロから創る暗号通貨](https://peaks.cc/books/cryptocurrency)という、PythonでP2Pネットワークの部分から実装を進めていって最終的にBitcoin的なものを創るというめっちゃ面白い本があって、最近これを [Rustで実装](https://github.com/ackintosh/blue) している。

本の序盤、P2Pネットワークの基盤を作っていくところで、接続してるPeerのヘルスチェックを実装する。といってもこの本ではシンプルにTCPのレイヤーで繋がるかどうか確認するだけなんだけど、なんとなくこの部分を実際のプロダクトはどんな感じに実装しているのか（何かスゴイことやってるのか？）気になってコードを見てみたのでブログに書いておく。

<!--more-->

ヘルスチェック自体はP2Pネットワークやブロックチェーンに限ったものではないけど、上記の経緯から、ブロックチェーン関連のプロダクトを調べた。

## ヘルスチェックの種類

まずはヘルスチェックについてググったところ下記がわかりやすかった。🙏 感謝

[4.14. ヘルスチェック — TERASOLUNA Server Framework for Java (5.x) Development Guideline 5.5.1.RELEASE documentation](https://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebApplicationDetail/HealthCheck.html#id4)

![HealthCheck](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/health-check/healthcheck.png)

OSI参照モデルのL3, L4, L7で、どのレイヤーのチェックを行うかで種類が分かれる。例えばHTTP(L7)だったらステータスコードやレスポンスボディをチェックする。ふむふむ。

## go-ethereum

[ethereum/go-ethereum: Official Go implementation of the Ethereum protocol](https://github.com/ethereum/go-ethereum)

#### Ping送信

ヘルスチェックのリクエストは下記をgoroutineで実行している。

[p2p/peer.go#L249-L265](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L249-L265)

```go
func (p *Peer) pingLoop() {
	ping := time.NewTimer(pingInterval)
	defer p.wg.Done()
	defer ping.Stop()
	for {
		select {
		case <-ping.C:
			if err := SendItems(p.rw, pingMsg); err != nil {
				p.protoErr <- err
				return
			}
			ping.Reset(pingInterval)
		case <-p.closed:
			return
		}
	}
}
```

メッセージコードとして `0x02` をセットしているだけで、それ以外は何も送っていない。

[p2p/peer.go#L54](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L54)

```go
const (
	// devp2p message codes
	handshakeMsg = 0x00
	discMsg      = 0x01
	pingMsg      = 0x02
	pongMsg      = 0x03
)
```

#### Ping受信 -> Pong返信

受信側は下記をgoroutineで実行している。

[p2p/peer.go#L267-L281](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L267-L281)

```go
func (p *Peer) readLoop(errc chan<- error) {
	defer p.wg.Done()
	for {
		msg, err := p.rw.ReadMsg()
		if err != nil {
			errc <- err
			return
		}
		msg.ReceivedAt = time.Now()
		if err = p.handle(msg); err != nil {
			errc <- err
			return
		}
	}
}
```

メッセージを受ける側のコード。受信したメッセージコードで判断して、`pongMsg(0x03)` のメッセージコードでレスポンスを返している。

[p2p/peer.go#L283-L287](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L283-L287)

```go
func (p *Peer) handle(msg Msg) error {
	switch {
	case msg.Code == pingMsg:
		msg.Discard()
		go SendItems(p.rw, pongMsg)
```

#### Pong受信

ふたたびPingを送信した方のコードに戻る。こちらはメッセージコードは見ていなくて、単純にPingが成功したかどうかだけの判断。

```go
			if err := SendItems(p.rw, pingMsg); err != nil {
				p.protoErr <- err
```

Pingに失敗したら、そのPeerとの接続を閉じる。

[p2p/peer.go#L234-L236](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L234-L236)

```go
		case err = <-p.protoErr:
			reason = discReasonForError(err)
			break loop
	// ...
	close(p.closed)
	p.rw.close(reason)
	// ...
```


## rust-libp2p

[libp2p/rust-libp2p: The Rust Implementation of libp2p networking stack.](https://github.com/libp2p/rust-libp2p)


