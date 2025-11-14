# [x] 想定読者を決める

- 開発環境のカスタマイズが好きで、その一環としてWindow Managerの自作に興味がある方
- Rustの基本的な文法は習得しているが、OSやX11のような低レイヤーの知識はこれから学びたい方
- Rustを使った実践的で低レイヤーのプログラムを書いてみたい方
- プロトコルの仕様書を読み解くことに (少しでも) 興味がある方

# [x] 本書のゴールを決める

1.  タイル型Window Managerの主要な機能 (ウィンドウのタイル配置、フォーカス制御、ショートカット機能など) をRustで実装し、仮想Xサーバ (Xephyr) 上で動作させるスキルを身につける。
2.  X Window System (X11) のクライアント・サーバモデルやイベント処理といった、基本的な仕組みを理解する。
3.  X11 Protocolの仕様書や関連ドキュメントを読み解く力を養い、本書の「機能追加のヒント」を足がかりに、自力で機能を実装する挑戦ができるようになる。

## [x] 「はじめに」に章構成と成果物を載せる

# [x] 章構成 (章タイトル・各章で何を書くか) を決める
## [x] X Window Systemの基礎知識
### [x] X Window Systemとはなにか

<!-- 読者がこのセクションを読み終えたとき -->
<!-- - X11という技術が何であるかを理解している -->
<!-- - 用語（X11, X Display Server, XCB, x11rb）が整理されている -->
<!-- - 次章以降で出てくる技術用語の基礎知識を持っている -->

- X Window System（X11）の定義
  - Unix-likeなOSでウィンドウを表示するためのシステム
  - 1980年代から使われている (年代は触れる程度)
- 名前のバリエーション
  - X Window System = X11 = X (すべて同じものを指す)
  - プロトコルバージョン11から「X11」と呼ばれる
  - 本書では「X11」で統一する
- 用語の整理 (階層構造で説明)
  - X11 Protocol: 仕様書 (ルールのセット)
  - X Display Server: サーバー実装
    - 例: Xorg (実機で使われる)
    - 例: Xephyr（仮想Xサーバー、本書で使用）
  - XCB/Xlib: C言語でX11を扱うライブラリ
  - x11rb: RustでX11を扱うライブラリ (本書で使用)
- 図を挿入
  - 上記の用語の関係性を示す
- Waylandへの言及
  - 近年はWaylandという新しいプロトコルも存在する
  - 本書では広く使われているX11を扱う

### [x] クライアント・サーバーモデル

<!-- 読者がこのセクションを読み終えたとき -->
<!-- - X11のクライアント・サーバーアーキテクチャを理解している -->
<!-- - 第2章で x11rb::connect を使う理由（サーバーに接続する必要がある）が理解できる -->

- X11はクライアント・サーバーモデルを採用している
  - クライアントとは
    - アプリケーション（ターミナル、ブラウザ、Window Managerなど）
    - 「ウィンドウを表示したい」「キーイベントを受け取りたい」などのリクエストをサーバーに送信する
  - サーバーとは
    - X Display Server（Xorg, Xephyrなど）
    - 画面・キーボード・マウスなどのハードウェアを管理
    - クライアントからの要求を処理して、画面に描画する
  - 通信方法
    - X11 Protocolでやりとりする
    - クライアントはサーバーに接続する必要がある
  - Window Managerの位置づけ
    - Window Managerも単なるクライアントの一種
    - ただしクライアントからのイベントを横取りできる特別な権限を持つ（詳細は後のセクションで）
  - 図を挿入
    - クライアント（アプリ群、WM）とサーバー（画面・デバイス管理）の関係

### [x] Display, Screen, Windowの概念

<!-- 読者がこのセクションを読み終えたとき -->
<!-- - display, screen, root-window の定義を理解している -->
<!-- - これらの関係性を理解している -->
<!-- - 第2章で screen_num や roots を扱うコードが理解できる -->

- display とは
  - キーボード、マウス、screen をまとめたもの
  - `x11rb::connect()` で接続する対象

- screen とは
  - 物理モニターに対応する概念
  - 1つのdisplayには複数のscreenが含まれる可能性がある

- root window とは
  - 各screenには1つのroot windowがある
  - すべてのウィンドウはroot windowの子孫として階層的に存在する
  - WMが管理するのは、root windowの子ウィンドウたち (タイリングWMだと、直下のトップレベルWindow)

- 補足 (Note/コラム形式):
  - 現代では、複数の物理モニターが1つの仮想screenとして扱われる
  - そのため、ほとんどの環境では `screen_num = 0` の1つだけが存在する
  - したがって、root windowも1つだけになる
  - 本書では単一screen環境を前提とする

- 図を挿入
  - display, screen, root window, windows の関係

### [x] Window Managerとは

<!-- 読者がこのセクションを読み終えたとき -->
<!-- - SubstructureRedirect と SubstructureNotify という概念を理解している -->
<!-- - Window Managerが他のクライアントとどう違うかが分かっている -->
<!-- - 第3章でこれらのマスクを設定するコードの意味が理解できる -->

- Window Managerの特別な役割
  - 通常のクライアント（ターミナル、ブラウザなど）は自分のウィンドウだけを管理
  - Window Managerは、他のクライアントのウィンドウを管理する

- SubstructureRedirect とは
  - root windowの子ウィンドウに対する操作リクエストをWMにリダイレクトする仕組み
  - 例: アプリが「ウィンドウを表示したい」とリクエストしたとき、WMがそれを受け取って配置を決められる
  - root windowに対してSubstructureRedirectマスクを設定する

- SubstructureNotify とは
  - root windowの子ウィンドウで起きたイベントをWMに通知する仕組み
  - 例: アプリのウィンドウが閉じられたとき、WMがそれを検知できる
  - root windowに対してSubstructureNotifyマスクを設定する

- 設定について
  - どちらも同時に1つのクライアントしか持てない権限（複数のWMは起動できない）

- 図を挿入
  - Redirect: クライアント → (リクエスト) → WM → (処理後) → Xサーバー
  - Notify: Xサーバー → (イベント通知) → WM

## [x] 環境構築

- Rust, x11rb, Xephyrのセットアップ
- `x11rb::connect` でXサーバーに接続する最小限のコードを書く

## [x] 最小限のウィンドウマネージャー

- WindowManagerの初期化 (`SubstructureRedirect` マスクの設定)
- イベントループの実装 (`wait_for_event`, `handle_event`)
- イベントのハンドリング (MapRequest, UnmapNotify, ConfigureNotify)

## [x] タイル型レイアウトの実装

- Windowのリストを管理する
- master-stackレイアウトを実装する

## [x] キーイベントのハンドリング

- KeyGrab, Modifierの仕組み
- フォーカスとボーダーの概念
- フォーカス移動の実装 (Mod+j/k)
- フォーカスのあるウィンドウをマスターと交換 (Mod+m)

## [x] 機能追加のアイデアと参考資料

- 機能追加のアイデアを伝える
- X11 Protocol, Xlib Programming Manual, Xlib C Programming Interface, ICCCM, EWEHなどのリファレンスを紹介する
# [x] 第1章を執筆する

# [x] 第2章を執筆する
## はじめに

- この章のゴール: Xephyr上でrwmを動かし、xtermを表示する
- 動作確認済み環境(Ubuntu 24.04など)

## 必要なツールのインストール

- sudo apt install xserver-xephyr
- sudo apt install xterm(または他のテスト用Xアプリ)

## Rustプロジェクトのセットアップ

- cargo new rwm
- cargo add x11rb anyhow tracing tracing-subscriber
- Cargo.tomlの確認

## Xephyrの動作確認

- Xephyrとは何か(ネストされたXサーバー)
- Xephyr :10 -screen 1280x720で起動
- 別ターミナルからDISPLAY=:10 xtermで接続確認
- $DISPLAYの役割
- フォーマット: [hostname]:displaynumber[.screennumber]
- 例: :0, :10, localhost:10.1
- なぜ:10を使うか

## Xサーバーへの接続

- x11rb::connect(None)のコード
- Noneの意味($DISPLAYを使う)
- screen_numの意味(デフォルトscreen番号)
- ログ出力の確認

## 動作確認スクリプト

- test.shの全体像
- 各コマンドの説明(Xephyr起動、rwm起動、xterm起動)
- スクリーンショット
- 期待される結果

## まとめ

- この章で達成したこと
- 次章の予告(最小限のWM実装)

# [x] 第3章を執筆する

# [ ] 第1章にシーケンス図を追加する

- SubstructureRedirect の仕組みを説明するセクションに、クライアント・WM・Xサーバー間のイベントの流れを示すシーケンス図を追加する
- 例: クライアントがウィンドウを作成しようとしたとき、リクエストがどのように WM にリダイレクトされ、WM が処理した後に X サーバーに送信されるか

# 第4章を執筆する

# 第5章を執筆する

# 第6章を執筆する
