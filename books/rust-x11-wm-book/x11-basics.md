---
title: "X Window Systemの基礎知識"
---

## X Window System

このセクションでは、X Window Systemの概要とクライアント・サーバーモデルについて説明します。

### X Window Systemの概要

X Window Systemは、Unix-likeなOSでグラフィカルなアプリケーションを動かすためのシステムです。1980年代にMITで開発され、現在は [X.org Foundation](https://x.org/wiki/) で開発されています。

X Window Systemは、XやX11と呼ばれることもあります。X11の11は、プロトコルのバージョン番号です。本書では、X Window SystemのことをX11と表記します。

:::message
近年は、X11に代わるWaylandという技術も登場していますが、本書ではX11を扱います。
:::

### クライアント・サーバーモデル

X11は、クライアント・サーバーモデルを採用しています。

Xサーバーは、クライアントからのリクエストを受け取って画面に表示したり、キーボードやマウスの入力をクライアントに送ったりするプログラムです。代表的な実装として [Xorg](https://gitlab.freedesktop.org/xorg/xserver) があります。

Xクライアントは、ターミナルやブラウザなどのアプリケーションです。Xサーバーにリクエストを送ってウィンドウを表示させたり、サーバーからキーボードやマウスの入力を受け取ったりします。

![X11クライアント・サーバーモデル](/images/x11-client-server-model.png)

クライアントとサーバーは、[X11 Protocol](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html)に従って通信します。

X11のアプリケーション開発では、X11 Protocolを使いやすくするライブラリを使います。C言語では XCB や Xlib が使われます。本書では、Rust向けのライブラリである [x11rb](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html) を使用します。

## Screen, Display, Windowの概念

このセクションでは、X11における screen, display, windowの概念と、それらの関係性について説明します。

### ScreenとDisplay

screenは、物理モニターに対応する概念です。

displayは、キーボード・マウスと1つ以上のscreenをまとめた概念です。一般的に「ディスプレイ」といえば物理モニターを指しますが、X11における display は異なる概念なので注意が必要です。

:::message
現代の環境では、複数の物理モニターが1つの仮想screenとして扱われることがほとんどです。そのため、ほとんどの環境ではデフォルトスクリーン (screen 0) の1つだけが存在します。本書では、この単一screen環境を前提とします。
:::

![display, screen, 物理モニターの関係](/images/x11-display-screen.png)

### Windowとその階層構造

screenには、1つのroot windowが存在します。root windowはscreen全体をカバーする、階層構造の頂点にあるウィンドウです。

すべてのウィンドウ (root window以外) は、root windowの子孫 (descendant) です。子ウィンドウ (child window) はさらに子ウィンドウを持つことができ、これにより任意の深さのツリー構造が作られます。

![screen, root window, 子ウィンドウの関係](/images/x11-window-hierarchy.png)

## Window Manager

このセクションでは、Window Managerの役割とその特別な権限について説明します。

Window Managerも、ターミナルやブラウザと同じXクライアントです。ただし、他のクライアントとは異なる特別な権限を持っています。

Window Managerは、Xサーバーが他のクライアントから受け取ったリクエストを、自分にリダイレクトさせる権限を持っています。これにより、他のクライアントが作成しようとしたウィンドウの配置やサイズを最終的に決定できます。

例えば、ターミナルを起動すると、通常はXサーバーがそのウィンドウをそのまま表示します。しかし、Window Managerが存在する場合、Xサーバーはそのリクエストを一旦Window Managerにリダイレクトします。Window Managerがこれを処理し、改めてXサーバーにリクエストすることで、タイル型レイアウトなどの配置を実現できます。

Window Managerは、SubstructureRedirect を使うことで、このリダイレクトを実現します。

![SubstructureRedirectの動作](/images/x11-substructure-redirect.png)

:::message
SubstructureRedirectを使えるのは、同時に1つのクライアントだけです。そのため、複数のWindow Managerを同時に起動することはできません。
:::
