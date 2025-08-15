---
search:
  exclude: true
---
# LiteLLM を介した任意モデルの利用

!!! note

    LiteLLM インテグレーションはベータ版です。特に小規模なモデルプロバイダーで問題が発生する可能性があります。[Github issues](https://github.com/openai/openai-agents-python/issues) から問題を報告いただければ、迅速に対応します。

[LiteLLM](https://docs.litellm.ai/docs/) は、単一のインターフェースで 100 以上のモデルを利用できるライブラリです。Agents SDK では LiteLLM インテグレーションを追加し、任意の AI モデルを使用できるようにしました。

## セットアップ

`litellm` が利用可能であることを確認してください。これは、オプションの `litellm` 依存グループをインストールすることで実現できます。

```bash
pip install "openai-agents[litellm]"
```

インストール後は、任意のエージェントで [`LitellmModel`][agents.extensions.models.litellm_model.LitellmModel] を使用できます。

## 例

以下は完全に動作する例です。実行するとモデル名と API キーの入力を求められます。たとえば次のように入力できます。

-   モデルに `openai/gpt-4.1`、API キーに OpenAI API キー
-   モデルに `anthropic/claude-3-5-sonnet-20240620`、API キーに Anthropic API キー
-   など

LiteLLM でサポートされているモデルの全一覧は、[litellm providers docs](https://docs.litellm.ai/docs/providers) を参照してください。

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