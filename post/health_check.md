+++
date = "2019-08-02T21:39:29+09:00"
draft = false
title = "ブロックチェーン関連プロダクトのヘルスチェックの実装を眺める"
+++

[ゼロから創る暗号通貨](https://peaks.cc/books/cryptocurrency)という、PythonでP2Pネットワークの部分から実装を進めていって最終的にBitcoin的なものを創るというめっちゃ面白い本があって、最近これを [Rustで実装](https://github.com/ackintosh/blue) している。

本の序盤、P2Pネットワークの基盤を作っていくところで、接続してるPeerのヘルスチェックを実装する。といってもこの本ではシンプルにTCPのレイヤーで繋がるかどうか確認するだけなんだけど、なんとなくこの部分を実際のプロダクトはどんな感じに実装しているのか（何かスゴイことやってるのか？）気になってコードを眺めてみたのでブログに書いておく。

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

ヘルスチェックのPing送信は下記をgoroutineで実行している。

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

Pingを受ける側のコード。受信したメッセージコードで判断して、`pongMsg(0x03)` のメッセージコードでレスポンスを返している。

[p2p/peer.go#L283-L287](https://github.com/ethereum/go-ethereum/blob/a1f8549262567ddacac3d4180f8a6ca0826036a9/p2p/peer.go#L283-L287)

```go
func (p *Peer) handle(msg Msg) error {
	switch {
	case msg.Code == pingMsg:
		msg.Discard()
		go SendItems(p.rw, pongMsg)
```

#### Pong受信

ふたたびPingを送信する方のコードに戻る。こちらはレスポンス(Pong)のメッセージコードは見ていなくて、単純にPingが成功したかどうかだけの判断。

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

#### go-ethereumはL4でのチェック

上記の通りざっとコードを眺めた感じではgo-ethereumのヘルスチェックはペイロードの中身まではチェックしていない。なので先述のヘルスチェックの種類でいうと `(2)` のL4のチェックに当てはまる。  
（命名に "Ping" を使っているが、[pingコマンド](http://linuxjm.osdn.jp/html/netkit/man8/ping.8.html)のようにICMP(L3)でやりとりしてるわけではない）


## libp2p

[libp2p - https://libp2p.io/](https://libp2p.io/)

P2Pネットワークアプリケーションの実装に必要なプロトコルや仕様をオープンに議論しながら決めていこうというプロジェクト。当初はP2Pの分散ファイルシステムである [IPFS](https://ipfs.io/) の一部として始まったが、現在では特定のプロダクトに依存しない形になっている。

https://github.com/libp2p/specs

仕様は上記リポジトリで管理していて、[いくつかの言語で仕様の実装が進んでいる](https://github.com/libp2p)。JS, Node, Goあたりが一番進んでいて、その他の言語も絶賛実装中という雰囲気。

## rust-libp2p

libp2pのRust実装であるrust-libp2pのコードを眺めてみる。

[libp2p/rust-libp2p: The Rust Implementation of libp2p networking stack.](https://github.com/libp2p/rust-libp2p)

### Ping送信

[rand::thread_rng](https://rust-num.github.io/num/rand/fn.thread_rng.html)で生成した32byteのランダムなデータを送信している。

[protocols/ping/src/protocol.rs#L88-L107](https://github.com/libp2p/rust-libp2p/blob/ce4ca3cc75b68780d980f92471e616b2c2e311e4/protocols/ping/src/protocol.rs#L88-L107)

```rust
impl<TSocket> OutboundUpgrade<TSocket> for Ping
where
    TSocket: AsyncRead + AsyncWrite,
{
    type Output = Duration;
    type Error = io::Error;
    type Future = PingDialer<Negotiated<TSocket>>;

    #[inline]
    fn upgrade_outbound(self, socket: Negotiated<TSocket>, _: Self::Info) -> Self::Future {
        let payload: [u8; 32] = thread_rng().sample(distributions::Standard);
        debug!("Preparing ping payload {:?}", payload);

        PingDialer {
            state: PingDialerState::Write {
                inner: nio::write_all(socket, payload),
            },
        }
    }
}
```

###### 補足: "Upgrade"について

libp2pでは、プロトコルで定めた様々な機能を層状に積み重ねていくような概念を持っていて、そのように積み重ねる様を指して、コネクションを "upgradeする" と呼んでいる。

https://github.com/libp2p/specs/blob/master/connections/README.md#upgrading-connections

当記事で抜粋しているrust-libp2pのコードのトレイトやメソッドの命名に "Upgrade" が入っているのは、そのようなlibp2pの思想が命名に反映されているため。


### Ping受信 -> Pong返信

Pingを受信した側は、[受け取った32byteのデータをそのまま返している](https://github.com/libp2p/rust-libp2p/blob/ce4ca3cc75b68780d980f92471e616b2c2e311e4/protocols/ping/src/protocol.rs#L81)。

[protocols/ping/src/protocol.rs#L62-L86](https://github.com/libp2p/rust-libp2p/blob/ce4ca3cc75b68780d980f92471e616b2c2e311e4/protocols/ping/src/protocol.rs#L62-L86)

```rust
impl<TSocket> InboundUpgrade<TSocket> for Ping
where
    TSocket: AsyncRead + AsyncWrite,
{
    type Output = ();
    type Error = io::Error;
    type Future = future::Map<
        future::AndThen<
        future::AndThen<
        future::AndThen<
            RecvPing<TSocket>,
            SendPong<TSocket>, fn((Negotiated<TSocket>, [u8; 32])) -> SendPong<TSocket>>,
            Flush<TSocket>, fn((Negotiated<TSocket>, [u8; 32])) -> Flush<TSocket>>,
            Shutdown<TSocket>, fn(Negotiated<TSocket>) -> Shutdown<TSocket>>,
    fn(Negotiated<TSocket>) -> ()>;

    #[inline]
    fn upgrade_inbound(self, socket: Negotiated<TSocket>, _: Self::Info) -> Self::Future {
        nio::read_exact(socket, [0; 32])
            .and_then::<fn(_) -> _, _>(|(sock, buf)| nio::write_all(sock, buf))
            .and_then::<fn(_) -> _, _>(|(sock, _)| nio::flush(sock))
            .and_then::<fn(_) -> _, _>(|sock| nio::shutdown(sock))
            .map(|_| ())
    }
}
```

### Pong受信

ふたたびPingを送信する方（PingDialer）のコードに戻る。Pongで返ってきたデータが正しいかどうかチェックしている。

[protocols/ping/src/protocol.rs#L159-L165](https://github.com/libp2p/rust-libp2p/blob/ce4ca3cc75b68780d980f92471e616b2c2e311e4/protocols/ping/src/protocol.rs#L159-L165)

```rust
...
...
                PingDialerState::Read { ref mut inner, payload, started } => {
                    let (socket, payload_received) = try_ready!(inner.poll());
                    let rtt = started.elapsed();
                    if payload_received != payload {
                        return Err(io::Error::new(
                            io::ErrorKind::InvalidData, "Ping payload mismatch"));
                    }
...
...
```

それと、Ping-Pongの往復にかかった時間（Round Trip Time）の計測をしている。


```rust
                PingDialerState::Flush { ref mut inner, payload } => {
                    ...
                    let started = Instant::now();
                    PingDialerState::Read {
                        ...
                        started,
                    }
                },
                PingDialerState::Read { ref mut inner, payload, started } => {
                    ...
                    let rtt = started.elapsed();
                    ...
                    PingDialerState::Shutdown {
                        ...
                        rtt,
                    }
                },
```

### rust-libp2pはL7でのチェック

はじめに見たgo-ethereumとは違ってlibp2p(rust-libp2p)では、返ってくるデータのチェックまで行っていた。これは先述のヘルスチェックの種類でいうと `(3)` のL7のチェックに当てはまる。

また、RTTの計測をしていたのは面白い違いだった。ブロックチェーンの話題から逸れるが、おそらくロードバランサの文脈でいうと、振り分け先を選ぶ基準にこのようにヘルスチェック時に計測したRTTを使うこともあるのだろう。
