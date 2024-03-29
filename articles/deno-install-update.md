---
title: "deno install したスクリプトを一括アップデートする"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno"]
published: true
---

:::message
この記事は [Deno Advent Calendar 2022](https://qiita.com/advent-calendar/2022/deno) 6日目の記事です
:::

Deno では `deno install` コマンドによってリモートのスクリプトを実行可能な形式でローカルにインストールすることができます：

https://deno.land/manual@v1.28.3/tools/script_installer

これはとても便利なのですが、インストールしたスクリプトを CLI からアップデートするのはやや面倒で、`--force` オプションまたは `-f` フラグを付けて再度同じスクリプトを `deno install` する必要があります。

実はもっと簡単な方法があります。`deno install` で作成される executable の実態は以下のようなシェルスクリプトです (deno-ja の Slack で教えてくれた @kt3k さんありがとうございます)：

```typescript
#!/bin/sh
# generated by deno install
exec deno run --allow-read --allow-write --allow-net --allow-env 'https://deno.land/x/nublar@0.2.0/nublar.ts' "$@"
```

インストール先は次のように決まります (上述の公式ドキュメントより引用)：

> The installation root is determined, in order of precedence:
> 
> - --root option
> - DENO_INSTALL_ROOT environment variable
> - $HOME/.deno

つまりこれらのシェルスクリプトに書かれているバージョン番号を書きかえてあげれば良いということになります。

というわけで例によって手前味噌ですが、この作業を自動化するツールを作りました：

https://github.com/hasundue/nublar

`deno install` でインストールして…

```bash
deno install --allow-read --allow-write --allow-env --allow-net \
https://deno.land/x/nublar@0.2.0/nublar.ts
```

こんな感じで使えます：

```bash
# list all scripts installed in your environment
$ nublar list
nublar 0.2.0
udd    0.5.0

# check updates for them
$ nublar update --check
Found udd 0.5.0 => 0.7.5

# update all outdated scripts
$ nublar update
Updated udd 0.5.0 => 0.7.5

# or you may specify scripts to be updated
$ nublar update udd
```

内部的には [Denopendabot](https://zenn.dev/hasundue/articles/denopendabot) と同じく [udd](https://github.com/hayd/deno-udd) を使っています。

よろしければお試し下さい。
