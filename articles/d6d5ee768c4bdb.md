---
title: "Storybookのカタログ設計"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["storybook"]
published: false
publication_name: "moshjp"
---

## はじめに

MOSHで実践している **Storyを「カタログ」と「Playground」に分離する設計思想** を紹介します。

### カタログStory

- 目的: コンポーネントの全バリエーションを一覧で確認する
- 特徴: `autodocs` の対象、自動生成されるドキュメントに表示される
- Figma: 設定する (デザインとの差分確認のため)

```typescript
export const Variant: Story = {
  parameters: {
    design: {
      type: "figma",
      url: "https://www.figma.com/design/...",
    },
  },
  render: () => (
    <div className="flex gap-2">
      <Button variant="default">デフォルト</Button>
      <Button variant="destructive">破壊的</Button>
      <Button variant="outline">アウトライン</Button>
      <Button variant="secondary">セカンダリー</Button>
      <Button variant="ghost">ゴースト</Button>
      <Button variant="link">リンク</Button>
    </div>
  ),
};
```

### Playground Story

- 目的: 開発者が自由にコンポーネントを試す
- 特徴: `tags: ["!autodocs"]` でドキュメントから除外
- Figma: 設定しない (編集可能なため差分比較は不要)

```typescript
export const Playground: Story = {
  tags: ["!autodocs"],
  args: {
    children: "ボタン",
    variant: "default",
    size: "default",
    disabled: false,
  },
};
```

## Storyファイルの工夫

### 1. バリエーション表示の一元化

```typescript
export const Default: Story = { args: { variant: "default" } };
export const Destructive: Story = { args: { variant: "destructive" } };
export const Outline: Story = { args: { variant: "outline" } };
```

上記のように定義すると「全バリエーションを一覧できない」「Story数が増える」ため、以下のように定義しています。

```typescript
export const Variant: Story = {
  parameters: {
    // ...
  },
  render: () => (
    <div className="flex gap-2">
      <Button variant="default">デフォルト</Button>
      <Button variant="destructive">破壊的</Button>
      <Button variant="outline">アウトライン</Button>
      <Button variant="secondary">セカンダリー</Button>
      <Button variant="ghost">ゴースト</Button>
      <Button variant="link">リンク</Button>
    </div>
  ),
};
```

### 2. Figma連携を活用

`@storybook/addon-designs` を使ってFigmaデザインとカタログ用のStoryを連することで差分チェックをしやすくしています。
Playground Storyは編集可能なStoryとして定義しており比較する用途には向かないため、設定していません。

```typescript
// カタログStory - Figma連携を設定
export const Variant: Story = {
  parameters: {
    design: {
      type: "figma",
      url: "https://www.figma.com/design/...",
    },
  },
  render: () => (
    // ...
  ),
};
```

### 3. 最小限のレイアウト指定

コンポーネント本来の挙動を確認しやすいよう、基本的にはStory内で独自のpadding/margin/width/heightを設定しないようにしています。
コンポーネントによってはある程度の高さが必要だったりして個別に設定したりしますが、あくまで例外として対応しています。

## さいごに

上記の内容をCoding Agentのガイドラインとして記述、[Storybook MCP](https://storybook.js.org/addons/@storybook/addon-mcp)の設定を追加することでAIによるStory実装の精度が高くなるのでオススメです。
