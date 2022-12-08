---
title: "ミニマリストのためのファジーセレクタ fzyselect.vim"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "vim"
  - "neovim"
published: false
published_at: "2022-12-16 19:00"
publication_name: "vim_jp"
---

この記事は[Vim Advent Calender 2022](https://qiita.com/advent-calendar/2022/vim)の17日目の記事です。

![screenshot.png](https://storage.googleapis.com/zenn-user-upload/424e2fab5bb2-20221209.png)

ミニマリストのためのファジーセレクタプラグイン: [fzyselect.vim](https://github.com/gw31415/fzyselect.vim)を開発しましたので、その紹介をさせていただきます。

# なぜfzyselect.vimなのか
ファジーファインダー・セレクタプラグインはVimプラグインの中でも近年トレンドになっています。[telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)や[ddu.vim](https://github.com/Shougo/ddu.vim)をはじめ、世の中には様々なファジーセレクタプラグインが開発され、機能追加され、公開されています。これらのファジーセレクタはよくできており、デザインや検索パフォーマンスも含めてユーザーを満足させてくれます。触っているだけでワクワクさせてくれるものでもあります。

しかし、それらはデフォルトのソースが多すぎたり、抽象度が高すぎて用いないオプションが溢れているものが多くなっています。
これはある種アタリマエな帰結です。ファジーセレクタは汎用的な存在で、それを導入する目的は「**様々な操作について、統一的に操作できるUIを提供する**」ことでしょう。なので、余計なデフォルトソースや汎用性のあるコードで溢れるのは当然なのです。
しかしながらミニマリストにとって使わないデフォルトソースや用いられないコードは気になってしまうのが性分です。

ミニマリストは、**汎用性と合理性を兼ね備えた必要最小限のインタフェース**で実装されたファジーセレクタを求めているのです。fzyselect.vimを開発したのはこれを実現するためです。

# 特徴
## 1. `vim.ui.select`に対応する関数しか提供しない。
本質として汎用性が必要であるファジーセレクタをミニマリスト向けにするためには、その汎用性に合理性があるものでなくてはなりません。つまり、**プラグイン作者の自作仕様には疑いの余地がある**のです。したがって今回fzyselect.vimを開発するにあたり、仕様について次のことを遵守することにしました。
> `vim.ui.select`のインタフェースに合わせる

`vim.ui.select`はNeovimで標準とされているインタフェースですので、これに遵守することはオレオレインタフェースを自作するよりも合理的です。これは多くのミニマリストを満足させることでしょう。

## 2. デフォルトソースを同梱しない。
ユーザーにとって要らないソースは同梱するべきではありません。このため、デフォルトソースやキーマップは何ひとつとして同梱しておりません。

このため、各ユーザーが設定ファイルでソースから自作することが求められます。
例えば、`git ls-files`でプロジェクト上のファイルをサーチする「ソース」は以下のように設定することができます。
```vim
fu! s:fzyselect_lsfiles() abort
	let out = system(['git', 'ls-files'])
	if v:shell_error
		echo out
	el
		cal fzyselect#start(split(out, '\n'), #{prompt:'git ls-files'}, {p-><SID>edit(p)})
	en
endfunction
fu! s:edit(path) abort
	if a:path != v:null
		exe 'e ' .. a:path
	en
endfunction
nnoremap <c-p> <cmd>cal <SID>fzyselect_lsfiles()<cr>
```
その他需要が高いと思われる設定は[README.md](https://github.com/gw31415/fzyselect.vim/blob/main/README.md#configuration-example)に公開しています。

## 3. `matchfuzzypos`の使用。
ファジーマッチ及びハイライトはVim組みこみの関数:`matchfuzzypos`によって行います。組みこみ関数のため高速で、かつ車輪の再発明となっていないことがポイントです。

確かにファジーマッチのアルゴリズムはそのプラグインの性能を特徴づける側面があります。しかしながら、今回は「あるものは使う」精神で自前実装しないことにしました。

# インストール方法など
ほとんどのユーザーは、自作ソース設定の他に**インストール**と**キーマップ設定**を行う必要があります。

[Plug.vim](https://github.com/junegunn/vim-plug)
```vim
Plug 'gw31415/fzyselect.vim'
" 中略
fu! s:fzy_keymap()
	nmap <buffer> i <Plug>(fzyselect-fzy)
	nmap <buffer> <cr> <Plug>(fzyselect-retu)
	nmap <buffer> <esc> <cmd>clo<cr>
endfu
au FileType fzyselect cal <SID>fzy_keymap()
```

[packer.nvim](https://github.com/wbthomason/packer.nvim)
```lua
use {
    'gw31415/fzyselect.vim',
    config = function()
        vim.api.nvim_create_autocmd('FileType', {
            pattern = 'fzyselect',
            callback = function ()
                vim.keymap.set('n', 'i','<Plug>(fzyselect-fzy)', { buffer = true })
                vim.keymap.set('n', '<cr>','<Plug>(fzyselect-retu)', { buffer = true })
                vim.keymap.set('n', '<esc>','<cmd>clo<cr>', { buffer = true })
            end
        })
    end
}
```

# 結び
巷のファジーファインダーは機能が多すぎるという方は、是非[fzyselect.vim](https://github.com/gw31415/fzyselect.vim)をご利用してみてください。
ご要望やバグ報告があればこのリポジトリにお願いいたします。
