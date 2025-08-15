---
search:
  exclude: true
---
# LiteLLM 経由の任意モデルの利用

!!! note

    LiteLLM 統合はベータ版です。特に小規模なモデルプロバイダーでは問題が発生することがあります。問題があれば [Github の issues](https://github.com/openai/openai-agents-python/issues) からご報告ください。迅速に修正します。

[LiteLLM](https://docs.litellm.ai/docs/) は、単一のインターフェースで 100+ のモデルを利用できるライブラリです。Agents SDK内で任意の AI モデルを利用できるように、 LiteLLM 統合を追加しました。

## セットアップ

`litellm` が利用可能であることを確認してください。オプションの `litellm` 依存関係グループをインストールすることで対応できます:

```bash
pip install "openai-agents[litellm]"
```

完了したら、任意のエージェントで [`LitellmModel`][agents.extensions.models.litellm_model.LitellmModel] を使用できます。

## 例

これは完全に動作する例です。実行すると、モデル名と API キーの入力を求められます。たとえば、次のように入力できます:

-   `openai/gpt-4.1` をモデルに、OpenAIの API キーを入力
-   `anthropic/claude-3-5-sonnet-20240620` をモデルに、 Anthropic の API キーを入力
-   など

LiteLLM でサポートされているモデルの一覧は、[LiteLLM のプロバイダー ドキュメント](https://docs.litellm.ai/docs/providers) を参照してください。

```python
from __future__ import annotations

import asyncio

from agents import Agent, Runner, function_tool, set_tracing_disabled
from agents.extensions.models.litellm_model import LitellmModel

@function_tool
def get_weather(city: str):
    print(f"[debug] getting weather for {city}")
    return f"The weather in {city} is sunny."


async def main(model: str, api_key: str):
    agent = Agent(
        name="Assistant",
        instructions="You only respond in haikus.",
        model=LitellmModel(model=model, api_key=api_key),
        tools=[get_weather],
    )

    result = await Runner.run(agent, "What's the weather in Tokyo?")
    print(result.final_output)


if __name__ == "__main__":
    # First try to get model/api key from args
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("--model", type=str, required=False)
    parser.add_argument("--api-key", type=str, required=False)
    args = parser.parse_args()

    model = args.model
    if not model:
        model = input("Enter a model name for Litellm: ")

    api_key = args.api_key
    if not api_key:
        api_key = input("Enter an API key for Litellm: ")

    asyncio.run(main(model, api_key))
```