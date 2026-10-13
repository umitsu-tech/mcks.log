---
title: "【Strands Decider】LM Studio と組み合わせてエージェントの判断までローカルで完結させてみる"
description: "2026年10月1日に、Strands Agents の周辺プロジェクトとして Strands Decider が公開されました。"
pubDate: 2026-10-07
tags: ['StrandsAgents', 'LMStudio', 'ローカルLLM', 'StrandsDecider', 'AIエージェント']
qiitaId: d21ed0b07bd1dc8a5622
importedDate: 2026-10-13
qiitaStats:
  views: 544
  likes: 2
  stocks: 2
  fetchedAt: 2026-10-13
---

## はじめに

2026年10月1日に、Strands Agents の周辺プロジェクトとして Strands Decider が公開されました。

https://github.com/strands-labs/strands-decider

エージェントの中の判断を、小さなモデルに任せるためのものです。普段ローカル LLM は LM Studio で動かしているので、LM Studio と組み合わせて試してみました。

この記事では、実際に動かしてみる手順と、動かしてみてわかったことをまとめます。

## Strands Decider とは

README では次のように説明されています。

> Unlike an LLM, which can generate arbitrary text, a decision model picks between sets of options and rates things on a scale.

訳すと、「任意の文章を生成する LLM と違い、決定モデルは選択肢から選んだり、尺度で評価したりする」とのことです。

中身は Qwen3.5-2B をもとにしたモデルで、文章を生成する部分を、決定を返すための専用の部分に付け替えてあります。答えられるのは次の3種類です。

| 質問の種類 | 返ってくるもの |
|---|---|
| choice | 選択肢ごとの確率 |
| noul | yes の確率 |
| score | 段階ごとの確率 |

以前の記事で紹介した TypeSafe AI の Jev と同じ、文章を返さずに判断だけを返すモデルの仲間です。質問の種類もエンドポイント名（`/v1/systemone`）も Jev とそろっていて、精度は Jev のベンチマークである JevBench で測られています。Jev は API での提供ですが、Strands Decider はオープンソースなので手元で動かせます。

https://qiita.com/ryu-ki/items/e3fe99b5f6704d08a19b

LM Studio で使ってみようと思ったのですが、文章を生成しないので LM Studio の OpenAI 互換サーバーでは扱えません。`strands-decider serve` で専用の HTTP サーバーとして起動し、`/v1/systemone` というエンドポイントに質問を送って使います。

## 前提環境

| 項目 | 内容 |
|---|---|
| マシン | MacBook Air M4 32GB |
| macOS | 26.2 |
| LM Studio | 0.4.25 |
| LLM | gpt-oss-20b（MLX） |
| Python | 3.13.11 |
| uv | 0.12.13 |
| strands-agents | 1.58.1 |
| strands-decider | 0.0.1.dev11（GitHub main 3e94e9d） |
| Decider のモデル | strands-decider-2B-hobson-v21 |

## 今回やりたいこと

Strands Decider のリポジトリには、Strands Agents と組み合わせた公式サンプル（`examples/strands/tool_call_intervention.py`）が入っています。

https://github.com/strands-labs/strands-decider/tree/main/examples/strands

エージェントがツールを呼ぶ直前に、会話とツール呼び出しの内容を Strands Decider に渡し、次の2つを yes/no で聞くサンプルです。

- ツールの引数は、ユーザーの発言に根拠があるか（args_grounded）
- ユーザーに確認する前にツールを呼ぶのは早すぎないか（premature）

怪しいと判定されたら、ツールを実行せずにモデルへ差し戻して、ユーザーに聞き返させます。この差し込みには、Strands Agents の Interventions という仕組みを使っています。Claude Code の Hooks のように、ツールを呼ぶ前に処理を差し込める仕組みです。

https://strandsagents.com/docs/user-guide/sdk/agents/interventions/

公式サンプルでは会話役の LLM が Amazon Bedrock の Claude Haiku 4.5 ですが、コメントに「`model=` はどのモデルプロバイダーに差し替えてもよい」と書かれています。ここを LM Studio の gpt-oss-20b に差し替えれば、LLM も Strands Decider もすべて Mac の中で動くという構図です。

図にすると以下のような感じです。

![今回の構成図](https://images.ryu-ki-learn.com/strands-decider-lmstudio/architecture.png)

では、早速やってみます。

## 手順

### 1. LM Studio の準備

LM Studio のサーバーを起動し、gpt-oss-20b をロードします。

https://lmstudio.ai/docs/cli

```bash
lms server start
lms load openai/gpt-oss-20b
```

### 2. パッケージのインストール

Strands Agents は Interventions が入っている新しいバージョンにします。手元にあった 1.52.0 には Interventions が無く、1.58.1 では使えました。

Strands Decider は Apple Silicon 向けの MLX で動かせますが、2026年10月7日時点の PyPI の版（0.1.0）にはまだ MLX 用の extra がありません。README でも、次のリリースまではリポジトリから入れるよう案内されています。今回は uv で GitHub の main を直接指定しました。

```bash
uv add "strands-agents[openai]>=1.58.1"
uv add "strands-decider[mlx] @ git+https://github.com/strands-labs/strands-decider"
```

https://docs.astral.sh/uv/reference/cli/#uv-add

PyTorch や transformers も一緒に入るので、インストールには10分ほどかかりました。

### 3. Strands Decider のサーバーを起動

別のターミナルで Strands Decider のサーバーを起動します。`--device mlx` で MLX を使います。初回は Hugging Face からモデルの重みをダウンロードします。

https://huggingface.co/StrandsAgents/strands-decider-2B-hobson-v21

```bash
uv run strands-decider serve StrandsAgents/strands-decider-2B-hobson-v21 \
  --device mlx --port 8099
```

起動できたかは `/health` で確かめられます。

```bash
curl -s http://127.0.0.1:8099/health
```

```json
{"status":"ok","model":"strands-decider-2B-hobson-v21","checkpoint":"StrandsAgents/strands-decider-2B-hobson-v21","base_model":"Qwen/Qwen3.5-2B-Base","num_slots":24,"max_length":4096,"temperature":0.935220152138837,"device":"mlx","prefix_cache":true,"vision":false}
```

### 4. 公式サンプルの LLM を LM Studio に差し替える

公式サンプルの `tool_call_intervention.py` を `decider_lmstudio.py` として、同じフォルダにある HTTP クライアントの `_client.py` を `decider_client.py` として手元に写し、`Agent` に渡すモデルを書き換えます。LM Studio は OpenAI 互換のサーバーなので、Strands Agents の `OpenAIModel` で `base_url` を向けます。

https://strandsagents.com/docs/user-guide/sdk/model-providers/openai/

```diff_python
+from strands.models.openai import OpenAIModel
-from _client import NAME, Decider, ServerUnavailable
+from decider_client import NAME, Decider, ServerUnavailable

+model = OpenAIModel(
+    client_args={
+        "api_key": "lm-studio",  # 認証なしなのでダミーでよい
+        "base_url": "http://localhost:1234/v1",
+    },
+    model_id="openai/gpt-oss-20b",
+    params={"max_tokens": 1000},
+)
+
 agent = Agent(
-    model="us.anthropic.claude-haiku-4-5-20251001-v1:0",
+    model=model,
     tools=[get_weather],
     interventions=[gate],
     system_prompt=SYSTEM_PROMPT,
     callback_handler=None,
 )
```

このほかに、判定の結果を見やすく出すようにし、判定を挟まずに動かす `--no-gate` オプションと、質問を引数で渡せるようにする変更を足しました。どちらも公式サンプルには無い、今回の比較用のものです。閾値と差し戻しの上限も変えていますが、こちらは後で理由を説明します。

最終的なコードは次のとおりです。

<details>
<summary>decider_lmstudio.py の全文（クリックで展開）</summary>

```python
"""Strands Decider × LM Studio の検証。

公式サンプル（strands-decider の examples/strands/tool_call_intervention.py、Apache-2.0）の
LLM を LM Studio の gpt-oss-20b に差し替え、閾値と差し戻しの上限を変えたもの。

  uv run decider_lmstudio.py                  # 判定を挟む
  uv run decider_lmstudio.py --no-gate        # 判定を挟まない
  uv run decider_lmstudio.py "東京の天気は？"  # 質問を変える
"""

from __future__ import annotations

import json
import sys

from strands import Agent, tool
from strands.interventions import Guide, InterventionHandler, Proceed
from strands.models.openai import OpenAIModel

from decider_client import NAME, Decider, ServerUnavailable

# Decider に聞く yes/no（公式サンプルのまま）
QUESTIONS = {
    "args_grounded": Decider.noul(
        "Are the tool's argument values grounded in facts the user actually provided?",
        {
            "true": "every argument value traces back to something the user said",
            "false": "an argument value was guessed or invented, not stated by the user",
        },
    ),
    "premature": Decider.noul(
        "Is it premature to call this tool now, before clarifying with the user?",
        {
            "true": "the assistant should ask a clarifying question before calling the tool",
            "false": "there is nothing left to clarify; calling now is appropriate",
        },
    ),
}

# yes とみなす確率。公式サンプルは両方 0.45 だが、根拠ありでも差し戻されたので args_grounded は 0.30 に下げた
GROUNDED_YES = 0.30
PREMATURE_YES = 0.45

# 差し戻しの上限（公式サンプルには無い。無いと差し戻しと呼び直しが終わらないことがある）
MAX_GUIDES = 2


class ToolCallReviewer(InterventionHandler):
    """ツールを実行する前に、Decider の判定で進めるか差し戻すかを決める。"""

    name = f"{NAME}-tool-call-reviewer"

    def __init__(self, decider: Decider) -> None:
        self._decider = decider
        self.log: list[dict[str, object]] = []

    @staticmethod
    def _render(messages: list[dict]) -> str:
        """会話のテキストだけを平文にする。"""
        lines: list[str] = []
        for message in messages:
            role = message.get("role", "unknown")
            for block in message.get("content", []):
                if isinstance(block, dict) and "text" in block:
                    lines.append(f"{role}: {block['text']}")
        return "\n".join(lines) if lines else "(no conversation yet)"

    def before_tool_call(self, event, **kwargs):
        name = event.tool_use["name"]
        arguments = event.tool_use.get("input") or {}
        state = (
            "A conversation between a user and an AI assistant is below, followed by a "
            "tool call the assistant now wants to make.\n\n"
            f"--- CONVERSATION ---\n{self._render(event.agent.messages)}\n\n"
            f"--- PROPOSED TOOL CALL ---\ntool: {name}\n"
            f"arguments: {json.dumps(arguments)}"
        )

        answers = self._decider.ask(state, QUESTIONS)
        args_grounded = answers["args_grounded"]["noul"]
        premature = answers["premature"]["noul"]

        print(f"[decider] {name}({json.dumps(arguments, ensure_ascii=False)})"
              f"  {self._decider.last_latency_ms:.0f} ms")
        print(f"    args_grounded : {args_grounded:.2f}")
        print(f"    premature     : {premature:.2f}")

        guided_so_far = sum(1 for row in self.log if row["verdict"] == "guided")
        if guided_so_far >= MAX_GUIDES:
            print(f"    差し戻しが {MAX_GUIDES} 回に達したので、これ以上は止めずに通す")
            verdict, action = "proceeded", Proceed()
        elif args_grounded < GROUNDED_YES:
            verdict, action = "guided", Guide(
                feedback=(
                    "The arguments are not grounded in anything the user said -- they look "
                    "guessed. Ask the user to confirm the values instead of inventing them."
                )
            )
        elif premature >= PREMATURE_YES:
            verdict, action = "guided", Guide(
                feedback=(
                    "It is too early to call this tool. Clarify with the user first, then "
                    "try again."
                )
            )
        else:
            verdict, action = "proceeded", Proceed()

        self.log.append({"tool": name, "verdict": verdict})
        print(f"    -> {verdict}\n")
        return action


@tool
def get_weather(location: str) -> str:
    """Get the current weather for a specific city.

    Args:
        location: The city to look up, e.g. "Seattle" or "Paris".
    """
    print(f"[tool] get_weather(location={location!r}) を実行した")
    return f"It's 21C and sunny in {location}."


# わざと都市を決めつけさせるプロンプト（公式サンプルのまま）
SYSTEM_PROMPT = (
    "You are an eager weather assistant. Always answer weather questions by calling the "
    "get_weather tool immediately. Never ask the user for clarification -- if no city is "
    "given, just assume Seattle and call the tool with location='Seattle'."
)

# わざと都市を書かない
USER_REQUEST = "What's the weather?"

# Bedrock の代わりに LM Studio（OpenAI 互換）につなぐ
model = OpenAIModel(
    client_args={
        "api_key": "lm-studio",  # 認証なしなのでダミーでよい
        "base_url": "http://localhost:1234/v1",
    },
    model_id="openai/gpt-oss-20b",
    params={"max_tokens": 1000},
)


def main() -> int:
    args = sys.argv[1:]
    use_gate = "--no-gate" not in args
    request = next((a for a in args if not a.startswith("--")), USER_REQUEST)  # 引数があれば質問にする

    interventions = []
    gate = None
    if use_gate:
        decider = Decider()
        try:
            print(decider.banner(), "\n")
        except ServerUnavailable as exc:
            print(exc)
            return 1
        gate = ToolCallReviewer(decider)
        interventions.append(gate)

    agent = Agent(
        model=model,
        tools=[get_weather],
        interventions=interventions,
        system_prompt=SYSTEM_PROMPT,
        callback_handler=None,  # ストリーミング表示は切る
    )

    print(f"USER: {request}\n")
    result = agent(request)

    if gate is not None:
        guided = sum(1 for row in gate.log if row["verdict"] == "guided")
        print(f"差し戻し {guided} 回 / ツール呼び出しの提案 {len(gate.log)} 回")
    print(f"\n--- エージェントの最終的な返事 ---\n{result}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

## 挙動の確認

公式サンプルの題材は天気を答えるエージェントです。システムプロンプトでわざと「都市がわからなければ聞き返さずに Seattle だと決めつけろ」と指示してあり、都市を書かずに「What's the weather?」と聞きます。

なお、比較のために、手順4で足した `--no-gate` オプションで、判定を挟まない場合も実行してみます。

### 判定を挟まない場合

```bash
uv run decider_lmstudio.py --no-gate
```

```text
# 実行結果
USER: What's the weather?

[tool] get_weather(location='Seattle') を実行した

--- エージェントの最終的な返事 ---
The current weather in Seattle is 21 °C and sunny.
```

都市を聞かれていないのに、Seattle の天気を答えてしまいました。システムプロンプトでわざとそう仕向けているので、ここまでは想定どおりです。この「聞き返さずに決めつける」動きを、Strands Decider で止められるかを見ていきます。

### 判定を挟む場合

```bash
uv run decider_lmstudio.py
```

```text
# 実行結果
USER: What's the weather?

[decider] get_weather({"location": "Seattle"})  2226 ms
    args_grounded : 0.13
    premature     : 0.70
    -> guided

差し戻し 1 回 / ツール呼び出しの提案 1 回

--- エージェントの最終的な返事 ---
I’m happy to help! Could you let me know which city you’re interested in? If you don’t specify a location, I’ll assume Seattle.
```

Strands Decider が「引数に根拠がない（0.13）」「呼ぶのが早すぎる（0.70）」と判定し、ツールは実行されずに差し戻されました。エージェントはどの都市かを聞き返しています。

判定にかかった時間は、サーバーを起動して最初の問い合わせが2.2秒、2回目以降は0.7〜1秒ほどでした。ただ、初めての長さの入力では準備に時間がかかるようで、別の質問に変えた直後は2秒ほどかかることがありました。

### 都市を書いても終わらない差し戻し

ここで、都市を書いた「What's the weather in Tokyo?」を試したところ、処理が終わらなくなりました。Strands Decider のサーバーのログを見ると、問い合わせが30回を超えても続いていました。

原因を探るため、いくつかの言い回しで Strands Decider に直接聞いてみました。

| 質問 | 引数 | args_grounded | premature |
|---|---|---|---|
| What's the weather? | Seattle | 0.13 | 0.70 |
| What's the weather where I live? | Seattle | 0.13 | 0.73 |
| Is it going to rain today? | Tokyo | 0.12 | 0.68 |
| 天気を教えて | Tokyo | 0.20 | 0.55 |
| What's the weather in Tokyo? | Tokyo | 0.37 | 0.42 |
| Tell me the weather in Paris, please. | Paris | 0.43 | 0.36 |
| 東京の天気は？ | Tokyo | 0.43 | 0.39 |
| I'm in Osaka today. How's the weather? | Osaka | 0.67 | 0.41 |

上の4つが根拠のない呼び出し、下の4つが根拠のある呼び出しです。どちらの質問も確率はきれいに分かれていますが、公式サンプルでは両方とも0.45を閾値にしています。args_grounded は根拠があっても0.37〜0.43にとどまるものがあり、ここで差し戻されていました。

差し戻されたモデルはまた同じ呼び出しをするので、差し戻しと呼び直しが延々と続いていたわけです。

### 閾値の調整と差し戻しの上限

公式サンプルのコメントにも、閾値は自分のトラフィックに合わせて選ぶよう書かれています。上の表から、args_grounded の閾値だけを0.30に下げました。あわせて、判定が外れ続けても止まるように、差し戻しは2回までという上限を足しました。

```python
# yes とみなす確率。公式サンプルは両方 0.45 だが、根拠ありでも差し戻されたので args_grounded は 0.30 に下げた
GROUNDED_YES = 0.30
PREMATURE_YES = 0.45

# 差し戻しの上限（公式サンプルには無い。無いと差し戻しと呼び直しが終わらないことがある）
MAX_GUIDES = 2
```

これで、都市を書いた質問はそのまま通るようになりました。

```bash
uv run decider_lmstudio.py "東京の天気は？"
```

```text
# 実行結果
USER: 東京の天気は？

[decider] get_weather({"location": "Tokyo"})  1023 ms
    args_grounded : 0.43
    premature     : 0.39
    -> proceeded

[tool] get_weather(location='Tokyo') を実行した
差し戻し 0 回 / ツール呼び出しの提案 1 回

--- エージェントの最終的な返事 ---
東京の天気は、21 ℃で晴れです。
```

### 差し戻しに従うかは LLM 次第

最後に、都市を書かない質問を5回続けて流してみました。

| 結果 | 回数 |
|---|---|
| 聞き返した | 2回 |
| 呼び直しを続けて上限で通った | 3回 |

Strands Decider の判定は毎回同じ（args_grounded 0.13）で、きちんと止めています。それでも gpt-oss-20b は、差し戻しよりもシステムプロンプトの「聞き返すな」を優先することが多く、3回は同じ呼び出しを繰り返して上限に達しました。別の回では、ツールを呼ばずに天気をでっち上げて答えたこともありました。

差し戻し（Guide）は、モデルに「こうしたら？」と返すだけで、従わせる仕組みではありません。止めるかどうかを決めるのは Strands Decider、そのあとどうするかを決めるのは LLM という分担です。

## おまけ：実行中のメモリ使用量

すべてを Mac の中で動かすとどのくらいメモリを使うのか、macOS の `footprint` コマンドでプロセスごとに測ってみました。LM Studio はモデルを別のプロセスで動かしているので、そのプロセスを測っています。

```bash
footprint -p <PID>
```

| プロセス | 待機中 | 判定を流している最中 | 起動してからの最大 |
|---|---|---|---|
| Strands Decider のサーバー | 4.5GB | 4.7GB | 4.9GB |
| LM Studio（gpt-oss-20b） | 12.6GB | 12.9GB | 13.9GB |
| エージェント（Python） | — | 0.1GB | — |

判定を流している最中は、合わせて17.7GB ほどでした。32GB の Mac なら、ほかの作業をしながらでも動かせそうです。「起動してからの最大」には、差し戻しがループしたときの分も含まれています。

## おわりに

Strands Decider は LM Studio に載せるものではありませんでしたが、LM Studio の LLM と並べて動かすことで、エージェントの判断まで含めて Mac の中だけで完結させられました。

判定そのものは安定していて、根拠のある呼び出しとない呼び出しの確率もきれいに分かれていました。一方で、公式サンプルの閾値をそのまま使うと差し戻しが止まらなくなるので、閾値は自分の質問で確率を見てから決めるのがよさそうです。差し戻しに上限を付けておくのも大事だと感じました。

今回は意地悪なシステムプロンプトのままだったので、もう少し素直なプロンプトにしたときや、別のローカル LLM に替えたときに差し戻しへの従い方がどう変わるかも試してみたいと思います。

ありがとうございました。
