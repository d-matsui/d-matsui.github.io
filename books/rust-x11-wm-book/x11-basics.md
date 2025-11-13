---
title: "X Window System の基礎知識"
---

## X Window System

このセクションでは、X Window System の概要とクライアント・サーバーモデルについて説明します。

### X Window System の概要

X Window System は、Unix-like な OS でグラフィカルなアプリケーションを動かすためのシステムです。1980 年代に MIT で開発され、現在は [X.org Foundation](https://x.org/wiki/) で開発されています。

X Window System は、X や X11 と呼ばれることもあります。X11 の 11 は、プロトコルのバージョン番号です。本書では、X Window System のことを X11 と表記します。

:::message
近年は、X11 に代わる Wayland という技術も登場していますが、本書では X11 を扱います。
:::

### クライアント・サーバーモデル

X11 は、クライアント・サーバーモデルを採用しています。

X サーバーは、クライアントからのリクエストを受け取ってウィンドウを画面に表示したり、キーボードやマウスの入力をクライアントに送ったりするプログラムです。代表的な実装として [Xorg](https://gitlab.freedesktop.org/xorg/xserver) があります。

X クライアントは、ターミナルやブラウザなどのアプリケーションです。X クライアントは、X サーバーにリクエストを送ってウィンドウを表示させたり、サーバーからキーボードやマウスの入力を受け取ったりします。

![X11 クライアント・サーバーモデル](/images/x11-client-server-model.png)

クライアントとサーバーは、[X11 Protocol](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html) に従って通信します。

X11 のアプリケーション開発では、X11 Protocol を使いやすくするライブラリが使われます。C 言語では `XCB` や `Xlib` が使われます。本書では、Rust 向けのライブラリである [`x11rb`](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html) を使用します。

## Screen, Display, Window の概念

このセクションでは、X11 における screen, display, window の概念と、それらの関係性について説明します。

### Screen と Display

screen は、物理モニターに対応する概念です。

display は、キーボード・マウスと 1 つ以上の screen をまとめた概念です。一般的に「ディスプレイ」といえば物理モニターを指しますが、X11 における display は異なる概念なので注意が必要です。

:::message
現代の環境では、複数の物理モニターが 1 つの仮想 screen として扱われることがほとんどです。そのため、ほとんどの環境ではデフォルトスクリーン (screen 0) の 1 つだけが存在します。本書では、この単一 screen 環境を前提とします。
:::

![display, screen, 物理モニターの関係](/images/x11-display-screen.png)

### Window とその階層構造

screen には、1 つの root window が存在します。root window は screen 全体をカバーする、階層構造の頂点にあるウィンドウです。

すべてのウィンドウ (root window 以外) は、root window の子孫 (descendant) です。子ウィンドウ (child window) はさらに子ウィンドウを持つことができ、これにより任意の深さのツリー構造が作られます。

![screen, root window, 子ウィンドウの関係](/images/x11-window-hierarchy.png)

## Window Manager

このセクションでは、Window Manager の役割とその特別な権限について説明します。

Window Manager も、ターミナルやブラウザと同じ X クライアントです。ただし、他のクライアントとは異なる特別な権限を持っています。

Window Manager は、X サーバーが他のクライアントから受け取ったリクエストを、自分にリダイレクトさせる権限を持っています。この権限により、Window Manager は他のクライアントが作成しようとしたウィンドウの配置やサイズを最終的に決定できます。

例えば、ターミナルを起動すると、ターミナルは X サーバーにウィンドウ作成リクエストを送ります。Window Manager が存在しない場合、X サーバーはリクエスト通りに表示します。しかし、Window Manager が存在する場合、X サーバーはリクエストを Window Manager にリダイレクトします。Window Manager がリクエストを処理し、改めて X サーバーにリクエストすることで、タイル型レイアウトなどの配置を実現できます。

このようなリダイレクトは、SubstructureRedirect という仕組みによって実現されています。Window Manager は root window に SubstructureRedirect を設定することで、X サーバーからウィンドウ作成リクエストを受け取ります。

![SubstructureRedirect の動作](/images/x11-substructure-redirect.png)

:::message
SubstructureRedirect を使えるのは、同時に 1 つのクライアントだけです。そのため、複数の Window Manager を同時に動作させることはできません。
:::
