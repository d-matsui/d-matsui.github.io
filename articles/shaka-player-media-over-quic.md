---
title: "Shaka Player で Media over QUIC を動かす"
emoji: "🎬"
type: "tech"
topics: ["quic", "webtransport", "moq", "streaming", "shaka"]
published: false
---

## 概要

- 異なる MoQT 実装間で Media over QUIC の相互接続を検証した
- 検証のために moq-wasm (OSS の MoQ 実装) に WebTransport を実装した
- 検証中に発見した Shaka Player のバグを修正し、本家にマージされた

## はじめに

:::message
この記事は主に MoQ / MoQT の仕様や実装に興味がある、あるいは実際に触っている人向けです。MoQT や fMP4 などの用語の説明は省略しています。
:::

近年、低遅延でスケーラブルなメディア配信を実現するための技術として、Media over QUIC (MoQ) に注目が集まっています。

MoQ がどういう技術で、どういう経緯で生まれたかに興味がある人は、[Cloudflare のブログ](https://blog.cloudflare.com/ja-jp/moq/) や[元 Twitch の Luke のブログ](https://moq.dev/blog/) がおすすめです。

OSS のメディアプレイヤーに [Shaka Player](https://github.com/shaka-project/shaka-player) があります。最近 (?) MoQ の Media Streaming Format (MSF) に対応したようで、この機能を実際に使ってみた話を今回書きます。

## 構成

- Publisher: MoQtail + Mediabunny を使って、gUM した映像・音声を fMP4 に変換して MoQT に載せて配信する
- Relay: moq-wasm に WebTransport を実装して、Relay Server としてローカルに構築する
- Subscriber: Shaka Player を使って、Publisher が配信した映像・音声を Subscribe する

```
Publisher                  Relay                    Subscriber
+---------+  WebTransport  +---------+  WebTransport  +---------------+
| MoQtail |  ============> | moq-wasm|  ============> | Shaka Player  |
+---------+                +---------+                +---------------+
  gUM → fMP4 → MoQT Object                          MoQT Object (fMP4) → MSE
```

実際に動かしたときのスクショです。

![Publisher (左) と Subscriber (右) で同じ映像が再生されている様子](/images/moq-interop-demo.png)

## 相互接続の検証

### MoQtail + Mediabunny で映像・音声を配信する

Publisher には、MoQtail と Mediabunny を使いました。Mediabunny (TypeScript の muxer ライブラリ) で gUM した映像・音声を fMP4 に変換し、MoQtail (MoQ の TypeScript ライブラリ) で MoQT Object として配信しました。

https://github.com/d-matsui/moq-wasm/tree/feat/interop-shaka-player

ここで1つハマったのが、MoQT Object のヘッダーの指定です。MoQtail で Object を生成する際、ヘッダーなしの意味で `[]` を渡していました。

```ts
MoqtObject.newWithPayload(
  trackName,
  new Location(groupId, 0n),
  0,                                    // publisherPriority
  ObjectForwardingPreference.Subgroup,  // forwardingPreference
  0n,                                   // subgroupId
  [],                                   // extensions: 空配列 = なし...のつもり
  payload,
);
```

:::message alert
`[]` は JavaScript では truthy なので、「ヘッダーあり」と判定され、余分な `0x00` がエンコードされてしまいました。正しくは `null` を渡す必要がありました。
:::

MoQT はバイナリプロトコルなので、1バイトずれただけでパース全体が壊れます。エラーメッセージからは原因がわからず、「なんでだめなんだ?」と print debug でバイト列を地道に追いかけてようやく気づきました。

### moq-wasm (draft-14) で WebTransport を使えるようにする

[moq-wasm](https://github.com/nttcom/moq-wasm) は OSS の MoQ プロトコル実装です。MoQT draft-14 への対応を進めていますが、現状の実装では QUIC のみで WebTransport が未対応だったため、ブラウザ (Shaka Player) から接続できるように WebTransport を実装しました。

https://github.com/d-matsui/moq-wasm/tree/feat/webtransport

もともと Transport の抽象化がされていたので、大きな修正にならずに済みました。

### Shaka Player で映像・音声を再生する

Shaka Player は最近 MoQ の Media Streaming Format (MSF) に対応しました。今回はこの機能を使って、Publisher が配信した映像・音声を Subscribe しました。

ただ、実際に動かしてみるといくつかバグがあり、そのままでは動作しませんでした。Shaka Player を fork して、バグと思われる箇所を修正しました。修正した内容は以下の3つです。

- **draft-14 の SubgroupHeader types のサポート** ([#9802](https://github.com/shaka-project/shaka-player/pull/9802)) - stream type 判定が draft-11 までしか対応しておらず、draft-14 の stream type が無視されていました。
- **namespace tuple encoding の修正** ([#9803](https://github.com/shaka-project/shaka-player/pull/9803)) - namespace の tuple encoding が間違っていて、SUBSCRIBE が失敗していました。
- **Closure Compiler の property mangling** ([#9804](https://github.com/shaka-project/shaka-player/pull/9804))

:::details #9804 の詳細 (興味がある人向け)
このうち #9804 が一番原因の特定に苦労しました。Shaka Player の MSF 機能は uncompiled モードでは正常に動作するのに、compiled モードでは動作しませんでした。

Shaka Player は Google Closure Compiler でビルドされており、最適化の一環としてプロパティ名を短い名前に置き換えます (property mangling)。外部から受信した JSON を `JSON.parse()` して得たオブジェクトに対して、mangling されたプロパティ名でアクセスしていた (e.g., `track.isLive` → `track.sa`) のが原因でした。

修正は、該当の型を mangling の対象外として扱う設定にするだけでした。
:::

## 後日談

fork で修正して動作確認していたところ、メンテナが watch していたようで、コメントをもらいました。

> ["Do you want create a PR to add it to Shaka Player?"](https://github.com/d-matsui/shaka-player/commit/aa516af5e7daddcd12422c4e02c0cff0329f7de9#commitcomment-178857192)

せっかくなので PR を出したところ、3つとも素早くレビュー & マージされました。マージ後すぐに次のバージョンとして[リリース](https://github.com/shaka-project/shaka-player/releases/tag/v5.0.5)もされていて、このスピード感は見習いたいと思いました。

## まとめ

moq-wasm に WebTransport を実装し、MoQtail (Publisher), moq-wasm (Relay), Shaka Player (Subscriber) の3者間で Media over QUIC の相互接続ができました。検証の過程で見つけた Shaka Player のバグ修正も本家にマージされ、v5.0.5 としてリリースされています。

実装の詳細は、(興味がある人がいれば) [私のブログ](https://d-matsui.github.io/) にでも書こうかなと思います。

---

この記事が役立ったら、LIKEやコメントで教えてください！

他の技術記事や開発記録は[私のブログ](https://d-matsui.github.io/)でも公開しています。
