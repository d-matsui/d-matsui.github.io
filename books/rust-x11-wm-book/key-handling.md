---
title: "キーイベントのハンドリング"
---

## はじめに

この章では、キーボードショートカットによるウィンドウ操作を実装します。これにより、X11 におけるキーイベントのハンドリング方法を学びます。

具体的には、以下の機能を実装します。

- Alt+j/k でウィンドウのフォーカスを移動する
- フォーカス中のウィンドウをボーダー色で区別する
- Alt+m でフォーカス中のウィンドウをマスターウィンドウとスワップする

<!-- TODO: デモGIFを入れる -->

## X11 におけるキーボードの扱い

<!--
- 全体像
  - 通常のキーイベントはフォーカス中のウィンドウに届く
  - GrabKey で特定のキーを横取りできる (詳細は次セクション)
  - KeyPress イベントには KeyCode と Modifier が含まれる
- KeyCode と KeySym
  - Why: キーボードはメーカーや配列によって物理的な構成が異なる
    - KeyCode: デバイス依存の物理キー識別子
    - KeySym: 論理的な意味 (環境非依存)
    - この分離により、アプリは「j が押された」という論理的意味だけを気にすればよい
  - 具体例: j キー → KeyCode 44、KeySym XK_j
  - 注釈: KeySym は X11/keysymdef.h に定義されている
  - 本章では簡略化のため KeyCode を直接使う (環境依存になる)
- Modifier
  - 修飾キー: Shift, Ctrl, Alt など
  - X11 では Mod1〜Mod5 として抽象化されている
  - 主要な対応: Shift, Ctrl, Alt (Mod1), Super (Mod4) など
  - 環境によって対応が異なる可能性がある
-->

## キーバインドの設定

<!--
- GrabKey
  - 特定のキーを Window Manager が横取りする仕組み
  - KeyboardGrab との違い: GrabKey は特定キーのみ、KeyboardGrab は全キー
- Action 列挙型
  - FocusPrev, FocusNext, SwapMaster を定義
- HashMap<(ModMask, u8), Action>
  - キーバインドを管理するデータ構造
  - 設計意図: キーとアクションの対応を一元管理
- register_keybindings()
  - grab_key() で X サーバに登録
-->

## フォーカスの移動

<!--
- フォーカスとは
  - focus window とその子孫がキーボード入力を受け取る
  - デフォルトは root window なので全ウィンドウに届く
  - WM が set_input_focus() で明示的にフォーカスを設定する
- KeyPress イベントのハンドリング
  - detail: keycode (どのキーが押されたか)
  - state: modifier (どの修飾キーが押されていたか)
  - x11rb での受け取り方と処理
- focused_window: Option<u32>
  - WM 側でフォーカス状態を管理
- focus_next/prev
  - windows リストのインデックスを循環させて次/前を選択
  - set_input_focus() で X サーバに通知
- フォーカス状態の視覚化
  - window_attributes: タイリングWM観点で主要なものを紹介
  - border_width: ボーダーの太さ
  - border_pixel: ボーダーの色
  - 図: ボーダーがレイアウト計算にどう関わるか
  - change_window_attributes() で色を変更
-->

## マスターウィンドウとのスワップ

<!--
- フォーカス中のウィンドウをマスター位置に移動
- windows.swap(0, idx) でリスト内の位置を入れ替え、再配置
- 既にマスターなら何もしない
-->

## まとめ

<!--
- この章で学んだこと
  - X11 の概念: KeyCode/KeySym, Modifier, GrabKey, フォーカス
  - 実装したこと: キーバインド設定、フォーカス移動、ボーダー表示、スワップ
- ここまでで基本的なタイル型 WM が完成した
- 次章では発展的な機能追加のアイデアや参考資料を紹介
-->
