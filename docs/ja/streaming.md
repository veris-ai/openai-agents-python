---
search:
  exclude: true
---
# ストリーミング

ストリーミングを利用すると、エージェント実行の進行に合わせて更新を購読できます。これにより、 エンドユーザー に進捗状況や部分的なレスポンスを表示するのに役立ちます。

ストリーミングを行うには、 [`Runner.run_streamed()`][agents.run.Runner.run_streamed] を呼び出します。これにより [`RunResultStreaming`][agents.result.RunResultStreaming] が返されます。続いて `result.stream_events()` を呼び出すと、以下で説明する [`StreamEvent`][agents.stream_events.StreamEvent] オブジェクトの非同期ストリームを取得できます。

## Raw response イベント

[`RawResponsesStreamEvent`][agents.stream_events.RawResponsesStreamEvent] は LLM から直接渡される raw イベントです。これらは OpenAI Responses API フォーマットで提供され、各イベントには `response.created` や `response.output_text.delta` などの type と data が含まれます。生成されたメッセージを即座に ユーザー にストリーミングしたい場合に便利です。

たとえば、以下のコードは LLM が生成するテキストをトークンごとに出力します。

```python
import asyncio
from openai.types.responses import ResponseTextDeltaEvent
from agents import Agent, Runner

async def main():
    agent = Agent(
        name="Joker",
        instructions="You are a helpful assistant.",
    )

    result = Runner.run_streamed(agent, input="Please tell me 5 jokes.")
    async for event in result.stream_events():
        if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
            print(event.data.delta, end="", flush=True)


if __name__ == "__main__":
    asyncio.run(main())
```

## Run item イベント と エージェント イベント

[`RunItemStreamEvent`][agents.stream_events.RunItemStreamEvent] はより高レベルのイベントで、アイテムが完全に生成されたタイミングを通知します。これにより、各トークンではなく「メッセージ生成完了」「ツール実行完了」などのレベルで進捗を ユーザー に送信できます。同様に、 [`AgentUpdatedStreamEvent`][agents.stream_events.AgentUpdatedStreamEvent] はハンドオフの結果などで現在のエージェントが変更された際に通知を行います。

たとえば、以下のコードは raw イベントを無視し、 ユーザー へ更新のみをストリーミングします。

```python
import asyncio
import random
from agents import Agent, ItemHelpers, Runner, function_tool

@function_tool
def how_many_jokes() -> int:
    return random.randint(1, 10)


async def main():
    agent = Agent(
        name="Joker",
        instructions="First call the `how_many_jokes` tool, then tell that many jokes.",
        tools=[how_many_jokes],
    )

    result = Runner.run_streamed(
        agent,
        input="Hello",
    )
    print("=== Run starting ===")

    async for event in result.stream_events():
        # We'll ignore the raw responses event deltas
        if event.type == "raw_response_event":
            continue
        # When the agent updates, print that
        elif event.type == "agent_updated_stream_event":
            print(f"Agent updated: {event.new_agent.name}")
            continue
        # When items are generated, print them
        elif event.type == "run_item_stream_event":
            if event.item.type == "tool_call_item":
                print("-- Tool was called")
            elif event.item.type == "tool_call_output_item":
                print(f"-- Tool output: {event.item.output}")
            elif event.item.type == "message_output_item":
                print(f"-- Message output:\n {ItemHelpers.text_message_output(event.item)}")
            else:
                pass  # Ignore other event types

    print("=== Run complete ===")


if __name__ == "__main__":
    asyncio.run(main())
```