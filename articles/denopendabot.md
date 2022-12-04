---
title: "Denopendabot で依存関係の更新を自動化する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Deno"]
published: true
---

:::message
この記事は [Deno Advent Calendar 2022](https://qiita.com/advent-calendar/2022/deno) 3日目の記事です (遅れてすみません…)
:::

# 背景

Deno では `import` ステートメントに URL を記述することで外部モジュールを直接読み込むことができます:

```typescript
import $ from "https://deno.land/x/dax@0.16.0/mod.ts";
```

このおかげで Node における `node_modules` や `package-lock.json` にまつわる煩わしさと付き合わずにすみますが、利用しているモジュールを更新するのはやや面倒です。 Node には `npm update` がありますが、これに相当する機能が Deno のランタイムには備わっていません。

そこでよく利用されているのがサードパーティツールの [**udd**](https://github.com/hayd/deno-udd) です。 udd はソースコード内の URL に含まれる SemVer を上げることに特化したシンプルな CLI ツールですが、活発に開発されており、完成度が高いです。

例えば、次のコマンドで `deps.ts` に含まれる外部モジュールの URL を一括でアップデートすることができます。

```bash
$ udd deps.ts
```

あまり知られていませんが `import_map.json` にも対応しています:

```bash
$ udd import_map.json
```

便利ですね。

ただ、GitHub には [Dependabot](https://docs.github.com/ja/code-security/dependabot) という機能があり、Node などのプロジェクトではそもそもこういった作業を手動で行なっていない方も多いと思います。しかし、Dependabot は今のところ Deno に対応しておらず、 [近い将来に対応してくれそうな雰囲気もありません](https://github.com/dependabot/dependabot-core/issues/2417)。したがって、依存関係の更新を自動化するには一手間必要です。

幸い、udd は [GitHub Action のサンプルファイル](https://github.com/hayd/deno-udd/blob/master/.github/workflows/udd.yml) を用意してくれているので、これを弄って使えばある程度 Dependabot に近いことが出来ますが、手軽さや機能の面で劣ってしまいます。

Dependabot のコア部分は[オープンソース](https://github.com/dependabot/dependabot-core)だけど、外部からのコントリビューションを想定している感じではないし、Deno 対応のプルリクを頑張って作るのも違う気がする…

# Denopendabot

じゃあ Deno 用の Dependabot を Deno で作ろう！と思って作り始めたのが **"Deno"pendabot** です:

![Denopendabot のイメージキャラクター](/images/denopendabot/logo.png)

https://github.com/hasundue/denopendabot

コア部分では udd を使っていて、主に GitHub とのインタラクションの部分を実装しています。まだ開発途中ですが、筆者は大きな不満無く実用できています。 Denopendabot は次の4つの形式でリリースされていて、好きな方法で使うことができます:

- GitHub App
- GitHub Action
- CLI ツール
- Deno モジュール

多くの場合 GitHub App を使うのが最も簡単かつ便利なので、今回はそれをメインに解説します。

## インストール

[Denopendabot の GitHub App のページ](https://github.com/apps/denopendabot) からインストールできます。それなりに強い権限を要求するので、Deno を使っているレポジトリのみにインストールすることをオススメします。

レポジトリに `deno.json` か `deno.jsonc` が存在していると、Denopendabot はそのレポジトリを Deno のプロジェクトだと判断して、セットアップ用のプルリクを投げてきます：

![セットアップ用のプルリクエスト](/images/denopendabot/pr.png)

中身は GitHub Action の workflow ファイルで、スケジュールに従って Deno Deploy で動いている Denopendabot 本体にジョブを送信します。またこの workflow は設定ファイルも兼ねています。必要に応じて編集してからマージすればセットアップは完了です。

レポジトリに `deno.json` や `deno.jsonc` が無い場合、この [workflow ファイル](https://github.com/hasundue/denopendabot/blob/main/app/denopendabot.yml) を自分で作成する必要があります。Deno プロジェクトの判定はもう少し色々なケースに対応できるようにしたいと思っています。

## 実行

デフォルトの workflow ファイルには `on: workflow_dispatch` が記述されているので、手動で実行することもできます。試しに実行してみましょう。

![実行](/images/denopendabot/run.png)

この workflow は repository dispatch を発行するだけのもので、実際の処理は Deno Deploy 上で動いている Denopendabot 本体が行います。

うまくいくと、下のようなプルリクが送られてくるはずです。

![結果](/images/denopendabot/result.png)

本文が無くてずいぶんとぶっきらぼうですね。近いうちにちゃんと本文を書かせるようにします。 各モジュールの更新は個別のコミットに分けられて、それらがひとつのプルリクにまとめられます。ここは本家と挙動が違うところですが、将来的にはオプションで制御できるようにしたいです。

デフォルトでは auto merge が有効になっているので、CI が通っている場合 Denopendabot がこのプルリクを自動でマージします。これを無効にしたい場合は `denopendabot.yml` の `auto-merge` の行をコメントアウトしてください。`auto-merge` の値は今のところ `any` だけですが、将来的には `patch` や `non-breaking` など自動でマージする条件を指定できるようにするつもりです。

各コミットの中身は以下のような感じです：

![コミット](/images/denopendabot/commit.png)

# まとめ

というわけで手前味噌ですが Denopendabot の紹介でした。まだ重要なプロジェクトで使用するのはオススメしませんが、ちょっとしたプロジェクトで試しに使ってみていただいて、Issue や PR もらえると嬉しいです。

よく知らない開発者が運用してる App にレポジトリの内容を送るのはちょっと…という方は、GitHub Action 版をお使いください。自分の GitHub Action 環境の中で Denopendabot を実行できます。

あと Deno Deploy で動く GitHub App というのもまだあまり多くないと思うので、そのあたりの技術的な話も近いうちに記事にできればと思っています。
