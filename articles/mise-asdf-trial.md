---
title: "asdf互換のmiseを試してみた"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "asdf", "direnv", "venv"]
published: true
published_at: "2024-04-03 09:00"
---

こちらは[この記事](https://blog.shinonome.io/mise-asdf-trial/)のクロスポストです。

開発で[asdf](https://asdf-vm.com)というツールを使うことになりました。しかしサブコマンドのセットが自分の性に合いませんでした。互換性のある **rtx** というツールがあることを知り、さらにリネームされて [mise](https://mise.jdx.dev) というツールになっていることを知り使ってみることにしました。

# asdfとかmiseって何

プロジェクト毎にツールチェーンのバージョンを切り替えるマネージャです。**個人的な感覚としては**NPMを使ったことがある人は、開発言語・技術の枠組みを越えたNPMかなと思ってもらえればいいです。Python開発者の方は、開発言語・技術の枠組みを越えたvenvと思ってもらえればいいです。

Pythonのバージョン、flutterのバージョン、Node.jsのバージョン、その他もろもろのツール・コマンド・ライブラリのバージョンを `cd` するだけで切り換えることができます。

# インストール

詳しいインストール方法は[公式](https://mise.jdx.dev/getting-started.html)が紹介してくれているので読んでください。簡単には以下の流れになります。

1. バイナリのインストール：当然ですね。
2. シェルの設定：miseは`cd`の度に`$PATH`を書き替えているのでそのために必要です。
3. `usage` のインストール：補完に必要になります。

私は **MacOS** & **fish** なので以下の通りです。

```bash
# バイナリのインストール
brew install mise

# シェルの設定
echo 'mise activate fish | source' >> ~/.config/fish/config.fish

# usage は mise で管理できる
mise use -g usage@latest
```

# 機能

## ツールのローカルバージョン管理

プロジェクトルートで `mise use` を使うだけで、そのディレクトリに入った際にそのツールが用いられるようになります。

| コマンド                 | 説明                                                                    |
| ------------------------ | ----------------------------------------------------------------------- |
| `mise use hoge@0.0.0`    | 現在のディレクトリで特定のツールを有効化する (必要に応じてインストール) |
| `mise install`           | `.mise.toml` など設定ファイルを読みこんで必要なツールをインストール     |
| `mise use hoge@0.0.0 -g` | グローバル版 `use`                                                      |

一旦セットアップされたプロジェクトに `cd` し、ツールが足りない場合きちんと警告してくれます。`mise install` を叩くだけで必要なツール一式がインストールされます。
miseではなくasdfのプロジェクトだった場合も安心です。`.mise.toml` だけではなく `.tool-versions`, 他の設定ファイル(`.node-version`など!)まで読みこんでくれます。

asdfの場合は以下の手順を踏む必要がありました。

1. 一旦グローバルにプラグインをインストール
2. 特定のバージョンの `shims` をインストール
3. `local` サブコマンドでプロジェクトで用いるツールを指定する。

miseの場合は `mise use hoge@0.0.0` だけでいいので便利です。

## タスクランナー

プロジェクトでツールを管理していくと、プロジェクトに依存するタスクも管理したくなりますね。`.mise.toml` にはタスクも書くことができます。

以下は実際に使っているプロジェクトからコピペしてきたものです。

```toml
[tools]
go = "1.22.1"
bun = "1.1.0"

[tasks.runserver]
description = 'Run the API server'
run = 'go run ./api/'
env = {GOMIGEMO_DICTDIR = './migemodict'}

[tasks.entgen]
description = 'Generate ent code'
run = 'go generate ./api/ent'
sources = ['./api/ent/schema/*.go']
outputs = ['./api/ent/*.go']

[tasks.gqlgen]
description = 'Generate GraphQL code'
run = 'go run github.com/99designs/gqlgen'
sources = ['./api/graph/schema/']
outputs = ['./api/graph/generated.go']
dir = './api'
depends = ['entgen']

[tasks.upd]
description = 'Docker compose up -d'
run = 'docker compose up -d'
```

- デフォルトでジョブ数4で並列実行されます。
- `sources` や `outputs` の日付を確認しており必要に応じてタスクをスキップするようですが、この設定だと上手く働いていないようでよく分かりません。
- `sources` にワイルドカードがあり、かつ `dir` が指定されている際にクラッシュしまして、これは[確認中](https://github.com/jdx/mise/issues/1852)です。

## Python venv との連携

毎度 `source ./venv/bin/activate` するのは面倒ですよね。miseで`python`を用いる場合、プロジェクトに移動した際に自動で `venv` に入る設定も一緒に `.mise.toml` に書けます。

```toml
[tools]
python = "3.10"

[env]
_.python.venv = "venv"
```

# 使用感

## スピード

asdfに比べmiseはかなり軽快に感じました。仕組み的には、asdfがかなりのオーバーヘッドがあることに起因しているようです。asdfがコマンドを呼び出す際、内部的には `asdf exec` コマンドを毎度呼び出しているのに対し、miseはダイレクトに`$PATH`を書き替えています。
どの程度の効果があるのか自分には評価できないですが、Rust製であることも高速化に一役かっていると宣伝されています。

## サブコマンド類

また、コマンド類が少なくフレンドリーなのも使いやすいと感じました。asdfはanyenv系のコマンド類を踏襲していそうなのですが、miseは一般のパッケージマネージャのコマンド類に似ており、なおかつ少ないため分かりやすいです。

## コマンドのスペル

個人的な感想ですが、私はQWERTY配列ではなく[Dvorak配列](https://ja.wikipedia.org/wiki/Dvorak配列)を用いているため`asdf`はかなり打ちづらく感じていました。miseは打ちやすくて快適です。

# 最後に

全体的な印象としては、asdfを触っている時にちょくちょく感じていた不便を改善した、痒いところに手が届くような便利さを感じました。バージョン管理システムの決定版になるか？という感覚なのでしばらく常用してみようと思います。
