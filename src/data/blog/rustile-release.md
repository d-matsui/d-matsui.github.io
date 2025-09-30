---
author: Daiki Matsui
pubDatetime: 2025-01-30T05:30:00Z
title: "自作ウィンドウマネージャーをリリースしました"
slug: rustile-release
featured: true
draft: false
tags:
  - rust
  - linux
  - window-manager
  - japanese
description: "プライベートでコツコツ、Claudeと一緒に自分専用のウィンドウマネージャーを2ヶ月で作った話"
---

前々から作ってみたかったウィンドウマネージャーを、Claudeと一緒に作りました。Rustで書かれたタイル型ウィンドウマネージャー [Rustile](https://github.com/d-matsui/rustile)です。

![Rustileで複数のウィンドウをタイル配置している様子](/rustile-screenshot.png)

## Rustileの特徴

これまでプライベートで使っていた[yabai](https://github.com/koekeishiya/yabai)や[xpywm](https://github.com/h-ohsaki/xpywm)のように、Rustileもタイル型ウィンドウマネージャーです。主な特徴は以下の通りです。

- **BSP (Binary Space Partitioning) レイアウト** - ウィンドウを自動的にタイル状に配置
- **キーボード操作** - マウスなしで完結する操作性
- **シンプルな設定** - TOMLファイルで直感的に設定できる
- **基本機能の実装** - focus、move、swap、fullscreen、half-screenなど

## 開発の過程

最初のcommitが7/16なので、約3ヶ月で作りました。プライベートの時間でコツコツ進めました。

正直、開発を始めた時点ではX11やウィンドウシステムの低レイヤーな知識がほとんどありませんでした。「ウィンドウをどうやって管理するのか」「キーボードイベントはどう拾うのか」といった基本的なことから、一つずつClaudeと対話しながら学んでいきました。分からないことを質問し、実装してみて、動かなかったらまた質問する。そのサイクルを繰り返すうちに、少しずつ形になっていくのが楽しかったです。

## 自分の環境を自分で作る

ウィンドウマネージャーを自作していて一番感じるのは、自分の作業環境を自分で作れることの楽しさです。使っていて「ここがこうだったらいいのに」と思ったら、すぐに実装できる。そういう自由度は、既存のツールにはない魅力です。

まだまだ普段使いできるレベルには到達していませんが、これからの開発が楽しみです。

興味のある方はぜひ[GitHubリポジトリ](https://github.com/d-matsui/rustile)をチェックしてみてください。
