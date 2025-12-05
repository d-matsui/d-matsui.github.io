---
title: "キーイベントのハンドリング"
---

## はじめに

この章では、キーボードショートカットでウィンドウの配置を操作する機能を実装します。

前章で、ウィンドウを自動的にタイル配置する機能を実装しました。しかし、現状ではウィンドウの配置順序を変更したり、特定のウィンドウを選択したりする手段がありません。タイル型 Window Manager では、キーボードショートカットでこれらの操作を行うのが一般的です。

そこで本章では、以下の機能を実装します。

- Alt+j / Alt+k でフォーカスウィンドウを切り替える
- フォーカスウィンドウをボーダー色で視覚的に区別する
- Alt+m でフォーカスウィンドウをマスターの位置にスワップする

これらの実装を通じて、X11 におけるキーイベントのハンドリング方法を学びます。

<!-- TODO: この章の機能を示したGIFを入れる-->

## キーボードショートカットの実現方法

### grab_key によるキーイベントの取得

Window Manager がキーボードショートカットを実装するには、特定のキーの組み合わせを自分宛てに届くようにする必要があります。これを実現するのが `grab_key()` です。

`grab_key()` を使って root window に対してキーと modifier の組み合わせを grab すると、その組み合わせが押されたときに KeyPress イベントを受け取れるようになります。

### KeyPress イベントの構成要素

KeyPress イベントには、押されたキーを識別するための情報として detail (KeyCode), state (Modifier) が含まれています。たとえば Alt+j が押されると、以下のようなイベントが届きます。

| フィールド | 値 |
|---|---|
| detail | 44 (j) |
| state | Mod1 (Alt) |
| ... | |

KeyCode は、押されたキーを表す数値です。一般的な QWERTY キーボードでは、j の位置にあるキーを押すと KeyCode 44 が返ってきます。

KeySym は、押されたキーの意味を表す値です。例えば KeyCode 44 は KeySym `XK_j` にマッピングされます。

本書では簡略化のため、KeySym への変換はせず、KeyCode を直接使用します。

KeyPress イベントにおける state は、イベント発生直前にどの modifier (Shift, Ctrl, Alt など) が押されていたかを示すビットマスクです。本章では Alt キー (Mod1) を modifier として使用します。

## キーバインドの実装

キーボードショートカットに対応する操作を `Action` 列挙型として定義します。

```rust
enum Action {
    FocusPrev,
    FocusNext,
    SwapMaster,
}
```

キーバインドは、Modifier と KeyCode の組み合わせを `Action` に対応付けた `HashMap` で管理することにします。

```rust
use std::collections::HashMap;
use x11rb::protocol::xproto::ModMask;

struct WindowManager {
    // ... 既存のフィールド ...
    keybindings: HashMap<(ModMask, u8), Action>,
}
```

`WindowManager::new()` 内でキーバインドを初期化します。

```rust
let keybindings = HashMap::from([
    ((ModMask::M1, 44), Action::FocusNext),  // Alt+j
    ((ModMask::M1, 45), Action::FocusPrev),  // Alt+k
    ((ModMask::M1, 58), Action::SwapMaster), // Alt+m
]);
```

Window Manager の初期化後に、`grab_key()` を呼び出してキーバインドを X サーバに登録します。これにより、指定したキーの組み合わせが押されたときに KeyPress, KeyRelease イベントを受け取れるようになります。

```rust
fn register_keybindings(&mut self) -> Result<()> {
    for (modifier, keycode) in self.keybindings.keys() {
        self.conn.grab_key(
            true,
            self.root_window,
            *modifier,
            *keycode,
            GrabMode::ASYNC,
            GrabMode::ASYNC,
        )?;
    }

    Ok(())
}
```

main 関数では、`WindowManager::new()` の後に `register_keybindings()` を呼び出します。

```rust
let mut wm = WindowManager::new(conn, screen_num)?;
wm.register_keybindings()?;
wm.run()?;
```

:::message
ここで使用しているキーコード (44, 45, 58) は環境依存の値です。`xev` コマンドを使って、自分の環境でのキーコードを確認できます。
:::

## フォーカスの移動

### フォーカスの概念

X11 では、キーボード入力はフォーカスウィンドウ (とその子孫) に届きます。Window Manager は `set_input_focus()` を使って、特定のウィンドウにフォーカスを設定します。これにより、キーボード入力の送り先を明示的に指定できます。

### フォーカス移動の実装

まず、`WindowManager` 構造体にフォーカス中のウィンドウを管理するフィールドを追加します。

```rust
struct WindowManager {
    // ... 既存のフィールド ...
    focused_window: Option<u32>,
}
```

フォーカスを設定する関数を実装します。

```rust
fn focus_window(&mut self, window_id: u32) -> Result<()> {
    self.conn
        .set_input_focus(InputFocus::PARENT, window_id, CURRENT_TIME)?;
    self.focused_window = Some(window_id);
    Ok(())
}
```

現在フォーカスしているウィンドウのインデックスを取得するヘルパー関数を用意します。

```rust
fn focused_index(&self) -> Option<usize> {
    self.windows
        .iter()
        .position(|w| Some(w.id) == self.focused_window)
}
```

フォーカスを次/前のウィンドウに移動する関数を実装します。ウィンドウリストの末尾に達したら先頭に戻り、先頭から前に移動したら末尾に戻るように循環させます。

```rust
fn focus_next(&mut self) -> Result<()> {
    if self.windows.is_empty() {
        return Ok(());
    }

    if let Some(idx) = self.focused_index() {
        let next_idx = (idx + 1) % self.windows.len();
        let next_window = self.windows[next_idx].id;
        self.focus_window(next_window)?;
    }

    Ok(())
}

fn focus_prev(&mut self) -> Result<()> {
    if self.windows.is_empty() {
        return Ok(());
    }

    if let Some(idx) = self.focused_index() {
        let prev_idx = if idx == 0 {
            self.windows.len() - 1
        } else {
            idx - 1
        };
        let prev_window = self.windows[prev_idx].id;
        self.focus_window(prev_window)?;
    }

    Ok(())
}
```

## フォーカス状態の表示

フォーカスの切り替えだけでは、どのウィンドウがフォーカスされているか視覚的にわかりません。フォーカス中のウィンドウをボーダー色で区別できるようにします。

### ボーダーの概念

X11 のウィンドウにはボーダーを設定できます。ボーダーに関連するウィンドウの属性は以下の2つです。

- border_width: ボーダーの太さ (ピクセル単位)
- border_pixel: ボーダーの色 (RGB値)

ボーダーはウィンドウの外側に描画されます。そのため、レイアウト計算時にはボーダーの幅を考慮してウィンドウサイズを調整する必要があります。前章で実装した `calculate_layout()` を以下のように修正します。

```rust
fn calculate_layout(&mut self) {
    let num_windows = self.windows.len() as u32;
    let master_width = (self.screen_width as f32 * MASTER_RATIO) as u32;

    for (idx, window) in self.windows.iter_mut().enumerate() {
        if idx == 0 {
            window.x = 0;
            window.y = 0;
            window.width = if num_windows == 1 {
                self.screen_width - BORDER_WIDTH * 2
            } else {
                master_width - BORDER_WIDTH * 2
            };
            window.height = self.screen_height - BORDER_WIDTH * 2;
        } else {
            let stack_count = num_windows - 1;
            let stack_height = self.screen_height / stack_count;
            let stack_index = (idx - 1) as u32;

            window.x = master_width as i32;
            window.y = (stack_height * stack_index) as i32;
            window.width = self.screen_width - master_width - BORDER_WIDTH * 2;
            window.height = stack_height - BORDER_WIDTH * 2;
        }
    }
}
```

各ウィンドウの width と height から `BORDER_WIDTH * 2` を引いています。左右 (または上下) 両側にボーダーがあるため、2倍する必要があります。

### ボーダー色変更の実装

ボーダーの太さと色を定数として定義します。

```rust
const BORDER_WIDTH: u32 = 5;
const BORDER_COLOR_FOCUSED: u32 = 0xFF0000;   // 赤
const BORDER_COLOR_UNFOCUSED: u32 = 0x000000; // 黒
```

`focus_window()` の先頭に、ボーダー色を更新する処理を追加します。

```rust
fn focus_window(&mut self, window_id: u32) -> Result<()> {
    if let Some(prev) = self.focused_window {
        let attr = ChangeWindowAttributesAux::default().border_pixel(BORDER_COLOR_UNFOCUSED);
        self.conn.change_window_attributes(prev, &attr)?;
    }

    let attr = ChangeWindowAttributesAux::default().border_pixel(BORDER_COLOR_FOCUSED);
    self.conn.change_window_attributes(window_id, &attr)?;

    // ...
}
```

新規ウィンドウが表示されるときにボーダーの太さを設定し、そのウィンドウにフォーカスを当てます。`handle_map_request()` を以下のように修正します。

```rust
fn handle_map_request(&mut self, event: &MapRequestEvent) -> Result<()> {
    self.windows.push(Window::new(event.window));
    self.calculate_layout();
    self.apply_layout()?;

    let geom = ConfigureWindowAux::default().border_width(BORDER_WIDTH);
    self.conn.configure_window(event.window, &geom)?.check()?;

    self.conn.map_window(event.window)?.check()?;

    self.focus_window(event.window)?;

    Ok(())
}
```

## マスターウィンドウとのスワップ

本書で実装した master-stack レイアウトでは、ウィンドウリストの先頭がマスターウィンドウとして左側に大きく表示されます。Alt+m でフォーカス中のウィンドウをマスター位置に移動できるようにします。

```rust
fn swap_master(&mut self) -> Result<()> {
    if let Some(idx) = self.focused_index()
        && idx != 0
    {
        self.windows.swap(0, idx);
        self.calculate_layout();
        self.apply_layout()?;
    }

    Ok(())
}
```

フォーカス中のウィンドウがすでにマスター位置 (index 0) にある場合は何もしません。それ以外の場合、`swap()` でリスト内の位置を入れ替え、レイアウトを再計算して適用します。

## まとめ

この章では、キーボードショートカットによるウィンドウの操作を実装しました。

具体的には、
- キーバインドの登録
- Alt+j / Alt+k によるフォーカス移動
- ボーダー色によるフォーカス状態の表示
- Alt+m によるマスターウィンドウとのスワップ
を実装しました。

また、実装を通して、以下を学びました。

- `grab_key()` によるキーイベントの取得
- KeyCode と Modifier によるキーの識別
- `set_input_focus()` によるフォーカス制御
- ボーダーによるフォーカス状態の表示


ここまでで、基本的なタイル型 Window Manager が完成しました。次章では、さらなる機能追加のアイデアや参考資料を紹介します。
