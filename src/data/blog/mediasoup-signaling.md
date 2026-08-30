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
description: "mediasoup は、クライアントとサーバーが交換する情報を SDP ではなく ORTC 由来のオブジェクトで表現する。そのデータモデル、ブラウザが WebRTC 接続を確立してメディアの送信を開始するまでの流れを説明する。"
---

## はじめに

広く使われている WebRTC SFU (Selective Forwarding Unit) 実装の 1 つに [mediasoup](https://mediasoup.org/) がある。
最近 mediasoup の実装を調べる機会があったため、その内容をまとめる。

本記事では、mediasoup が WebRTC 接続を確立し、メディアを受け取るまでの流れを説明する。

## アーキテクチャ

まず、mediasoup の主要な概念を確認する。

![mediasoup のアーキテクチャ](/mediasoup-architecture.svg)

### [Worker](https://mediasoup.org/documentation/v3/mediasoup/api/#Worker)

> A worker represents a mediasoup C++ subprocess that runs in a single CPU core and handles Router instances.

Worker は、Router インスタンスを持つ mediasoup の C++ サブプロセスである。
通常、CPU コア 1 個につき Worker を 1 個起動する。

### [Router](https://mediasoup.org/documentation/v3/mediasoup/api/#Router)

> A router enables injection, selection and forwarding of media streams through Transport instances created on it.

Router は、メディアストリームの挿入、選択、転送を行う。
これらの処理は、Router 上に作成した Transport インスタンスを通して行う。

Worker と Router は、下記のように作成する。

```ts
import * as mediasoup from "mediasoup";

const worker = await mediasoup.createWorker();
const router = await worker.createRouter();
```

### [Transport](https://mediasoup.org/documentation/v3/mediasoup/api/#Transport)

> A transport connects an endpoint with a mediasoup router and enables transmission of media in both directions by means of Producer, Consumer, DataProducer and DataConsumer instances created on it.

Transport は、Endpoint (ブラウザなどのクライアント) と Router を接続し、双方向のメディア伝送を可能にする。
メディアの送受信は、Transport 上に作成した Producer / Consumer インスタンスを通して行う。

Transport には、WebRtcTransport, PlainTransport, PipeTransport, DirectTransport の 4 種類がある。
本記事では、ブラウザとの接続に使う WebRtcTransport のみを扱う。
たとえば WebRtcTransport は、下記のように Router から作成する。

```ts
const transport = await router.createWebRtcTransport(webRtcTransportOptions);
```

### [Producer](https://mediasoup.org/documentation/v3/mediasoup/api/#Producer), [Consumer](https://mediasoup.org/documentation/v3/mediasoup/api/#Consumer)

> A producer represents an audio or video source being injected into a mediasoup router. It's created on top of a transport that defines how the media packets are carried.

Producer は、Router に入力される音声・映像ソースを表現する。
Producer は、下記のように Transport から作成する。

```ts
const producer = await transport.produce(producerOptions);
```

> A consumer represents an audio or video source being forwarded from a mediasoup router to an endpoint. It's created on top of a transport that defines how the media packets are carried.

Consumer は、Router から Endpoint へ転送される音声・映像ソースを表現する。
Consumer も Producer と同様、Transport から作成する。

```ts
const consumer = await transport.consume(consumerOptions);
```

## シグナリングのデータモデル

ここまでで、mediasoup の主要な概念と API を確認した。
次に、クライアントとサーバーが交換する情報を説明する。

### 一般的な WebRTC の場合

まず、WebRTC のシグナリングを復習する。

WebRTC 接続を確立するためには、下記の情報をピア間で交換する必要がある。

- ICE Candidate と、ICE の接続性チェックに使う認証情報 (ufrag と password)
- DTLS 接続確立に使う証明書の fingerprint
- 音声・映像コーデック

音声・映像コーデックは、双方が対応するものをネゴシエーションで決定する必要がある。

上記 3 種類の情報は、Session Description Protocol (SDP) を用いて記述する。

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

なお、実際の SDP はさらに長い。

### mediasoup の場合

mediasoup は SDP を使わず、ORTC (Object Real-Time Communication) 由来のデータモデルでシグナリングに必要な情報を表現する。
mediasoup はシグナリングチャネル自体を提供せず、情報を交換する手段はアプリケーションに委ねられている。

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

`routerRtpCapabilities` は `codecs` と `headerExtensions` を持ち、コーデックには `preferredPayloadType` が付く。
「この Router は VP8 なら受け取れる。ペイロードタイプは 100 を使いたい」という、対応可能なコーデックを表す。

`RtpParameters` は、実際に送信するストリームの設定であり、`codecs` と `headerExtensions` に加えて、`encodings` と `rtcp`、`mid` などを持つ。「これから送る 1 本のストリームは、ペイロードタイプ 96 の VP8 で、SSRC は 2180035812 である」ということを表す。

受信側は、`RtpCapabilities` で受信可能な範囲を提示し、送信側は、その範囲に収まるように送信内容を `RtpParameters` として記述する。
この `RtpCapabilities` と `RtpParameters` のマッチングが、SDP の offer/answer によるネゴシエーションに相当する。

## シグナリングの流れ

![ブラウザと mediasoup の間でオブジェクトがやり取りされる流れ](/mediasoup-produce-sequence.svg)

クライアント側では、mediasoup-client の `Device` を通して Transport と Producer を作成する。

図を上から順に追う。

1. サーバーが Router を作成する。クライアントはその `rtpCapabilities` を受け取り、`device.load()` に渡す。
2. クライアントが Transport の作成を要求する。サーバーは `createWebRtcTransport()` で作成し、`id` と ICE / DTLS のパラメータを返す。クライアントはそれを `createSendTransport()` に渡す。
3. ここまでの手順では、メディアの送信はまだ始まっていない。`sendTransport.produce({ track })` を呼ぶと、2 つのイベントが順に発火する。
4. `connect` イベントでは、ブラウザ自身の `dtlsParameters` が渡される。これをサーバーの `transport.connect()` に渡し、`callback()` を呼ぶ。
   - コールバックを呼ぶと、ブラウザが ICE の接続性チェックを開始する。接続性チェックが成功すると、DTLS ハンドシェイクを実行する。DTLS ハンドシェイクの完了後、ブラウザは SRTP パケット (暗号化された RTP パケット) の送信を開始する。
5. `produce` イベントでは、`kind` と `rtpParameters` が渡される。これをサーバーの `transport.produce()` に渡し、返ってきた `id` を `callback({ id })` で返す。
   - サーバーは、`transport.produce()` の呼び出しを受けてはじめて Producer を作成する。Producer の作成前に到着した RTP パケットは、対応する Producer が存在しないため破棄する。
6. 両方のコールバックが呼ばれてはじめて `produce()` が返る。

## まとめ

本記事では、mediasoup のアーキテクチャと、WebRTC 接続の確立からメディアの送信開始までの流れを説明した。

mediasoup には Worker, Router, Transport, Producer / Consumer という概念があり、アプリケーションはこれらの API を通してメディアを中継する。

mediasoup では、SDP ではなく `iceParameters` などのオブジェクトを Endpoint とやり取りすることで、WebRTC 接続を確立する。

次回は、mediasoup が RTP パケットを処理する仕組みについて書く。

## 参考

- [mediasoup: API](https://mediasoup.org/documentation/v3/mediasoup/api/)
- [mediasoup: RTP Parameters and Capabilities](https://mediasoup.org/documentation/v3/mediasoup/rtp-parameters-and-capabilities/)
- [mediasoup: Communication Between Client and Server](https://mediasoup.org/documentation/v3/communication-between-client-and-server/)
