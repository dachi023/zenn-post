---
title: "Slack上で動くAIエージェントを実装し、社内情報を取得してくるまで"
emoji: "🐿️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "mcp", "slack", "vercel"]
published: true
publication_name: "moshjp"
---

Slack上で動くAIエージェントをハブにして社内業務を改善できないかなを試してみました。

## この記事について

本記事は実装で詰まった部分ややりたいことを実現するために考えた部分の話がほとんどで、実際にどんなコードを書くのかという話はあまりしていません。実装のほとんどはテンプレートで、入れたものもそれぞれのREADMEに書いてあることばかりなのでそのあたりは省きます(リンク貼っておきます)。

- AIエージェントを自分で動かしたい
- MCPサーバーに接続して社内ツールと連携したい

という方向けの記事になっています。

## モデルコンテキストプロトコル（MCP）

いろんな記事で説明されているのでさっくりとですが。

>MCPは、アプリケーションがLLMにコンテキストを提供する方法を標準化するオープンプロトコルです。MCPは、AIアプリケーション向けのUSB-Cポートのようなものと考えてください。USB-Cがデバイスを様々な周辺機器やアクセサリーに接続する標準的な方法を提供するように、MCPはAIモデルを異なるデータソースやツールに接続する標準的な方法を提供します。

https://docs.anthropic.com/ja/docs/agents-and-tools/mcp

## モチベーション

ローカル環境でのMCP利用はすでに当たり前のように普及していますが組織全体で利用しています、というケースをあまり聞いたことがなかったので検証しようと思いました。

このタイミングでは特に具体的な実用イメージとかはなく、社内SlackにAIエージェントがいてお話しができますくらいのゆるいゴール設定をしていました。

## とりあえずやってみる

というわけなので、とりあえず動く状態を目指します。

### Slack上で会話ができるところまで

とにかくサクッと動くものが欲しかったのでVercelのtemplatesにある[ai-sdk-slackbot](https://vercel.com/templates/other/ai-sdk-slackbot)を使ってVercel functionsにデプロイします。Slack AppのPermissionをはじめとした設定方法などはすべてREADMEに記載してくれているのでそれに沿ってAppを作成、その後にenvを設定するだけで動きます。

[テンプレートの中身](https://github.com/vercel-labs/ai-sdk-slackbot)はSlack APIまわりとVercel AI SDKを利用したLLMへの接続に関する実装がほとんどです。フレームワークも使っていないのでコード量も少ないですし、AIエージェント開発初心者の私にはとても良いサンプルでした。

今回モデルはgpt-4.1-miniにしましたが[価格や用途](https://openai.com/api/pricing/)から適切なものを選ぶのが良いでしょう。対応しているモデルの一覧は[AI SDKのドキュメント](https://sdk.vercel.ai/docs/introduction)から確認できます。

### 不要なものを消しておく

テンプレートのコード内にはgetWetherとsearchWebの2つがデフォルトで設定されています。getWetherは不要なので削除、searchWebは欲しい機能ですがあとで[@modelcontextprotocol/server-brave-search](https://github.com/modelcontextprotocol/servers/tree/main/src/brave-search)に置き換えるつもりなので消します。

https://github.com/vercel-labs/ai-sdk-slackbot/blob/513f1053919ce2559fd4821090921ad8664dd616/lib/generate-response.ts#L19-L69

この状態でデプロイをするとSlack上で動くAIチャットボットが誕生します。もともとToolとして設定されていたものは全て消してあるので今できることは素のLLMと会話することだけです。

## 社内リソースへアクセス可能にする

このチャットボットをMCPサーバーへ接続可能な状態にし、社内の情報を検索・詳細を取得できるようにしていきます。

### MCPサーバーと接続する

セキュリティ面でのリスク[^1]があるので公式による提供かつ安全だと判断したものだけ扱います。また、今回は個別にMCPサーバーを立てるのではなく子プロセスとして起動してそこにリクエストを投げる形を取っています。

起動時の注意点としてディレクトリの書き込みに対しての制限などがあるため事前にインストール済みのパッケージを参照する必要があります。つまり、npxなどでその場で取得してきて実行するやり方だとPermissionの問題で上手くいきません。

一応、/tmp以下なら書き込み可能ですが容量の制限があるので今回は使用していません。

https://community.vercel.com/t/how-do-install-dependencies-in-the-tmp-directory/1849/5

### 例: notion-mcp-server

```sh
pnpm add @notionhq/notion-mcp-server
```

```ts:notion-mcp.ts
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { experimental_createMCPClient } from "ai";
import { dirname, join } from "path";

export async function initNotionMCPServer() {
  if (!process.env.NOTION_API_KEY) {
    throw new Error("NOTION_API_KEY is not defined in environment variables");
  }

  const path = dirname(
    require.resolve("@notionhq/notion-mcp-server/package.json"),
  );
  const transport = new StdioClientTransport({
    command: process.execPath,
    args: [join(path, "bin/cli.mjs")],
    env: {
      ...process.env,
      OPENAPI_MCP_HEADERS: JSON.stringify({
        Authorization: `Bearer ${process.env.NOTION_API_KEY}`,
        "Notion-Version": "2022-06-28",
      }),
    },
  });

  return await experimental_createMCPClient({ transport });
}
```

```ts:generate-response.ts
const notionMCPClient = await initNotionMCPServer();
const notionTools = await notionClient.tools();
// 中略
tools: {
  ...notionTools,
},
```

#### Notion検索を試してみる

![](/images/fcba8b9c83fee4/slack.jpg)

割といい感じに回答してくれます(GPTのおかげではある)。

今回の場合は以下のプロンプトを `generateText()` 呼び出し時に渡しました。

```js:generate-text.ts
system: `
  ## 基本情報
  - 会話・回答のルール
    - 質問が明確でない場合はまずは不明点をまとめて質問する

  ## Notionドキュメントの検索
  回答フォーマットは以下で、調査した結果を適切に記載する。
  また、引用元についてはURLを必ず記載する。
  \`\`\`
  【質問】
  【回答】
  【引用元】
  \`\`\`
`
```

https://sdk.vercel.ai/docs/reference/ai-sdk-core/generate-text#generatetext

### 他MCPサーバーとの接続

ここまでの流れが1回できたらあとは流用して接続したいMCPサーバーの設定と利用の目的を設定するだけなので詳細は省きますが、[sentry-mcp](https://github.com/getsentry/sentry-mcp)でエラー情報を取得・分析して[brave-search](https://github.com/modelcontextprotocol/servers/tree/main/src/brave-search)で類似のエラーについて検索するというフローを作ってみたのですが結構良さそうです。

```js:generate-text.ts
system: `
  ## Sentryのエラー調査
  回答フォーマットは以下で、調査した結果を適切に記載する。
  また、引用元についてはURLを必ず記載する。
  \`\`\`
  【概要】
  - XXX
  【分析】
  - XXX
  【緊急度】
  - XXX
  【Web検索で見つけた類似のエラー】
  - XXX
  【修正方法】
  - XXX
  【引用元】
  - XXX
  \`\`\`

  1. エラー内容をSentryから取得し確認する
  2. 情報を整理し以下を明確にした上で回答に記載する
    - いつからエラーが発生・頻発しているのか
    - エラーが発生している原因・影響範囲
    - どの程度の緊急度で対応しなければいけないか
      - 発生数やユーザー数などから推測する
  3. Webで類似のエラーを検索し、情報を収集する
  4. SentryやWebから収集した情報をもとに修正方法の提案をする
  \`\`\`
`
```

## まとめ

どんな質問にも質の高い回答をさせるのは難しいと感じたので特定のワークフローで価値が出ることを目指してみました。身近な課題を潰していくのがイメージ湧きやすくておすすめです。実装難易度は思っているより全然低かったのでぜひチャレンジしてみてください。

[^1]: https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks
