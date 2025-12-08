---
title: "if-elseの代わりにHashMapを使ってコードをシンプルにする"
emoji: "🔧"
type: "tech"
topics: ["rust", "プログラミング", "設計", "リファクタリング"]
published: true
---

## はじめに

if-else の条件式が複雑になって、コードが読みにくくなったこと、あると思います。

この記事では、if-else を HashMap に置き換えることで、コードをシンプルにする方法を紹介します。

**要約**
- 条件と処理の対応を HashMap で管理する
- 対応が一目でわかり、コードもシンプルになる

---

最近、X11 Window Manager をいちから実装する本を Zenn で公開しました。

https://zenn.dev/d_matsui/books/rust-x11-wm-book

その本の中で、`Alt + j` でウィンドウのフォーカスを移動するという機能の実装を書きました。

この記事では、その実装で考えたことを整理します。素朴な実装の問題点と、その改善策を具体的なコードで見ていきます。

## 素朴な実装とその問題点

まず、素朴に実装するとどうなるかを見てみます。

キーが押されたときのイベント処理で、`if-else` を使って分岐するコードは以下のようになります。

```rust
fn handle_key_press(&mut self, event: &KeyPressEvent) -> Result<()> {
    let modifier = event.state;
    let keycode = event.detail;

    if modifier == ModMask::M1 && keycode == KEY_J {
        self.focus_next()?;
    } else if modifier == ModMask::M1 && keycode == KEY_K {
        self.focus_prev()?;
    } else if modifier == ModMask::M1 && keycode == KEY_M {
        self.swap_master()?;
    }

    Ok(())
}
```

このコードには、読みやすさの観点からいくつか問題があります。

1. Modifier (修飾キー) の組み合わせが増えると条件分岐が複雑になる。今は Alt (M1) だけだが、Ctrl, Super, Shift なども使うと、条件式が長くなり読みにくくなる。
2. キーとアクションの対応付けが一覧できない。どのキーが何をするかを確認するには、`if-else` のコードを上から順に読んでいく必要がある。

## 解決策

これらの問題を解決するために、キーの組み合わせ (Modifier, KeyCode) とアクションの対応を HashMap で管理します。

```rust
enum Action {
    FocusNext,
    FocusPrev,
    SwapMaster,
}

struct WindowManager {
    keybindings: HashMap<(ModMask, Keycode), Action>,
    // ...
}
```

キーバインドの設定は、HashMap で定義するだけです。

```rust
let keybindings = HashMap::from([
    ((ModMask::M1, KEY_J), Action::FocusNext),
    ((ModMask::M1, KEY_K), Action::FocusPrev),
    ((ModMask::M1, KEY_M), Action::SwapMaster),
]);
```

この設定を見れば、どのキーが何をするかが一目でわかります。

イベント処理は以下のように書けます。

```rust
fn handle_key_press(&mut self, event: &KeyPressEvent) -> Result<()> {
    let modifier = event.state;
    let keycode = event.detail;

    if let Some(action) = self.keybindings.get(&(modifier, keycode)).copied() {
        match action {
            Action::FocusNext => self.focus_next()?,
            Action::FocusPrev => self.focus_prev()?,
            Action::SwapMaster => self.swap_master()?,
        }
    }

    Ok(())
}
```

HashMap からアクションを取得し、match で処理を振り分けています。Modifier の組み合わせがどれだけ増えても、この関数の構造は変わりません。

## まとめ

キーバインドを HashMap で管理することで、コードが読みやすくなりました。

- キーとアクションの対応が一覧できる
- イベント処理のコードがシンプルになる

また、この設計なら将来、設定ファイルからキーバインドを読み込む形にも拡張しやすいです。

調べてみると、これに似たパターンとして Table-Driven Methods というものがあるようです。

Table-Driven Methods ではデータ構造に関数を直接入れておき、取得してそのまま呼び出します。今回の例で言えば、HashMap の value に enum ではなく関数を入れる形です。そうすると match が不要になりシンプルに書けます。

今回は enum を使ったので、match による網羅性チェックが効くというメリットがあります。
