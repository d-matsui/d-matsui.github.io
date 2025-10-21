---
author: Daiki Matsui
pubDatetime: 2025-10-21T22:54:27+09:00
title: dotfilesで開発環境を管理する
slug: managing-dev-environment-with-dotfiles
featured: true
draft: false
tags:
  - linux
  - ai
  - dotfiles
  - essay
description: AI開発ツールで開発環境を消し飛ばしてしまった経験から、dotfilesによる環境管理とバックアップ戦略を見直した。
---

## はじめに

Claude Codeで開発していたときに、想定外のコマンドが実行されて開発環境を失ってしまいました。幸い、開発中のコードの大半はGitHubにあったので致命的ではなかったのですが、環境の復元には結構時間がかかりました。

この経験をきっかけに、開発環境の管理方法やバックアップについて改めて考え直すことになりました。この記事では、対応したことと今後の方針について書きます。

## 何が起きたか

日本語作文チェッカーを作っていました。これは、日本語の文章をAIのAPIに渡して、添削結果とスコアを返してくれるツールです。

APIのレスポンスが遅かったので、Puppeteerでブラウザを直接操作する方式に変更しようとしていました。具体的には、ブラウザ上のAIサービスに文章を入力させて、表示されたHTMLをパースして添削結果を取得する方法です。

PuppeteerからChromeを操作できるようにするために、`--remote-debugging-port` でChromeを起動する際、`--user-dir` オプションの指定で問題があり、`/home/daiki/` に `~/` というディレクトリが作られてしまいました（おそらく変数展開されて、想定通りの挙動にならなかった）。

このディレクトリと、その中に作られた不要なファイルを削除するよう、Claude Codeに依頼したところ、想定していたのとは違うコマンドが提案されました。確認が不十分なまま実行してしまい、結果的に `~/` 全体を削除してしまいました。

`testdisk` で復元を試みましたが、うまくいきませんでした。開発中のコードは基本的にGitHubにあったものの、`~/` にあった設定ファイルが消えてしまいました。その復元がかなり面倒でした。

## dotfilesで環境を管理する

環境の復元を楽にするため、これを機に `~/` 配下の設定ファイルを `dotfiles` としてGitHubで管理することにしました。

### 構成

```
~/dotfiles/
├── .bashrc                      # Bash設定
├── .profile                     # 環境変数
├── .gitconfig
├── emacs.d/                     # Emacs設定
├── .config/
│   ├── fcitx5/                  # 日本語入力
│   └── alacritty/               # ターミナル設定
├── .gitignore
└── install.sh                   # セットアップスクリプト
```

### 復元方法

新しいマシンやクリーンな環境では、以下のコマンドで設定を復元できます。

```bash
git clone git@github.com:d-matsui/dotfiles.git
cd dotfiles
./install.sh
```

これで環境構築が完了します。

`install.sh` では、各設定ファイルのシンボリックリンクを作成しています。

```bash
#!/bin/bash

ln -sf ~/dotfiles/.bashrc ~/.bashrc
ln -sf ~/dotfiles/.gitconfig ~/.gitconfig
ln -sf ~/dotfiles/emacs.d ~/.emacs.d
# ... その他の設定ファイル
```

### 管理のポイント

dotfilesをGitで管理する際のポイントは以下の通りです。

設定ファイルをdotfilesディレクトリにまとめて、`~/` にはシンボリックリンクを配置する形にしました。こうすることで、ツールが設定ファイルに自動追記した内容もまとめてGitで追跡できます。

`.gitignore` では、`emacs.d/` 配下のキャッシュファイルや自動生成ファイルなど、バージョン管理が不要なものを除外します。`.bashrc` や `.profile` でAPI_KEYなどの秘密情報を環境変数にexportしている場合は、そのような設定を別ファイルに分離して `.gitignore` に追加する必要があります。こうすることで、秘密情報はローカルのみに保持され、Gitの履歴にも残らず、誤ってインターネットにアップロードされることもありません。dotfilesは基本的にプライベートリポジトリで管理するのが安全です。

dotfilesを整備する過程で、長年放置していたEmacs設定や、何に使っていたか忘れた古い設定ファイルを見直すことができました。どの設定が本当に必要なのか考えながら整理できて、結果的に環境もすっきりしました。

## 今後の対策

dotfilesで設定ファイルは管理できるようになりましたが、さらに安全性を高めるための対策も検討しています。

Claude Codeには[hooks機能](https://docs.claude.com/en/docs/claude-code/hooks#security-best-practices)があります。ドキュメントによると、特定のファイル（`.env`、`.git/`など）への操作を避けるように設定できるようです。ただ、hooksは任意のシェルコマンドを実行できる機能であるため、自己責任で利用してねという注意書きがあります。

また、開発環境をホストOSから分離する方法も有効そうです。Anthropicの[ベストプラクティス](https://www.anthropic.com/engineering/claude-code-best-practices)でも、DevContainerなどのコンテナ環境での開発が紹介されています。想定外のコマンドが実行されても、ホストシステムへの影響を最小限に抑えられます。

あとは基本的なことですが、定期的なバックアップも重要です。`Timeshift` がシンプルで使いやすそうなので導入を検討しています。システム全体のスナップショットを定期的に作成しておけば、問題が起きても復元できます。設定さえしておけば、それほど面倒なことでもないので、導入しようと思います。

## まとめ

AI開発ツールは強力で便利ですが、想定外のコマンドが実行されるリスクもあります。

今回の経験から、開発環境の管理方法を見直すことができました。dotfilesをGitHubで管理することで環境復元が容易になり、さらにhooksやバックアップツールを組み合わせることで、より安全に開発できる環境を整えられそうです。

同じように「やってしまった!」という方の参考になれば幸いです。

## 参考

- [Claude Code Hooks - Security Best Practices](https://docs.claude.com/en/docs/claude-code/hooks#security-best-practices)
- [Claude Code Best Practices - Anthropic](https://www.anthropic.com/engineering/claude-code-best-practices)
