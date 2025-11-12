---
title: "環境構築"
---

## はじめに

この章では、次章以降で開発する Window Manager を動作させるための環境を構築します。
具体的には、Xephyr 上で自作 Window Manager とターミナルを動作させることをゴールとします。

Xephyr は、既存の X 環境の中で動作する仮想的な X サーバーです。これを使うことで、開発中の Window Manager を安全にテストできます。

Unix-like な OS では問題なく動作すると思いますが、動作確認済みの環境は下記の通りです。

- Ubuntu 24.04.2 LTS
- X.Org X Server 1.21.1.11
- X Protocol Version 11, Revision 0

環境は以下のコマンドで確認できます。

```bash
cat /etc/os-release
Xorg --version
```

:::message
`Xorg --version` は、X サーバーが既に動作している環境では権限エラーになります。別のコンソール (Ctrl+Alt+F3) で実行してください。
:::


## 必要なツールのインストール

この章で使用するツールをインストールします。

```bash
sudo apt update
sudo apt install xserver-xorg xserver-xephyr xterm
```

- `xserver-xorg`: X Window System の実装
- `xserver-xephyr`: 仮想 X サーバー
- `xterm`: 動作確認用のターミナル

## Rust プロジェクトの構築

rwm (Rust Window Manager) という名前で開発を進めます。

```bash
cargo new rwm
cd rwm
```

X11 のライブラリは x11rb を使用します。
加えて、エラーハンドリング用の anyhow、ロギング用の tracing と tracing-subscriber も追加します。

```bash
cargo add x11rb anyhow tracing tracing-subscriber
```

## Xephyr の動作確認

まず、rwm のコードを書く前に、Xephyr が正しく動作するか確認します。

### $DISPLAY 環境変数

X クライアントは、どの X サーバーに接続するかを `$DISPLAY` 環境変数で判断します。

`$DISPLAY` のフォーマットは以下の通りです。

```
protocol/hostname:number.screen_number
```

各部分の意味
- `protocol`: プロトコルファミリー (省略可)
- `hostname`: ホスト名 (省略可)
- `number`: display番号
- `screen_number`: screen番号 (省略可、通常 0)

具体例
- `:0` - ローカルホストのdisplay 0、screen 0
- `:10` - ローカルホストのdisplay 10、screen 0

通常、デスクトップ環境では `:0` が既に使われています。Xephyr では、既存の環境と分離するために `:10` など異なる番号を使います。

### Xephyr を起動して xterm を表示

ターミナルで Xephyr を起動します。

```bash
Xephyr :10 -screen 1280x720
```

別のターミナルで xterm を起動します。

```bash
DISPLAY=:10 xterm
```

Xephyr のウィンドウ内に xterm が表示されれば成功です。

![Xephyr のウィンドウ内に xterm が表示される](/images/xephyr.png)

## X サーバーに接続する

次に、rwm から X サーバーに接続するコードを書きます。

`src/main.rs` を以下の内容に書き換えてください。

```rust
use anyhow::Result;
use tracing::info;

fn main() -> Result<()> {
    // Initialize tracing subscriber to enable logging
    tracing_subscriber::fmt::init();

    // Connect to X server using $DISPLAY
    let (_conn, screen_num) = x11rb::connect(None)?;
    info!("Connected to X server with screen {:?}", screen_num);

    Ok(())
}
```

`x11rb::connect(None)` は、`$DISPLAY` 環境変数を使って X サーバーに接続します。戻り値の `screen_num` は、接続した screen の番号です (通常は 0)。

## 動作確認

Window Manager を含めて動作確認するためのシェルスクリプトを作成します。

`test.sh` を以下の内容で作成してください。

```bash
#!/bin/bash

# Building rwm
cargo build 2>/dev/null || exit 1

# Use nested X server (Xephyr)
dpy_name=":10"
Xephyr $dpy_name -screen 1280x720 2>/dev/null &
XEPHYR_PID=$!
sleep 1

# Exec window manager
DISPLAY=$dpy_name RUST_LOG=debug ./target/debug/rwm &
RWM_PID=$!
sleep 1

# Launch xterm
DISPLAY=$dpy_name xterm &

trap "kill $XEPHYR_PID $RWM_PID 2>/dev/null" EXIT
wait $XEPHYR_PID
```

実行:

```bash
chmod +x test.sh
./test.sh
```

以下の結果が確認できれば成功です。
- Xephyr のウィンドウが開く
- ターミナルに "Connected to X server with screen 0" のログが出力される
- xterm が Xephyr 内に表示される

## まとめ

この章では、Window Manager 開発のための環境を構築しました。

- Xephyr と必要なツールをインストールした
- rwm プロジェクトを作成し、x11rb などの依存関係を追加した
- `$DISPLAY` 環境変数とそのフォーマットを理解した
- Xephyr 上で rwm と xterm を動作させた

次の章では、最小限の Window Manager を実装していきます。
