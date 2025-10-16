---
author: Daiki Matsui
pubDatetime: 2025-10-16T16:32:43+09:00
title: "X11で左右のAltキーを区別する"
slug: x11-distinguish-left-right-alt-keys
featured: true
draft: false
tags:
  - X11
  - Linux
  - keyboard
  - Rust
description: "X11でAlt_LとAlt_Rを個別に扱う方法を紹介。キーボード処理の基礎について触れつつ、XQueryKeymapを使った実装アプローチを解説。"
---

## はじめに

自分好みのショートカットキーを設定する際、キーボードの左右のAltキーを区別して使いたいときがあります。例えば、「左Alt (Alt_L) + m」でChromeを起動して、「右Alt (Alt_R) + m」ではTerminalを起動するといった具合です。

X11で動くアプリケーション (ウィンドウマネージャー) を開発する際に、この機能を実装しようとしたのですが、一手間工夫が必要でした。

本記事では、X11で左右のAltキーを区別する方法として、`XQueryKeymap` を使ったアプローチを紹介します。また、X11におけるキーボード処理の基礎となる `ModMask`, `KeyCode`, `KeySym` という3つの概念と、キーイベントだけでは区別が難しい理由についても触れます。

> 本記事では、X11 Protocol, X Window System関連の用語を総称して「X11」と表記します。

## 結論: XQueryKeymapを使う方法

X11で左右のAltキーを区別する一つの方法として、X11 Protocolの`QueryKeymap`リクエスト (Rustのx11rbでは`query_keymap()`関数) を使うアプローチがあります。

これは、現在押されている全てのキーについて、Press/Releaseの状態を32バイトのビットベクトルで取得できるものです。各ビットが1つのKeyCodeに対応しており、Alt_L (通常KeyCode 64) と Alt_R (通常KeyCode 108) のビットを個別にチェックすることで区別が可能になります。

```rust
// Rustとx11rbを使った実装例
let reply = conn.query_keymap()?.reply()?;
let keys = reply.keys;

// Alt_LのKeyCode (通常64) とAlt_RのKeyCode (通常108) をチェック
let alt_l_pressed = is_key_pressed(&keys, 64);
let alt_r_pressed = is_key_pressed(&keys, 108);
```

X11のキーイベント (KeyPress) では、「Altが押されている」という情報しか得られません。しかし、`query_keymap()`を使えば、全キーのPress/Release状態を取得できるので、これを使えば左右のAltを区別できます。

具体的な実装方法とその仕組みは、以下で説明します。

## なぜキーイベントだけでは区別できないのか

X11のキーボード処理を理解するには、`ModMask`, `KeyCode`, `KeySym` という3つの概念を知る必要があります。

### ModMask

キーボードには、修飾キー (Modifier) が存在します。他のキーと組み合わせて使用されるキーのことで、代表的なものは、Shift、Control、Altです。

X11では、このようなModifierをModMaskと呼ばれるビットマスクで表現します ([X11 Protocol: Common Types](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#Common_Types))。各Modifierに1ビットが割り当てられており、そのキーが押されているかどうかを示します。

```
ShiftMask   = 0x0001  (2進数: 00000001)
LockMask    = 0x0002  (2進数: 00000010)
ControlMask = 0x0004  (2進数: 00000100)
Mod1Mask    = 0x0008  (2進数: 00001000)  ← 通常はAlt
Mod2Mask    = 0x0010  (2進数: 00010000)
Mod3Mask    = 0x0020  (2進数: 00100000)
Mod4Mask    = 0x0040  (2進数: 01000000)  ← 通常はSuper/Win
Mod5Mask    = 0x0080  (2進数: 10000000)
```

(ビットマスクにすることで、複数のキーが同時に押されている状況をビット演算で表現できて、なにかと便利なのだと思います。)

ここで重要なポイントは、Alt_LもAlt_Rもどちらも同じMod1Maskとして扱われるということです。

X11のキーイベント (KeyPressやKeyRelease) には、このModMaskの情報が含まれています。つまり、「mキーが押された + Altも押されている」という情報は得られますが、「左Altなのか右Altなのか」は区別できません。

例えば、`xev`コマンドで、左Altを押しながらmキーを押した場合のキーイベントを見てみると下記のようになります。

```
KeyPress event, serial 34, synthetic NO, window 0x2200001,
    root 0x5ab, subw 0x0, time 12345680, (100,150), root:(150,200),
    state 0x8, keycode 58 (keysym 0x6d, m), same_screen YES,
    XLookupString gives 1 bytes: (6d) "m"
```

重要なポイントは以下の通りです。

- `keycode 58` - これはmキーのKeyCode
- `state 0x8` - これはMod1Mask (Alt) が押されていることを示す
- `keysym 0x6d, m` - mキーのKeySymが表示される

しかし、どちらのAltキーが押されているかの情報はありません。`state`フィールドにはModMaskのみが含まれ、Alt_LとAlt_Rは区別されません。

### KeyCode (物理キー)

先程、KeyCodeという概念が登場しました。KeyCodeは、物理的なキーを表す番号です。

[X11 Protocol: Keyboards](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#Keyboards)によると、KeyCodeの特徴は以下の通りです。

> A KEYCODE represents a physical (or logical) key. Keycodes lie in the inclusive range [8,255]. A keycode value carries no intrinsic information, although server implementors may attempt to encode geometry information (for example, matrix) to be interpreted in a server-dependent fashion. The mapping between keys and keycodes cannot be changed using the protocol.

- 8から255の範囲の整数値
- 物理的なキーに対応
- KeyCode自体には意味的な情報は含まれない
- X11サーバーの実装に依存する

重要なのは、このレベルでは`Alt_L`と`Alt_R`は区別されるということです。例えば、標準的なLinux環境では以下のようになるはずです。

- Alt_LのKeyCode: 64
- Alt_RのKeyCode: 108

(これらの値は`xev`コマンドで実際に確認できます。)

X11のキーイベント (KeyPress) には、KeyCodeの情報も含まれています。しかし、キーイベントは「今押されたキー」のKeyCodeしか教えてくれません。

### KeySym (論理的な記号)

KeySym (Key Symbol) という概念を説明します。[X11 Protocol: Keyboards](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#Keyboards)によると、KeySymはキーキャップに刻印された記号を符号化したものとあります。

> A KEYSYM is an encoding of a symbol on the cap of a key. The set of defined KEYSYMs include the character sets Latin-1, Latin-2, ..., and Korean as well as a set of symbols common on keyboards (Return, Help, Tab, and so on).

つまり、KeySymは以下のような特徴があります。

- KeyCodeに対して割り当てられる論理的な記号名 (`a`, `B`, `Return`など)

`Alt_L`と`Alt_R`も、それぞれにKeySymがあり、対応するKeyCodeを持っています。

### 問題の整理

ここまでの内容を整理すると、以下のようになります。

1. mのKeyPressイベントには、mのKeyCodeしか含まれず、Altキー自体のKeyCodeは含まれない
2. mのKeyPressイベントのstateフィールドには「Altが押されている」というModMaskの情報しかなく、左右の区別ができない
3. しかし、Alt_LとAlt_Rはそれぞれ異なるKeyCodeを持っている

## QueryKeymapで左右Altキーを判定する

### QueryKeymapの仕組み

X11 Protocolには、現在押されている全てのキーの状態を取得するための`QueryKeymap`リクエストがあります ([X11 Protocol: QueryKeymap](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#requests:QueryKeymap))。

以下では、Rustのx11rbライブラリにある`query_keymap()`関数を使って、X11アプリケーション (WindowManager) 側で左右のAltキーを判定する方法について書きます。

### WindowManagerでの実装例

以下のような流れで左右のAltキーを区別しています。

1. X11からKeyPressイベント (mキー) を受信
2. `query_keymap()` でその瞬間のキーボード全体の状態を取得
3. Alt_L (KeyCode 64) とAlt_R (KeyCode 108) のビットをチェック
4. どちらが押されているかに応じて異なるアプリケーションを起動

```rust
fn handle_key_press(
    conn: &impl Connection,
    event: KeyPressEvent,
    alt_l_keycode: u8,
    alt_r_keycode: u8,
) -> Result<(), Box<dyn std::error::Error>> {
    // mキー (KeyCode 58) が押された場合
    if event.detail == 58 {
        // その瞬間のキーボード状態を取得
        let reply = conn.query_keymap()?.reply()?;

        // Alt_LまたはAlt_Rが押されているかチェック
        if is_key_pressed(&reply.keys, alt_l_keycode) {
            // 左Alt + m → Chromeを起動
            std::process::Command::new("google-chrome").spawn()?;
        } else if is_key_pressed(&reply.keys, alt_r_keycode) {
            // 右Alt + m → Terminalを起動
            std::process::Command::new("terminal").spawn()?;
        }
    }
    Ok(())
}

// KeyCodeに対応するビットをチェック
fn is_key_pressed(keys: &[u8; 32], keycode: u8) -> bool {
    let byte_index = (keycode / 8) as usize;
    let bit_position = keycode % 8;
    (keys[byte_index] & (1 << bit_position)) != 0
}
```

### ビットチェックの仕組み

`is_key_pressed`関数では、以下の計算でKeyCodeに対応するビットをチェックしています。

```rust
let byte_index = (keycode / 8) as usize;    // どのバイトか
let bit_position = keycode % 8;              // バイト内の何ビット目か
(keys[byte_index] & (1 << bit_position)) != 0  // ビットが立っているか
```

例えば、Alt_L (KeyCode 64) の場合:

- `byte_index = 64 / 8 = 8`
- `bit_position = 64 % 8 = 0`
- `keys[8]`の0ビット目をチェック

この方法で、任意のKeyCodeの押下状態を確認できます。

## 他に方法はないのか

他のアプローチとして、以下のような方法も考えられます。

1. アプリケーション側での状態管理

Alt_LとAlt_RのKeyPressイベントを監視し、アプリケーション側で「どちらのAltが現在押されているか」を状態として保持する方法です。状態の不整合が起きないように、上手く管理する必要がありそうです。

2. xmodmapで別Modifierにマッピング

`xmodmap`などのツールを使って、Alt_LとAlt_Rを別々のModifierにマッピングする方法です。この場合、`query_keymap()`を使わずにModMaskだけで区別できます。しかし、システム全体の設定を変更するため影響範囲が大きそうです。

`QueryKeymap`のアプローチは、アプリケーション単体で完結し、状態管理の複雑さもありません。キーイベントの度にリクエストするコストはかかりますが、実用上のパフォーマンス問題はなかったため、このアプローチを採用しました。

## まとめ

X11で左右のAltキーを区別する方法として、`QueryKeymap`リクエストを使うアプローチを紹介しました。

キーイベントだけではModMaskレベルの情報しか得られませんが、`query_keymap()`を使えばKeyCodeレベルで全キーのPress/Release状態を取得できるため、左右の修飾キーを個別に扱うことができます。

他のウィンドウマネージャーでは、左右の修飾キーの区別をどのように実装しているのか、気になるところです。

## 参考

- [X11 Protocol: Common Types](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#Common_Types)
- [X11 Protocol: Keyboards](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#Keyboards)
- [X11 Protocol: QueryKeymap](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#requests:QueryKeymap)
- [x11rb - Rust X11 bindings](https://docs.rs/x11rb/)
