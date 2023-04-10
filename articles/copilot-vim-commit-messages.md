---
title: "Copilot.vim でコミットメッセージを補完する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim", "copilot"]
published: true
---

## Copilot.vim

みなさん GitHub Copilot は使っていますでしょうか。私は普段 Neovim を使っているので、Vim / Neovim から Copilot を呼び出せる [copilot.vim](https://github.com/github/copilot.vim) というプラグインを使っています。始めはあまり期待していなかったのですが、思っていたよりかなり優秀で、普通に生産性が上がっています。

## コミットメッセージを補完する

コードを補完してくれるだけでも十分ありがたいのですが、VSCode では Copilot がコミットメッセージも考えてくれる、という情報を最近 Twitter で目にして、そういえばうちの Copilot 君はやってくれないなあ、と気付きました。

というわけで、Vim / Neovim でもコミットメッセージを(それなりに)補完できるようにしてみました。

### 1. ファイル形式 gitcommit に対して Copilot を有効にする

とりあえず Google で "copilot.vim commit messages" とかで検索してみると、[この記事](https://codeinthehole.com/tips/vim-and-github-copilot/) がすぐに見つかります。なるほど、デフォルトでは gitcommit 形式に対して Copilot が有効になっていないようです。記事にもあるように、次のように設定してみます。

```vim
let g:copilot_filetypes = #{
  \   gitcommit: v:true,
  \   markdown: v:true,
  \   text: v:true,
  \   ddu-ff-filter: v:false,
  \ }
```

gitcommit 以外にも、markdown などをお好みで追加すると良い感じです。`ddu-ff-filter` は [ddu.vim](https://github.com/Shougo/ddu.vim) というファジーファインダープラグインの入力エリアに対応していて、ここで有効になっているのが邪魔だったので無効にしています。

では試してみましょう。上の設定に yaml を追加してみます：

```diff
@@ -4,6 +4,7 @@
   \   markdown: v:true,
+  \   yaml: v:true,
   \   text: v:true,
```

コミット！

![](/images/copilot/copilot_1.png)

……。

デタラメですね。Copilot.vim が gitcommit をデフォルトで有効にしていない理由がよく分かりました。なんとなく気付いていましたが、Copilot はアクティブなバッファの内容しか読んでいないようです。

### 2. コミットの diff を gitcommit に追記する

これが IDE ならここで諦めるところですが、Vim ならハックが効きます。Copilot がアクティブなバッファしか読んでいないのであれば、必要な情報つまりコミットの差分をそこに書き込んでやれば良さそうです。

というわけで、次のような設定を追加します (ちなみにこのコードはほとんど GPT-4 に書いてもらいました) ：

```vim
function s:append_diff() abort
  " Get the Git repository root directory
  let git_dir = FugitiveGitDir()
  let git_root = fnamemodify(git_dir, ':h')

  " Get the diff of the staged changes relative to the Git repository root
  let diff = system('git -C ' . git_root . ' diff --cached')

  " Add a comment character to each line of the diff
  let comment_diff = join(map(split(diff, '\n'), {idx, line -> '# ' . line}), "\n")

  " Append the diff to the commit message
  call append(line('$'), split(comment_diff, '\n'))
endfunction

autocmd BufReadPost COMMIT_EDITMSG call s:append_diff()
```

詳しい説明は割愛しますが、コミットメッセージを入力するバッファが開いたときに、そのバッファに現在のコミットの差分を追記するように設定しています。私は [Fugitive](https://github.com/tpope/vim-fugitive) を使っているので、`FugitiveGitDir()` で対象となる Git レポジトリのパスを取得していますが、ここは使っているツールによって変わってきます。

ではもう一度試してみましょう。コミット！

![](/images/copilot/copilot_2.png)

バッチリです！ただ、出だし ("Add..") はこちらで書いてあげないとダメみたいです。それでも、だいぶ楽になりそうです。

## まとめ

[Copilot.vim](https://github.com/github/copilot.vim) でコミットメッセージを補完するハックとして、

1. ファイル形式 gitcommit に対して Copilot を有効にする
2. コミットの diff を gitcommit に追記する

がとりあえずは有効そうです。もっと良い方法があれば教えていただけると嬉しいです。
