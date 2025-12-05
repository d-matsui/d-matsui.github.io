# キーイベントのハンドリング

## はじめに

- この章で実装すること
  - Alt+j/k でフォーカス移動
  - フォーカス中のウィンドウをボーダー色で区別
  - Alt+m でマスターウィンドウとスワップ
- この章で学ぶこと
  - X11 におけるキーイベントのハンドリング方法

## キーボードショートカットの実現方法

- grab_key によるキーイベントの取得
  - grab_key() で特定のキーの組み合わせを WM 宛に届ける
- KeyPress イベントの構成要素
  - KeyCode: 押されたキーを表す数値 (環境依存)
  - KeySym: キーの意味を表す値 (本書では KeyCode を直接使用)
  - state (Modifier): 修飾キーのビットマスク

## キーバインドの実装

- Action 列挙型
  - FocusPrev, FocusNext, SwapMaster を定義
- HashMap<(ModMask, u8), Action>
  - キーバインドを管理するデータ構造
- register_keybindings()
  - grab_key() で X サーバに登録

## フォーカスの移動

- フォーカスの概念
  - キーボード入力はフォーカスウィンドウ (とその子孫) に届く
  - WM が set_input_focus() でフォーカスを設定
- フォーカス移動の実装
  - `focused_window`: Option<u32> フィールド追加
  - `focus_window()`, `focused_index()` 関数
  - `focus_next/prev()`: 循環させて次/前を選択

## フォーカス状態の表示

- ボーダーの概念
  - border_width: ボーダーの太さ (ピクセル単位)
  - border_pixel: ボーダーの色 (RGB値)
  - ボーダーはウィンドウの外側に描画 → レイアウト計算で考慮が必要
  - calculate_layout() の修正
- ボーダー色変更の実装
  - 定数定義 (`BORDER_WIDTH`, `BORDER_COLOR_FOCUSED`, `BORDER_COLOR_UNFOCUSED`)
  - `focus_window()` に処理追加
  - `handle_map_request()` の修正

## マスターウィンドウとのスワップ

- swap_master の実装
  - フォーカス中のウィンドウをマスター位置に移動
  - `windows.swap(0, idx)` でリスト内の位置を入れ替え
  - `calculate_layout()` + `apply_layout()` で再配置
  - 既にマスターなら何もしない

## まとめ

- 実装したこと
  - キーバインドの登録
  - Alt+j / Alt+k によるフォーカス移動
  - ボーダー色によるフォーカス状態の表示
  - Alt+m によるマスターウィンドウとのスワップ
- 学んだこと
  - `grab_key()` によるキーイベントの取得
  - KeyCode と Modifier によるキーの識別
  - `set_input_focus()` によるフォーカス制御
  - ボーダーによるフォーカス状態の表示
- 次章への橋渡し
  - ここまでで基本的なタイル型 WM が完成した
  - 次章では発展的な機能追加のアイデアや参考資料を紹介
