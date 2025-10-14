---
author: Daiki Matsui
pubDatetime: 2025-10-14T10:26:11Z
title: "AIに作らせたコードからElispを学ぶ"
slug: learning-elisp-with-ai
featured: true
draft: false
tags:
  - elisp
  - emacs
  - ai
  - learning
description: "Claude Codeに生成させたコードを題材にして、Elisp特有の構文や概念を学んでいくアプローチ。"
---

## きっかけ

私は大学3年生のときからEmacsを使っています。当時所属していた研究室の指導教員のすすめで、「なんだかかっこよさそう」と思って使い始めたのを覚えています。

長年使ってきたとはいえ、使いこなせているとは言えません。インターネットで便利な機能を検索して自分のEmacsにインストールし、少し設定を変えるというのが、私の主なEmacsの使い方でした。

これまで、Elispを学ぶためにいくつかの本や既存のコードを読んでみましたが、あまり身につきませんでした。関数型言語に慣れていないことや、Elispのデバッグ方法がわからなかったこともあり、次第にモチベーションが下がっていったのだと思います。

そんな中、AIを活用した新しい学習方法を試してみることにしました。教科書やマニュアルを読むのは個人的に楽しめなかったので、どうせなら自分が欲しいものを題材にして、実践的に学んでみようと考えたのです。

この記事では、Claude Codeに生成させたコードを題材にElispを学んだ過程について書きます。

## 何を作ったか

Claude Codeに、日本語文章を添削してくれるシステム (sakubun-checker) を作ってもらいました。Geminiの無料API (2.5 Flash) を使用して、日本語の作文技術の独自ルールに基づいて日本語文章を評価してくれます。

![sakubun-checkerでこの記事を添削する様子](/sakubun-checker-demo.png)

Elisp, Pythonスクリプト, 日本語作文技術のルールをまとめたプロンプトの3つで構成されています。Elispは入出力を担当し、PythonスクリプトはGeminiへのAPIリクエストを担当しています。プロンプトには「修飾・被修飾関係の言葉を近付ける」といったルールを記述しつつ、マークダウン形式で添削結果を出力するようにしています。

## コードを読み解く

Claude Codeが生成したコードを読み解きながら、Elisp特有の概念や慣習を学んでいきました。開発途中のコードも含めて、特に印象的だった学びをいくつか紹介します。

### 外部プロセスの実行 (PythonスクリプトとのI/O)

ElispからPythonスクリプトを呼び出す方法を学びました。

最初は`call-process`で同期的に実行していましたが、Gemini APIからのレスポンスに数秒かかってEmacsがハングするので、`make-process`を使って非同期処理に変更しました。

```elisp
;; 同期処理版
(call-process "python3" nil output-buffer nil "check.py" text)

;; 非同期処理版
(make-process
  :command (list "python3" "check.py" text)
  :sentinel (lambda (proc event)
              ;; 完了時にコールバックが呼ばれる
              (funcall callback result)))
```

非同期処理版では、`sentinel` にコールバック関数を渡すことで、文章チェック実行中もEmacsを操作できるようになります。

### レキシカルスコープ (クロージャを使うための設定)

非同期版を実装したときに遭遇したエラー:

```
Symbol's value as variable is void: callback
```

原因は、Emacs Lispのデフォルトが「動的スコープ」で、lambda式が外側の変数をキャプチャできないことでした。

```elisp
;;; -*- lexical-binding: t -*-
```

ファイルの先頭にこの1行を追加すると、JavaScriptやPythonのようなレキシカルスコープ（クロージャ）が使えるようになります。

動的スコープは1980年代のLispの名残で、今では後方互換性のために残されているだけとのことでした。こういうのなんだか面白い。

### エラーハンドリング (try-finally)

メモリリークを防ぐために、途中でエラーが起きてもクリーンアップの処理は必ず実行したいというユースケースがよくあります。

```elisp
(let ((buffer (generate-new-buffer "*temp*")))
  (unwind-protect
      ;; メイン処理（エラーが起きるかも）
      (call-process ...)
    ;; 必ず実行されるクリーンアップ
    (kill-buffer buffer)))
```

`unwind-protect`は、JavaScriptの`try-finally`やPythonの`try-except-finally`と同じ役割で、正常終了でもエラーでも、必ず処理を実行してくれます。

### データ構造 (cons cellでの値の返却)

Elispでは、成功/失敗と結果データを同時に返すために`cons cell`が使えます。

```elisp
;; 成功時
(cons 0 "チェック結果のテキスト")

;; 失敗時
(cons 1 "エラーメッセージ")

;; 使う側
(if (= (car result) 0)
    (cdr result)  ; 成功 → データを取得
  (cdr result))   ; 失敗 → エラーメッセージ
```

`car`は最初の要素（終了コード）、`cdr`は2番目の要素（データ）を取得します。
この命名は歴史的なもので、"Contents of Address Register"と"Contents of Decrement Register"の略だそうです。初見だと謎でした。

### 設定値の定義 - defvar vs defcustom

プロジェクトのディレクトリパスなど、設定値を定義する方法が2つあります。

```elisp
;; defvar: 内部変数
(defvar sakubun-dir "~/work/...")

;; defcustom: ユーザー設定可能
(defcustom sakubun-dir "~/work/..."
  "Directory path..."
  :type 'directory
  :group 'sakubun)
```

`defvar`で機能としては十分ですが、`defcustom`を使うことで、ユーザーが`M-x customize`でGUIから設定変更できるようになります。型情報（`:type 'directory`）も指定できるため、ファイル選択ダイアログが使えるようになるなどの利点があります。

### 標準ライブラリの活用

AIが最初に生成したコードには、カーソル位置の段落を取得する少し複雑な処理がありました。

```elisp
(save-excursion
  (buffer-substring-no-properties
    (progn (backward-paragraph) (point))
    (progn (forward-paragraph) (point))))
```

こんなlow-levelの実装をすることに違和感があったので、組み込み関数がないのか聞いてみると、これはEmacs標準の`thing-at-point`で置き換えられることを知りました。

```elisp
(thing-at-point 'paragraph t)
```

## まとめ

AIに生成させたコードを題材にすることで、教科書を読むよりも実践的にElispを学べました。特に良かった点は下記です。

- 具体的な目的がある: 自分が欲しいツールを作るので、モチベーションが続く
- 疑問をその場で解決できる: わからない部分をAIに質問し、コードを改善していける
- 現代的なベストプラクティスを学べる: lexical-bindingや非同期処理など、最近のElispの書き方がわかる

一方で、AIが生成するコードは車輪の再発明をしていることもあるので、標準ライブラリやドキュメントを自分でも調べることが重要だと感じました。

今後は、このsakubun-checkerをさらに拡張して、どういう間違いをおかしやすいか統計処理をしたり、英語版に対応したりしたいと思っています。
