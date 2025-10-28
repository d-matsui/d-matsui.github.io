---
author: Daiki Matsui
pubDatetime: 2025-10-28T23:07:26+09:00
title: x11rb に見る Fluent Interface
slug: x11rb-fluent-interface
featured: true
draft: false
tags:
  - Rust
  - X11
  - Design Pattern
description: Rust の X11 ライブラリ x11rb における Fluent Interface パターン
---

## はじめに

Rust における X11 ライブラリの 1 つに `x11rb` があります。`Xlib` 相当の機能を Rust で提供するライブラリです。

`x11rb` のソースコードを読んでいたら、`Fluent Interface` っぽいパターンを見つけました。

せっかくなので、自分の学びのために、記事にまとめておこうと思います。

## Fluent Interface

Fluent Interface はインタフェースのスタイルの一種で、「流れるように読める API の書き方」と理解しています。

Martin Fowler と Eric Evans が名付け親のようです。

[FluentInterface - Martin Fowler](https://martinfowler.com/bliki/FluentInterface.html)

特徴としては、

1. Method Chaining ができて
2. DSL (Domain Specific Language) で書けて
3. Nested Function が使える

ものだと私は理解しています。

Martin Fowler の記事から具体例を借りると、

```
mock.expects(once()).method(“m”).with( or(stringContains(“hello”), stringContains(“howdy”)) );
```

となっていて、

- `expects().method().with()...` のように Method Chaining していて
- `mock expects once method with hello or howdy` のように書けて
- `expects(once())` のようにネストできる

DSL については詳しくありませんが、この例では `mock expects method m once` のように、ドメイン固有の用語を使って自然に読める文章として書けることを指しているのでしょう。

## x11rb の ChangeWindowAttributesAux

X11 Protocol には、Window 属性を変更するリクエスト `ChangeWindowAttributes` が存在します。Window のボーダーを設定したり、Window Manager として動作させるために Root Window に `SubstructureRedirectMask` のイベントマスクを設定したりするときに使います。

その `ChangeWindowAttributes` は Fluent Interface っぽくて、下記のように使えます。

```rust
let change = ChangeWindowAttributesAux::default()
    .background_pixel(0xFFFFFF)
    .event_mask(EventMask::SUBSTRUCTURE_REDIRECT | EventMask::SUBSTRUCTURE_NOTIFY);
```

下記のようにネストできたら、Martin Fowler が言うところの Fluent Interface なのでしょうかね。

```rust
let change = ChangeWindowAttributesAux::default()
    .background_pixel(rgb(255, 255, 255))
    .border_pixel(rgb(0, 0, 0));
```

## まとめ

x11rb で Method Chaining を使った API 設計を見つけて、Fluent Interface について学び直す良い機会になりました。完全な Fluent Interface ではないかもしれませんが、Method Chaining による読みやすさの工夫は感じられました。

## 参考文献

- [FluentInterface - Martin Fowler](https://martinfowler.com/bliki/FluentInterface.html)
- [x11rb - docs.rs](https://docs.rs/x11rb/latest/x11rb/)
