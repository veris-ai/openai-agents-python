---
search:
  exclude: true
---
# エージェント

エージェントはアプリの中心的な構成要素です。エージェントとは、指示とツールで構成された大規模言語モデル（ LLM ）です。

## 基本設定

エージェントを設定する際によく使うプロパティは次のとおりです。

- `name`: エージェントを識別する必須の文字列です。  
- `instructions`: 開発者メッセージ、またはシステムプロンプトとも呼ばれます。  
- `model`: 使用する LLM と、`temperature`、`top_p` などのモデル調整用パラメーターを設定する任意の `model_settings`。  
- `tools`: エージェントがタスクを達成するために利用できるツール群です。  

```python
from agents import Agent, ModelSettings, function_tool

@function_tool
def get_weather(city: str) -> str:
     """returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Haiku agent",
    instructions="Always respond in haiku form",
    model="o3-mini",
    tools=[get_weather],
)
```

## コンテキスト

エージェントは `context` 型を汎用的に扱います。コンテキストは依存性注入のためのツールで、`Runner.run()` に渡すオブジェクトです。これはすべてのエージェント、ツール、ハンドオフなどに渡され、エージェント実行時の依存関係や状態をまとめて保持します。任意の Python オブジェクトをコンテキストとして提供できます。

```python
@dataclass
class UserContext:
    name: str
    uid: str
    is_pro_user: bool

    async def fetch_purchases() -> list[Purchase]:
        return ...

agent = Agent[UserContext](
    ...,
)
```

## 出力タイプ

デフォルトでは、エージェントはプレーンテキスト（`str`）を出力します。特定の型で出力させたい場合は `output_type` パラメーターを使用します。一般的には [Pydantic](https://docs.pydantic.dev/) オブジェクトを指定しますが、Pydantic の [TypeAdapter](https://docs.pydantic.dev/latest/api/type_adapter/) でラップできる型（ dataclass、リスト、TypedDict など）であれば利用できます。

```python
from pydantic import BaseModel
from agents import Agent


class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar extractor",
    instructions="Extract calendar events from text",
    output_type=CalendarEvent,
)
```

!!! note

    `output_type` を渡すと、モデルは通常のプレーンテキスト応答ではなく [structured outputs](https://platform.openai.com/docs/guides/structured-outputs) を使用するようになります。

## ハンドオフ

ハンドオフは、エージェントが委譲できるサブエージェントです。ハンドオフのリストを渡すと、エージェントは必要に応じてそれらに委譲します。これにより、単一タスクに特化したモジュール型のエージェントをオーケストレーションする強力なパターンが実現します。詳細は [handoffs](handoffs.md) ドキュメントをご覧ください。

```python
from agents import Agent

booking_agent = Agent(...)
refund_agent = Agent(...)

triage_agent = Agent(
    name="Triage agent",
    instructions=(
        "Help the user with their questions."
        "If they ask about booking, handoff to the booking agent."
        "If they ask about refunds, handoff to the refund agent."
    ),
    handoffs=[booking_agent, refund_agent],
)
```

## 動的インストラクション

多くの場合、エージェント作成時に instructions を指定しますが、関数を通じて動的に instructions を提供することも可能です。その関数はエージェントとコンテキストを受け取り、プロンプトを返す必要があります。通常の関数と `async` 関数の両方を使用できます。

```python
def dynamic_instructions(
    context: RunContextWrapper[UserContext], agent: Agent[UserContext]
) -> str:
    return f"The user's name is {context.context.name}. Help them with their questions."


agent = Agent[UserContext](
    name="Triage agent",
    instructions=dynamic_instructions,
)
```

## ライフサイクルイベント（フック）

エージェントのライフサイクルを監視したい場合があります。たとえば、イベントをログに記録したり、特定のイベント発生時にデータを事前取得したりできます。`hooks` プロパティを使ってエージェントのライフサイクルにフックできます。[`AgentHooks`][agents.lifecycle.AgentHooks] クラスをサブクラス化し、必要なメソッドをオーバーライドしてください。

## ガードレール

ガードレールを使用すると、エージェント実行と並行してユーザー入力に対するチェックやバリデーションを実行できます。たとえば、ユーザー入力の関連性をスクリーニングすることが可能です。詳細は [guardrails](guardrails.md) ドキュメントをご覧ください。

## エージェントのクローン／コピー

エージェントの `clone()` メソッドを使用すると、エージェントを複製し、任意のプロパティを変更できます。

```python
pirate_agent = Agent(
    name="Pirate",
    instructions="Write like a pirate",
    model="o3-mini",
)

robot_agent = pirate_agent.clone(
    name="Robot",
    instructions="Write like a robot",
)
```

## ツール使用の強制

ツールをリストで渡しても、必ずしも LLM がツールを使用するとは限りません。[`ModelSettings.tool_choice`][agents.model_settings.ModelSettings.tool_choice] を設定することでツール使用を強制できます。利用可能な値は次のとおりです。

1. `auto`: LLM がツールを使用するか否かを決定します。  
2. `required`: LLM にツールの使用を必須とします（どのツールを使用するかは自動で選択）。  
3. `none`: LLM にツールを使用しないことを要求します。  
4. 特定の文字列（例: `my_tool`）を設定すると、そのツールを必ず使用します。  

```python
from agents import Agent, Runner, function_tool, ModelSettings

@function_tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    model_settings=ModelSettings(tool_choice="get_weather") 
)
```

## ツール使用時の挙動

`Agent` の `tool_use_behavior` パラメーターは、ツール出力の扱い方を制御します。
- `"run_llm_again"`: デフォルト。ツールを実行後、その結果を LLM が処理して最終応答を生成します。  
- `"stop_on_first_tool"`: 最初のツール呼び出しの出力を最終応答として使用し、以降の LLM 処理は行いません。  

```python
from agents import Agent, Runner, function_tool, ModelSettings

@function_tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    tool_use_behavior="stop_on_first_tool"
)
```

- `StopAtTools(stop_at_tool_names=[...])`: 指定したいずれかのツールが呼び出された時点で停止し、そのツールの出力を最終応答として使用します。  
```python
from agents import Agent, Runner, function_tool
from agents.agent import StopAtTools

@function_tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

@function_tool
def sum_numbers(a: int, b: int) -> int:
    """Adds two numbers."""
    return a + b

agent = Agent(
    name="Stop At Stock Agent",
    instructions="Get weather or sum numbers.",
    tools=[get_weather, sum_numbers],
    tool_use_behavior=StopAtTools(stop_at_tool_names=["get_weather"])
)
```
- `ToolsToFinalOutputFunction`: ツール結果を処理し、停止するか LLM 継続かを決定するカスタム関数です。  

```python
from agents import Agent, Runner, function_tool, FunctionToolResult, RunContextWrapper
from agents.agent import ToolsToFinalOutputResult
from typing import List, Any

@function_tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

def custom_tool_handler(
    context: RunContextWrapper[Any],
    tool_results: List[FunctionToolResult]
) -> ToolsToFinalOutputResult:
    """Processes tool results to decide final output."""
    for result in tool_results:
        if result.output and "sunny" in result.output:
            return ToolsToFinalOutputResult(
                is_final_output=True,
                final_output=f"Final weather: {result.output}"
            )
    return ToolsToFinalOutputResult(
        is_final_output=False,
        final_output=None
    )

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    tool_use_behavior=custom_tool_handler
)
```

!!! note

    無限ループを防ぐため、ツール呼び出し後にフレームワークは自動的に `tool_choice` を "auto" にリセットします。この挙動は [`agent.reset_tool_choice`][agents.agent.Agent.reset_tool_choice] で設定可能です。ツール結果が LLM に送られるたびに `tool_choice` により再度ツール呼び出しが発生し、無限に続く可能性があるためです。