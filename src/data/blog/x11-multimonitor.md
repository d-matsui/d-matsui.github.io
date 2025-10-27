---
author: Daiki Matsui
pubDatetime: 2025-10-23T10:00:00Z
title: "X11におけるマルチモニターの仕組み"
slug: x11-multimonitor
featured: true
draft: false
tags:
  - X11
  - Linux
  - WindowManager
description: "X11のディスプレイ関連用語に疑問を持ったのを機に、マルチモニター対応のためにX11プロトコルがどう変化してきたか調べた"
---

<!-- このセクションで伝えたいこと: WM開発中に遭遇した画面関連用語(dpy_name, screen_num, $DISPLAY)の混乱を具体的に示し、記事のきっかけを説明する -->

## きっかけ

最近、RustileというX11 Window ManagerをClaude Codeと一緒に開発しました。どうせ作ったなら中身をちゃんと理解したいので、Rustileをフルスクラッチで書き直す試みを始めました。

開発中、X11におけるディスプレイ関連の概念がいくつか出てきました。たとえば、x11rbでX11サーバに接続するコードは、

```rust
let (conn, screen_num) = x11rb::connect(None).unwrap();
```

であり、`connect` の引数には `dpy_name` (ディスプレイネーム) を指定します。

`connect` 関数の定義には、`None` を指定した場合、環境変数 `$DISPLAY` の値が使われると書かれていました。

```rust
    /// Establish a new connection.
    ///
    /// If no `dpy_name` is provided, the value from `$DISPLAY` is used.
    pub fn connect(dpy_name: Option<&str>) -> Result<(Self, usize), ConnectError>
```

そして、冒頭のコードは `connect` の返り値として、X11のconnection と `screen_num` (スクリーンナンバー) を受け取ります。

ここで、`dpy_name`, `screen_num`, `$DISPLAY` と似たような名前が出てきて、それぞれどう違うのかがよくわかりませんでした。

X11におけるディスプレイの扱いについて、思う存分AIに質問したところ、X11のマルチモニターの仕組みがどう変化してきたのかを深掘りできました。この記事では、それらについてまとめます。

---

## X11における screen, display

screen, display などの用語を理解するには、[Xlib - C Language X Interface](https://www.x.org/releases/X11R7.6/doc/libX11/specs/libX11/libX11.html) が参考になりました。特に `Overview of the X Window System`, `Opening the Display`, `Glossary` の章が有用でした。以下では、この資料をベースに用語をまとめます。

### screen

> A screen is a physical monitor and hardware that can be color, grayscale, or monochrome.

screen は、物理的なモニターのことです。たとえば私の環境だと、ThinkPadのモニターとLGの外部モニターを使って、ブログを書いています。それぞれがscreenに対応します。

### display

> A set of screens for a single user with one keyboard and one pointer (usually a mouse) is called a display.

キーボードとマウス + 物理モニターをまとめて display と定義しているようです。普通、ディスプレイといったらモニターを想像すると思うので、注意が必要です。

### display_name

> Specifies the hardware display name, which determines the display and communications domain to be used. On a POSIX-conformant system, if the display_name is NULL, it defaults to the value of the DISPLAY environment variable.
> ...
> On POSIX-conformant systems, the display name or DISPLAY environment variable can be a string in the format:
> protocol/hostname:number.screen_number

display_name は、`protocol/hostname:number.screen_number` という形式のようです。`NULL` の場合に `DISPLAY` 環境変数が使われるというのは、x11rbの実装と一緒でした。

形式についてまとめると、下記の通りです。

- `protocol` : プロトコルファミリー (省略可)
- `hostname` : ホスト名 (省略可)
- `number` : ディスプレイサーバー番号
- `screen_number` : スクリーン番号

例えば私の環境だと、

```
$ echo $DISPLAY
:2
```

となっています。ローカルマシンで起動しているXディスプレイサーバー2番 (のデフォルトスクリーン 0) を意味します。

Xlibの仕様には記載が見つかりませんでしたが、どうやら現代ではほとんどの場合 `screen_num = 0` のようです。そのため、`$DISPLAY` では省略されていたのでしょう。

ここで、なぜ `screen_num = 0` しか使われないのに、screen という概念が存在するの? と疑問に思いました。
調べた結果わかったのは、マルチモニターを扱うために色々あったということです。以下では、その話について書きます。

---

## 1 Screen = 1 モニターの時代

X11におけるマルチモニターの話は、[freedesktop.orgの記事](https://nouveau.freedesktop.org/MultiMonitorDesktop.html) が非常に参考になりました。

昔のX11では、モニターが複数存在している場合、それらをそれぞれのscreenとして管理していたそうです。たとえば、`:0.0` (ラップトップのモニター) と `:0.1` (外部モニター) という具合です。

この時代の大きな問題は、screen間でウィンドウを移動できなかったことです。

Xlibの仕様によると、screenは独自のroot windowを持ちます。

> All the windows in an X server are arranged in strict hierarchies. At the top of each hierarchy is a root window, which covers each of the display screens.

ウィンドウは `CreateWindow` で生成する際に、親ウィンドウを指定します。このとき、ウィンドウは特定のscreenのroot windowの配下に作られるため、生成された瞬間にそのscreenに所属することになります。

ウィンドウの親を変更する `ReparentWindow` リクエストには、[X11 Protocolの仕様](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html)で以下のような制限があります。

> A Match error is generated if: The new parent is not on the same screen as the old parent.

つまり、screen間でウィンドウの親を変更することはプロトコルレベルで禁止されていました。ラップトップのモニター (`:0.0`) で開いたウィンドウを、外部モニター (`:0.1`) に移動させることができないということです。現代のマルチモニター環境で考えてみると、けっこう不便です。

---

## Xinerama: 複数モニター → 1つの巨大Screen

この不便な状況を解決するために、[XineramaがDECから登場しました](https://en.wikipedia.org/wiki/Xinerama)。1990年代中盤〜後半のことです。

Xineramaがやったことはシンプルで、複数の物理モニターを1つの仮想スクリーンとして見せかける、というものです。たとえば、1920x1080のモニター2台を横に並べていたとして、Xサーバーは「3840x1080の1つのスクリーンがあります」とクライアントに伝えます。

これにより、クライアントからはroot windowが1つしかないように見え、単一のウィンドウマネージャーがscreen間をまたいでウィンドウを管理できるようになります。

---

## RandR時代

Xineramaは画期的でしたが問題がありました。設定が静的で、物理モニターを抜き差しするたびにXサーバーの設定を書きかえて再起動しないといけませんでした。

この問題を解決したのがRandR 1.2です。

RandR は元々、画面のリサイズと回転のために開発されました。[freedesktop.orgの記事](https://nouveau.freedesktop.org/MultiMonitorDesktop.html)によると、Version 1.2 でマルチモニター対応が追加され、Xineramaと同じように、複数のモニターを1つのスクリーンとして扱えるようになりました。

> Randr exposes the dual-head card as a single SCREEN, yet having a standard way of managing the multiple monitors.

さらに、[X11R7.3のリリースノート](https://www.x.org/archive/X11R7.3/doc/RELNOTES.txt)によると、

> Also new in the X Server since X11R7.2 is the 1.2 version of the RandR
> extension, which allows for runtime configuration of outputs within X Screens
> and an improved static configuration system for multihead in a RandR 1.2
> environment.

とあり、1.2でruntime configurationに対応したことがわかります。

RandRはその後もVersion 1.5 で Monitor という、より高レベルな抽象化が追加されるなど、進化を続けているようです。

Window Managerの開発者目線だと、RandRのイベントを上手く使って、それぞれのモニターにWindowを描画したりするんでしょう。これについてはまた今度調べようと思います。

---

## まとめ

「スクリーンって何のためにあるんだ」という小さな疑問から始まり、現代ではX11でマルチモニターをどう扱っているのかを掘り下げました。

用語の整理

- display = キーボードとマウス + screen
- screen = 元々は物理モニターに対応していたが、Xinerama/RandR時代では複数モニターを含む仮想screenに変化

マルチモニター対応の変遷

1. screen only: プロトコルの制約により、screen間のウィンドウ移動が不可能
2. Xinerama: 複数モニターを1つのscreenとして扱えるように仮想化 (ただし、設定は静的)
3. RandR 1.2: 仮想スクリーン + 動的な設定

普段何気なく抜き差しして問題なく使えているモニターの裏側で、こういう話があったと思うと興味深いです。

---

## 参考

- [Xlib - C Language X Interface](https://www.x.org/releases/X11R7.6/doc/libX11/specs/libX11/libX11.html)
- [X Window System Protocol](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html)
- [RandR Protocol Specification](https://www.x.org/releases/X11R7.5/doc/randrproto/randrproto.txt)
- [X11R7.3 Release Notes](https://www.x.org/archive/X11R7.3/doc/RELNOTES.txt)
- [Multi-Monitor Desktop (freedesktop.org)](https://nouveau.freedesktop.org/MultiMonitorDesktop.html)
- [Xinerama - Wikipedia](https://en.wikipedia.org/wiki/Xinerama)
