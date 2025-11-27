---
author: Daiki Matsui
pubDatetime: 2025-11-20T13:16:00+09:00
title: jcorrect のインストール手順
slug: install-jcorrect
featured: true
draft: false
tags:
  - japanese
  - tools
  - emacs
  - writing
description: CaboCha ベースの日本語文章校正ツールである jcorrect のインストール手順
---

## はじめに

わかりやすい日本語の文章を自分でも書けるようになりたいと思って、h-ohsaki 氏の日本語文章校正ツール jcorrect を試してみました。

jcorrect は CaboCha による係り受け解析を利用して、日本語文章の文法的な問題を検出するツールです。依存ライブラリをソースからビルドしたり、UTF-8 対応のために修正が必要だったりしたので、その手順をまとめます。

参考: [jcorrect を利用した技術文章校正のヒント (草稿)](http://www.lsnl.jp/~ohsaki/research/tips-jcorrect/)

## 依存ライブラリ

jcorrect は CaboCha を使用するため、まず CaboCha とその依存ライブラリをインストールする必要があります。

[CaboCha: Yet Another Japanese Dependency Structure Analyzer](https://taku910.github.io/cabocha/)

CaboCha の依存関係

- CRF++ (0.55 以降)
- MeCab (0.993 以降)
- mecab-ipadic, mecab-jumandic, unidic のいずれか

### CRF++ のインストール

[CRF++: Yet Another CRF toolkit](https://taku910.github.io/crfpp/)

Google Drive から latest である 0.58 をダウンロードします。
https://drive.google.com/drive/folders/1S8FrgZVQQaZWWp_Gw84v919eFDdyw3Yx

ダウンロード後、解凍してビルドします。

```bash
tar xzf CRF++-0.58.tar.gz
cd CRF++-0.58
./configure
make
sudo make install
```

### MeCab & mecab-ipadic のインストール

[MeCab: Yet Another Part-of-Speech and Morphological Analyzer](https://taku910.github.io/mecab/)

ダウンロード後、解凍してビルドします。

MeCab 本体

```bash
tar xzf mecab-0.996.tar.gz
cd mecab-0.996
./configure
make
sudo make install
```

mecab-ipadic 辞書

```bash
tar xzf mecab-ipadic-2.7.0-20070801.tar.gz
cd mecab-ipadic-2.7.0-20070801
./configure --with-charset=utf8
sudo ldconfig
make
sudo make install
```

動作確認

```bash
$ echo "すもももももももものうち" | mecab
すもも	名詞,一般,*,*,*,*,すもも,スモモ,スモモ
も	助詞,係助詞,*,*,*,*,も,モ,モ
もも	名詞,一般,*,*,*,*,もも,モモ,モモ
も	助詞,係助詞,*,*,*,*,も,モ,モ
もも	名詞,一般,*,*,*,*,もも,モモ,モモ
の	助詞,連体化,*,*,*,*,の,ノ,ノ
うち	名詞,非自立,副詞可能,*,*,*,うち,ウチ,ウチ
EOS
```

### CaboCha のインストール

[CaboCha: Yet Another Japanese Dependency Structure Analyzer](https://taku910.github.io/cabocha/)

ダウンロード後、解凍してビルドします。

```bash
tar xjf cabocha-0.69.tar.bz2
cd cabocha-0.69
./configure --with-charset=UTF8 --enable-utf8-only
sudo ldconfig
make
sudo make install
```

## jcorrect のインストール

jcorrect 本体と Emacs 用の設定ファイルをダウンロードします。

```bash
wget http://www.lsnl.jp/~ohsaki/software/perl/jcorrect
wget http://www.lsnl.jp/~ohsaki/software/elisp/jcorrect.el
chmod +x jcorrect
```

`jcorrect` は `/usr/local/bin` や `~/bin` など PATH の通った場所に配置します。`jcorrect.el` は `~/.emacs.d/elisp` などに配置し、`load-path` に追加します。

UTF-8 で入出力するよう jcorrect を修正します。

```diff
@@ -21,6 +21,8 @@
 use Getopt::Std;
 use IPC::Open2;
 use strict;
+use utf8;
+use open qw(:std :utf8);

 my $MAX_PHRASE_LEN   = 60;
 my $MAX_SENTENCE_LEN = 180;
@@ -199,6 +201,8 @@ sub check_kakari {

     # open and write sentence to cabocha
     my $pid = open2(*IN, *OUT, 'cabocha -f1');
+    binmode(IN, ':utf8');
+    binmode(OUT, ':utf8');
     print OUT $str;
     close(OUT);

@@ -239,6 +243,8 @@ sub dump_kakari {
     my $str = shift;

     my $pid = open2(*IN, *OUT, 'cabocha');
+    binmode(IN, ':utf8');
+    binmode(OUT, ':utf8');
     print OUT $str;
     close(OUT);
```

## Emacs 設定

`jcorrect.el` を load-path の通った場所に配置し、以下の設定を追加します。

```elisp
(autoload 'jcorrect-at-point "jcorrect" "Run jcorrect on paragraph at point" t)
(autoload 'jcorrect-region "jcorrect" "Run jcorrect on region" t)
(bind-key "C-c j" 'jcorrect-at-point)
```

`C-c j` でカーソル位置の段落を校正できます。

## 使用例

実際に jcorrect で文章を校正してみます。以下のような文書を入力します。

```
Window Manager が、
ウィンドウ操作に関するイベントを受け取るためには、
root window に対して SubstructureRedirect と SubstructureNotify のイベントマスクを設定する必要があります。
```

jcorrect の出力

```
Window Manager が、
ウィンドウ操作に関するイベントを受け取るためには、
root window に対して SubstructureRedirect と SubstructureNotify のイベントマスクを設定する必要があります。

     WindowManagerが、---------------------D
  ウィンドウ操作に関する-D                 |
                イベントを-D               |
                    受け取る-D             |
                    ためには、-------------D
              rootwindowに対して-------D   |
            SubstructureRedirectと-D   |   |
                SubstructureNotifyの-D |   |
                      イベントマスクを-D   |
                                設定する-D |
                                    必要が-D
                                  あります。

1: **** too long phrase (should be <= 60 chars)
1: check meaning of `WindowManagerが、|ためには、|必要が -> あります。'
1: check meaning of `設定する -> 必要が'
1: check meaning of `rootwindowに対して|イベントマスクを -> 設定する'
1: check meaning of `SubstructureNotifyの -> イベントマスクを'
1: check meaning of `SubstructureRedirectと -> SubstructureNotifyの'
1: check meaning of `受け取る -> ためには、'
1: check meaning of `イベントを -> 受け取る'
1: check meaning of `ウィンドウ操作に関する -> イベントを'
```

係り受け関係が図で表示され、以下のような問題が指摘されます

- `too long phrase` - フレーズが長すぎる (60文字以下推奨)
- `check meaning of ...` - 係り受けの意味の確認

すべてが必ずしも修正すべき問題ではありませんが、文章を見直すきっかけになります。
