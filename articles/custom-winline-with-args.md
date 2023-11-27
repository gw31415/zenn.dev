---
title: Vimで、位置を指定できる `winline()` を作成する
emoji: 📏
type: tech # tech: 技術記事 / idea: アイデア
topics:
  - vim
published: true
published_at: "2023-12-13 07:00"
publication_name: "vim_jp"
---

:::message
この記事はVim駅伝の2023-12-13向けの記事です。
:::

# きっかけ

最近Vimを触りはじめた友人とのチャットがきっかけでした。
![discord-chat](/images/custom-winline-with-args/discord-chat.png)

Vimでスクロールをすると、ウィンドウの最下行をテキストの終端が通り越して一番上まで移動することができます。逆に言えば、マウスで勢い良くスクロールをするとテキスト欄が一気に見えなくなってしまいます。
VSCodeの場合、これを抑制する設定値 `editor.scrollBeyondLastLine` があります。これは `editor.padding.bottom` の余白を残しスクロールをストップする設定値だそうです。
https://qiita.com/msickpaler/items/134d1dd2c0ef0fdade6b

Vimではこの個別の設定はありませんが、内部的には `<MouseDown>` や `<MouseUp>` のキーを反復するという形で扱われているので上手く関数を作成しリマップすることで再現できるはずです。いざ作成してみたところ、ウィンドウからの相対位置を取得する必要が出てきました。

組込みの関数 `winline()` はカーソル位置からのウィンドウからの相対位置を取得することができますが、これは引数を持たずカーソル限定になっています。本記事では `Winline({expr} [, {winid}])` を作成します。

# 本題

## TL;DR

以下でおおよそ想定どおりの挙動をすると思います。

```vim
function Winline(expr = v:null, nr = winnr()) abort
  " Fallback
  if a:expr == v:null
    return winline()
  endif

  " 引数のパース
  let [l, c] = type(a:expr) == v:t_list ? [line(a:expr[0]), col(a:expr[1])] : [line(a:expr), 0]

  " 計算箇所
  let pos = screenpos(a:nr, l, c).row - screenpos(a:nr, line('w0'), 0).row + 1

  " 範囲外スロー
  if pos < 1 || pos > winheight(a:nr)
    throw "The position is out of range."
  endif

  return pos
endfunction
```

## 問題1: 折り返し

当初はスクロールストップのプラグインを以下のように書いていました。

https://github.com/gw31415/scrollUptoLastLine.vim/blob/86b66b588954416cc87292e6ded6a048fb533d34/plugin/plugin.vim#L2-L9

これを書き換えると以下のようにして相対位置を取得していることになります。
```vim
return line(a:expr) - line('w0') + 1
```
指定した位置とウィンドウ最上行の行数の差をとり1を足すシンプルな実装です。ただし、これだと文字列がWindow幅を越える折り返しに対応できません。

## 問題2: 折り畳み

折り返しの行数を計算し、`'w0'` から順に足し合わせていくことで相対位置を計算するというアイデアで実装したプラグインが以下になります。

https://github.com/gw31415/scrollUptoLastLine.vim/blob/9c02cdea105fa2c6d5af0fbca1985ee133c3b896/plugin/plugin.vim#L3-L23

:::message alert
`s:winline` という関数がありますが、これ自体は `winline` とは関係なく、指定した行の表示行数を計算する関数です。間違えて命名してしまいました。すみません。
:::

例によって相対位置取得部分を抜き出します。

```vim
fu! s:lineheight(expr) abort
  let wi = getwininfo(win_getid())[0]
  let width = wi.width - wi.textoff
  return (strdisplaywidth(getline(a:expr)) - 1) / width + 1
endfu

let pos = 0
for l in range(line('w0'), line(a:expr))
  let pos += s:winline(l)
endfor
return pos
```

`s:lineheight` 関数は指定した行の表示行数を計算する関数です。
- `width` はウィンドウのうち行表示カラムなどを除いた表示箇所の横幅を計算しています。
- `strdisplaywidth` で行の横幅を計算し、`width` で割ることでその行の縦幅を計算しています。

これを`'w0'`からFor文で足し合わせることで計算しています。計算量は多いですが、テキストエディタの縦幅は高々数十行程度なので速度的には問題ない範囲かと思います。

これで(汚いながらも)上手くできたと思いましたが、折り返しのことを失念していたので全部を書き直すことにしました。ボツにはなりましたが、 `s:lineheight` はどこか別の場所で使えるかと思います。

## 解決版

実は指定したウィンドウ、指定した行と列でスクリーン位置を取得する関数 `screenpos()` が存在していました(最初からこれ使えば良かったやん)。これを用いて最初のバージョンを書き換えればよかったということですね。

```vim
return screenpos(winnr(), a:expr, 0).row - screenpos(winnr(), line('w0'), 0).row + 1
```

最終的なプラグインはこちらに載せています。

https://github.com/gw31415/scrollUptoLastLine.vim

# 最後に

最終的には便利な組み込み関数 `screenpos()` が存在していたのでシンプルな結論になりましたが、Vimの表示箇所の扱いなどこれまで触れていなかったところに触れることができたいい機会でした。
