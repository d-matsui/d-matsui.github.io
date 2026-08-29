---
author: Daiki Matsui
pubDatetime: 2026-08-29T15:33:24+09:00
title: "mediasoup がメディアを受け取るまで"
slug: mediasoup-signaling
featured: false
draft: false
tags:
  - webrtc
  - mediasoup
  - rtp
description: "mediasoup が ORTC 由来のオブジェクトでシグナリングする仕組みと、ブラウザが WebRTC トランスポートを確立してメディアを送りはじめるまでの流れを解説する。"
---

## はじめに

広く使われている WebRTC SFU 実装の 1 つに [mediasoup](https://mediasoup.org/) がある。
最近、mediasoup の実装を調べる機会があったので、簡単にまとめようと思う。

この記事では、mediasoup がどのようにして WebRTC 接続を確立して、メディアを受け取るかについて書く。

## アーキテクチャ

まずは mediasoup に登場する主要な概念をおさえる。

![mediasoup のアーキテクチャ](/mediasoup-architecture.svg)

### [Worker](https://mediasoup.org/documentation/v3/mediasoup/api/#Worker)

> A worker represents a mediasoup C++ subprocess that runs in a single CPU core and handles Router instances.

Router を処理する C++ のサブプロセスのこと。
普通、CPU コア 1 個につき Worker を一個立てる。

### [Router](https://mediasoup.org/documentation/v3/mediasoup/api/#Router)

> A router enables injection, selection and forwarding of media streams through Transport instances created on it.

メディアストリームを入れたり、どれを送るか選んだり、転送したりするためのもの。
Router 上に作られる Transport インスタンスを通して、挿入・選択・転送をする。

下記のように Worker を作って、Router を作る。

```ts
import * as mediasoup from "mediasoup";

const worker = await mediasoup.createWorker();
const router = await worker.createRouter();
```

### [Transport](https://mediasoup.org/documentation/v3/mediasoup/api/#Transport)

> A transport connects an endpoint with a mediasoup router and enables transmission of media in both directions by means of Producer, Consumer, DataProducer and DataConsumer instances created on it.

Endpoint と Router を繋いで双方向のメディア送信をするためのもの。
Transport 上に作られる Producer / Consumer インスタンスを通して、双方向のメディア送信をする。

Transport には、WebRtcTransport, PlainTransport, PipeTransport, DirectTransport の 4 種類がある。
たとえば WebRtcTransport は、下記のように Router から作る。

```ts
const transport = await router.createWebRtcTransport(webRtcTransportOptions);
```

### [Producer](https://mediasoup.org/documentation/v3/mediasoup/api/#Producer), [Consumer](https://mediasoup.org/documentation/v3/mediasoup/api/#Consumer)

> A producer represents an audio or video source being injected into a mediasoup router. It's created on top of a transport that defines how the media packets are carried.

Router に入ってくる音声・映像ソースを表現したものが Producer。
下記のように Transport から作る。

```ts
const producer = await transport.produce(producerOptions);
```

> A consumer represents an audio or video source being forwarded from a mediasoup router to an endpoint. It's created on top of a transport that defines how the media packets are carried.

Router から Endpoint へ転送される音声・映像ソースを表現したものが Consumer。
Producer と同様、Transport から作る。

```ts
const consumer = await transport.consume(consumerOptions);
```

## シグナリングの概要

ここまでで、mediasoup に登場する概念と API がなんとなくわかった。
次は、Endpoint と Router がどう連携して WebRTC 接続を確立するのかを理解する。

### 普通のシグナリング

ちょっと WebRTC の復習をする。

WebRTC 接続を確立するためには、ざっくり下記の情報をピア間で交換・ネゴシエーションする必要がある。

- ICE Candidate と、ICE の接続性チェックに使う認証情報 (ufrag と password)
- DTLS 接続確立に使う証明書の fingerprint
- 音声・映像コーデック

そして、これらの情報を Session Description Protocol (SDP) で扱うのだった。

```text
v=0
o=- 3546004397921447048 1596742744 IN IP4 0.0.0.0
s=-
t=0 0
a=fingerprint:sha-256 0F:74:31:25:CB:A2:13:EC:...:A4:60:A8:8E
m=video 9 UDP/TLS/RTP/SAVPF 96
c=IN IP4 0.0.0.0
a=setup:active
a=mid:0
a=ice-ufrag:CsxzEWmoKpJyscFj
a=ice-pwd:mktpbhgREmjEwUFSIJyPINPUhgDqJlSd
a=rtpmap:96 VP8/90000
a=candidate:foundation 1 udp 2130706431 192.168.1.1 53165 typ host generation 0
a=sendrecv
```

なお実物はもっと長い。

### mediasoup におけるシグナリング

mediasoup は SDP を使わずに、ORTC 由来のデータモデルを使ってシグナリングする。
具体的には、下記のようなオブジェクトを扱う。

```ts
const iceParameters = {
  usernameFragment: "CsxzEWmoKpJyscFj",
  password: "mktpbhgREmjEwUFSIJyPINPUhgDqJlSd",
  iceLite: true,
};

const iceCandidates = [
  {
    foundation: "udpcandidate",
    priority: 2130706431,
    address: "192.168.1.1",
    protocol: "udp",
    port: 53165,
    type: "host",
  },
];

const dtlsParameters = {
  role: "auto",
  fingerprints: [
    { algorithm: "sha-256", value: "0F:74:31:25:CB:A2:13:EC:...:A4:60:A8:8E" },
  ],
};

const routerRtpCapabilities = {
  codecs: [
    {
      kind: "video",
      mimeType: "video/VP8",
      preferredPayloadType: 100,
      clockRate: 90000,
    },
  ],
  headerExtensions: [],
};
```

SDP で 1 つのテキストに混ざっていた 3 種類の情報が、そのまま別々のオブジェクトになっている。

ここでもう 1 つ、`RtpParameters` という似た名前のオブジェクトが登場する。
`RtpCapabilities` と対になるもので、役割は逆である。

`RtpCapabilities` は受け取れるものの一覧である。
`codecs` と `headerExtensions` を持ち、コーデックには `preferredPayloadType` が付く。
「この Router は VP8 なら受け取れる。ペイロードタイプは 100 を使いたい」という、可能性の集合を表す。

`RtpParameters` は実際に送るものの記述である。
`codecs` と `headerExtensions` に加えて、`encodings` と `rtcp`、`mid` を持つ。

```ts
const rtpParameters = {
  mid: "0",
  codecs: [
    {
      mimeType: "video/VP8",
      payloadType: 96,
      clockRate: 90000,
    },
  ],
  headerExtensions: [{ uri: "urn:ietf:params:rtp-hdrext:sdes:mid", id: 4 }],
  encodings: [{ ssrc: 2180035812 }],
  rtcp: { cname: "XHbOTNRFnLtesHwJ" },
};
```

`preferredPayloadType` ではなく `payloadType` になり、`encodings` に SSRC が入っている。
つまり「これから送る 1 本のストリームは、ペイロードタイプ 96 の VP8 で、SSRC は 2180035812 である」という確定した設定である。

受信側が `RtpCapabilities` で受け取れるものを提示し、送信側がその範囲に収まるように、実際に送るものを `RtpParameters` として記述する。
これが mediasoup のメディアネゴシエーションである。

## シグナリングの詳細

![ブラウザと mediasoup の間でオブジェクトがやり取りされる流れ](/mediasoup-produce-sequence.svg)

図を上から順に追う。

1. サーバーが Router を作る。クライアントはその `rtpCapabilities` を受け取り、`device.load()` に渡す
2. クライアントが Transport の作成を要求する。サーバーは `createWebRtcTransport()` で作り、id と ICE / DTLS のパラメータを返す。クライアントはそれを `createSendTransport()` に渡す
3. ここまでで通信はまだ始まっていない。`sendTransport.produce({ track })` を呼ぶと、2 つのイベントが順に発火する
4. `connect` イベントでは、ブラウザ自身の `dtlsParameters` が渡される。これをサーバーの `transport.connect()` に渡し、`callback()` を呼ぶ
   - コールバックを呼ぶと、ブラウザから ICE の接続性チェックが飛び、繋がったら DTLS ハンドシェイクが走る。これが終わり次第、ブラウザは SRTP を送りはじめる。
5. `produce` イベントでは、`kind` と `rtpParameters` が渡される。これをサーバーの `transport.produce()` に渡し、返ってきた id を `callback({ id })` で返す
   - サーバーはこれを受け取ってはじめて Producer を作る。それより前に届いたパケットは、対応する Producer がないので捨てられる。
6. 両方のコールバックが呼ばれてはじめて `produce()` が返る

## まとめ

mediasoup のアーキテクチャと WebRTC 接続からメディア送信までに何が行なわれているのかについて書いた。

Worker, Router, Transport, Producer/Consumer という概念があり、mediasoup-client/mediasoup はそれらの API を触る。

mediasoup では、SDP ではなく `iceParameters` などのオブジェクトを Endpoint とやりとりすることで、WebRTC 接続を確立する。

次は mediasoup が RTP パケットをどのように処理するかについて書こうと思う。

## 参考

- [mediasoup: API](https://mediasoup.org/documentation/v3/mediasoup/api/)
- [mediasoup: RTP Parameters and Capabilities](https://mediasoup.org/documentation/v3/mediasoup/rtp-parameters-and-capabilities/)
- [mediasoup: Communication Between Client and Server](https://mediasoup.org/documentation/v3/communication-between-client-and-server/)
