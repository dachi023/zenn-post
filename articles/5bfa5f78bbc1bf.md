---
title: "音声入力、テキスト読み上げを活用したい"
emoji: "🎤"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "superwhisper"]
published: true
publication_name: "moshjp"
---

自身の悩みを解消するために音声を試してみました。

## 悩み

2つあります。

1つ目はデスクに座っている時間が長く、慢性的な肩こりが発生していることです。
ストレッチしたり整体受けたりなどしていますがまた徐々にコリが蓄積されていくのを日々感じながら生きています。

蓄積の要因の中でもタイピングによる巻き肩からのダメージが1番大きそうなので、
キーボードから手を離す時間を増やせばコリの蓄積が緩やかになるはず → 音声入力を使っていこうという発想になりました。

---
2つ目は離席や別の作業をやっている時など、画面を見ていない時のコーディングエージェント(ここではClaude Code)のレスポンスを確認する方法についてです。
以下のポストを参考にHookを設定することで完了した時に音が鳴ってとても便利だったのですが、今度はどういった対応がなされたのかを画面を見ずに把握したい欲が出てきました。

https://x.com/kei31/status/1939987303415558243?s=46

## 音声入力

音声書き起こしソフトのsuperwhisperを使ってみました。

https://superwhisper.com/

superwhisperは録音した音声を書き起こし、アクティブになっているアプリケーションによってその後の振る舞いを変えることができます。
例えば以下のように設定が可能で、アクティブなアプリケーションに応じて設定が勝手に切り替わってくれるのが非常に便利でした。

- Slack: 入力欄に貼り付け
- Zoom: メモの作成

また、書き起こしに使用するAIモデルやプロンプトを個別に設定できるので用途に応じて使うモデルやプロンプトを切り替えるといったことも可能です。

**Shift長押しで自動Enter**

`Hold shift to auto-send after paste` のオプションを有効にした状態でShiftを押しっぱなしにすると録音終了後に自動で送信(Enterキーを押す)してくれます。

![](/images/5bfa5f78bbc1bf/hold-shift.png)

### やってみた感想

書き起こし精度は十分で、きちんと言ったことを入力できていた印象です。
喋っている最中に立ち上がってストレッチをするなどが可能になったので今よりも健康になれそうです。

## テキスト読み上げ

Claude Codeの作業完了時のメッセージを読み上げて欲しいのでClaude Codeにスクリプトを作ってもらい、それをHookに設定しました。

```sh
#!/bin/bash
set -euo pipefail

input=$(cat)
[[ $(echo "$input" | jq -r '.stop_hook_active') == "true" ]] && exit 0

text=$(python3 -c "
import json
with open('$(echo "$input" | jq -r '.transcript_path')') as f:
    for entry in reversed([json.loads(l) for l in f if l.strip()]):
        if (msg := entry.get('message', {})).get('role') == 'assistant':
            if texts := [c['text'] for c in msg.get('content', []) if c.get('type') == 'text']:
                print(' '.join(texts)[:300]); break
" 2>/dev/null || echo "No message")

say "$text" && afplay /System/Library/Sounds/Glass.aiff 2>/dev/null &
```

https://gist.github.com/dachi023/9b0c834925b10355c253e8fb3ce3ee15


### やってみた感想

とりあえず読み上げてもらって追加で指示をしたほうがいいのかどうか、を判断できるようになり案外良かったです。
微妙ポイントとして大量の文章を読み上げられるのが結構キツかったので読み上げる文字数の上限は100文字くらいでも良さそうでした。
