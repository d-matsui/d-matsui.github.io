---
title: "タイル型レイアウトの実装"
---

## はじめに

この章では、これまでの章で作成した Window Manager にタイリングレイアウト機能を実装します。

タイリングレイアウトでは、ウィンドウを自動的にタイル状に配置し、画面領域を効率的に使います。ウィンドウの追加・削除に応じて、Window Manager が自動的にウィンドウのサイズと位置を調整します。

本章では、シンプルなタイリングレイアウトである master-stack レイアウトを実装します。master-stack レイアウトは、タイリングレイアウトの中でも最もシンプルな形式の1つです。このシンプルなレイアウトを題材に、Window Manager が X Server からのイベントをどのようにハンドリングするかを学びます。

<!-- TODO: master-stack レイアウトで Window が管理されている gif を挿入する。 -->

## master-stackレイアウトの概要

master-stack レイアウトでは、1つの master ウィンドウを画面の左半分に配置し、残りのウィンドウは右半分に stack として縦に並べます。以下の図で具体的な配置を見ていきましょう。

ウィンドウの数が1つのとき、そのウィンドウは画面の全領域を使用します。

![master-stackレイアウト: 1つのウィンドウ](/images/master-stack-1window.png)

ウィンドウの数が2つのとき、画面を左右に分割します。左側に master、右側に stack を配置します。

![master-stackレイアウト: 2つのウィンドウ](/images/master-stack-2windows.png)

ウィンドウの数が3つのとき、master は左側に配置し、右側の stack 領域を2つに分割します。

![master-stackレイアウト: 3つのウィンドウ](/images/master-stack-3windows.png)

ウィンドウの数が4つのとき、master は左側に配置し、右側の stack 領域を3つに分割します。

![master-stackレイアウト: 4つのウィンドウ](/images/master-stack-4windows.png)

## master-stackレイアウトのアルゴリズム

master-stack レイアウトの計算方法を図で示します。

![master-stackレイアウトのアルゴリズム](/images/master-stack-algorithm.png)

画面を左右に 1/2 ずつ分割し、左側を master、右側を stack 領域とします。stack 領域は、ウィンドウの個数に応じて縦方向に均等分割します (この例では 1/3 ずつ)。

具体的な計算方法は以下の通りです。

### master ウィンドウ

- 位置: 画面左上 `(0, 0)`
- 幅: 画面幅の 1/2 (ウィンドウが1つの場合は全幅)
- 高さ: 画面の高さ全体

### stack ウィンドウ

- 位置: 画面幅の 1/2 から開始、縦方向に均等配置
- 幅: 画面幅の 1/2
- 高さ: 画面の高さ ÷ stack の個数

## master-stackレイアウトの実装

X Server から送られてくるイベントに応じてレイアウトを計算することで、master-stack レイアウトを実現します。具体的には、ウィンドウ表示のイベント (MapRequest) でウィンドウをリストに追加し、非表示のイベント (UnmapNotify) でリストから削除します。それぞれのタイミングでレイアウトを計算し、X Server にウィンドウの配置を指示します。

### window の追加

MapRequest イベントの処理では、新しいウィンドウをリストに追加し、レイアウトを再計算します。

```rust
fn handle_map_request(&mut self, event: &MapRequestEvent) -> Result<()> {
    info!("[MapRequest] win={}", event.window);
    // add requested window to managed window list
    self.windows.push(Window::new(event.window));

    // calculate tiling layout
    self.calculate_layout();

    // request to X server
    for window in &self.windows {
        let change = ConfigureWindowAux::default()
            .x(window.x)
            .y(window.y)
            .width(window.width)
            .height(window.height);
        self.conn.configure_window(window.id, &change)?.check()?;
    }

    let new_window = self
        .windows
        .last()
        .expect("Window list should not be empty");

    self.conn.map_window(new_window.id)?.check()?;

    Ok(())
}
```

`calculate_layout()` ですべてのウィンドウの位置とサイズを計算した後、`ConfigureWindow` で配置を更新し、`MapWindow` で新しいウィンドウを表示します。

### window の削除

UnmapNotify イベントの処理では、ウィンドウをリストから削除し、レイアウトを再計算します。

```rust
fn handle_unmap_notify(&mut self, event: &UnmapNotifyEvent) -> Result<()> {
    info!("[UnmapNotify] window={}", event.window);
    self.windows.retain(|w| w.id != event.window);

    self.calculate_layout();

    for window in &self.windows {
        let change = ConfigureWindowAux::default()
            .x(window.x)
            .y(window.y)
            .width(window.width)
            .height(window.height);
        self.conn.configure_window(window.id, &change)?.check()?;
    }

    Ok(())
}
```

`calculate_layout()` で残りのウィンドウの位置とサイズを計算した後、`ConfigureWindow` ですべてのウィンドウの配置を更新します。

### レイアウトの計算

`calculate_layout()` では、リスト内のすべてのウィンドウに対して位置とサイズを計算します。インデックス 0 のウィンドウを master として、それ以降を stack として配置します。

```rust
fn calculate_layout(&mut self) {
    const MASTER_RATIO: f32 = 0.5;
    let num_windows = self.windows.len() as u32;
    let master_width = (self.screen_width as f32 * MASTER_RATIO) as u32;

    for (idx, window) in self.windows.iter_mut().enumerate() {
        if idx == 0 {
            // master
            window.x = 0;
            window.y = 0;
            window.width = if num_windows == 1 {
                self.screen_width
            } else {
                master_width
            };
            window.height = self.screen_height;
        } else {
            // stack
            let stack_count = num_windows - 1;
            let stack_height = self.screen_height / stack_count;
            let stack_index = (idx - 1) as u32;

            window.x = master_width as i32;
            window.y = (stack_height * stack_index) as i32;
            window.width = self.screen_width - master_width;
            window.height = stack_height;
        }

        debug!(
            "layout win={}: {}x{} pos({},{})",
            window.id, window.width, window.height, window.x, window.y
        );
    }
}
```

`MASTER_RATIO` 定数で master の幅を画面幅の半分に設定しています。ウィンドウが1つの場合は、master が全画面を使用します。stack ウィンドウは、個数に応じて縦方向に均等分割されます。

## まとめ

この章では、master-stack レイアウトを実装し、タイリング機能を持つ Window Manager を完成させました。

- master-stack レイアウトの仕組みを理解した
- MapRequest と UnmapNotify のイベントハンドリングを実装した

この実装を通じて、Window Manager がイベントに応じてウィンドウを配置する仕組みを学びました。

現在の実装では、ウィンドウは自動的に配置されますが、ユーザーが操作する手段がありません。次の章では、KeyPress イベントをハンドリングして、キーボードでウィンドウを操作する機能を実装します。
