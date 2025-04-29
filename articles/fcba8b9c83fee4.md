---
title: "Slackに常駐させたAIエージェントがMCP経由で社内情報を取得してくるまで"
emoji: "🐿️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "mcp", "slack", "vercel"]
published: false
publication_name: "moshjp"
---

Slack上にAIアシスタントを置いて社内業務を改善できないかなと模索した時の記録です。

## この記事について

本記事では実装で詰まった部分ややりたいことを実現するために考えた部分の話がほとんどなので実際のコードとかの話はあまりしていません。ただ、実装自体はテンプレートだったりライブラリのREADMEに書いてあることばかりなのでそのあたりは省きます(リンク貼っておきます)。

- 自前のAIエージェントを動かしたい
- エディタ以外からMCPに繋いで色々やってみたい
- 実際に作ったとして何ができそうか知りたい

という方向けの記事になっています。

## モチベーション

ローカルの開発におけるMCPの利用はすでに当たり前のように普及していますが「みんなが社内情報検索できる環境あったら便利なんじゃないの」と思ったのがきっかけでした。

そこでまずは社内のSlackにAIエージェントがいて話しかけたら調べ物してきて回答してくれるというところをひとまずのゴールとします。

## とりあえずやってみる

とにかく今は動いている姿を確認したので最小限の対応だけします。

### Slack上で会話ができるところまで

とにかくサクッと動くものが欲しかったのでVercelのtemplatesにある[ai-sdk-slackbot](https://vercel.com/templates/other/ai-sdk-slackbot)を使ってVercel functionsにデプロイします。Slack Appの設定についてもすべてREADMEに記載してくれているのでenvに自分のトークンを設定するだけで動きます。

テンプレートの中身はSlack関連の処理とVercel AI SDKを利用したLLMとの接続に関する実装がほとんです。フレームワークも使っていないのでコード量も少ないですし、AIエージェント初心者の私にはとても良いサンプルでした。

今回モデルはgpt-4.1-miniにしましたが[価格や用途](https://openai.com/api/pricing/)から適切なものを選ぶのが良いでしょう。対応しているモデルの一覧は[AI SDKのドキュメント](https://sdk.vercel.ai/docs/introduction)から確認できます。

### 不要なものを消す

テンプレートのコード内にはtoolsにgetWetherとsearchWebの2つが設定されています。getWetherは不要なので削除、searchWebはあとで[@modelcontextprotocol/server-brave-search](https://github.com/modelcontextprotocol/servers/tree/main/src/brave-search)に置き換える予定なので消します。

https://github.com/vercel-labs/ai-sdk-slackbot/blob/513f1053919ce2559fd4821090921ad8664dd616/lib/generate-response.ts#L19-L69

## MCPを接続する

ここまでくると自分のSlackワークスペース上で動くgpt-4.1-miniのSlackbotが誕生します。toolsに設定されていたものは全て消している状態なので今はGPTと自社のSlack上でお話しできるね、というだけです。

ここからは今立ち上げたAIエージェントとMCPを接続して社内の情報を閲覧・検索できるようにしていきます。

### MCPサーバーと接続する

基本的には公式で提供されているものだけを使います。また、今回は個別にMCPサーバーを立てるのではなく子プロセスとしてMCPを起動してそこにリクエストを投げる形を取っています。

起動時の注意点としてディレクトリの書き込みに対しての制限などがあるため事前にインストール済みのパッケージを参照する必要があります。つまり、npxなどでその場で取得してきて実行する方式だと上手くいきません。

一応、/tmp以下なら書き込み可能ですが容量の制限があるので今回は使用していません。

https://community.vercel.com/t/how-do-install-dependencies-in-the-tmp-directory/1849/5

```sh
pnpm add @notionhq/notion-mcp-server
```

```ts:notion-mcp.ts
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
