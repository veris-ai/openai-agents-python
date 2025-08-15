---
search:
  exclude: true
---
# ツール

ツールを使用すると、エージェントはデータ取得、コード実行、外部 API 呼び出し、さらにはコンピュータ操作などのアクションを実行できます。Agents SDK には 3 種類のツールがあります。

-   ホスト型ツール: これらは LLM サーバー上で AI モデルと一緒に実行されます。OpenAI は retrieval、Web 検索、コンピュータ操作をホスト型ツールとして提供しています。
-   関数ツール: 任意の Python 関数をツールとして利用できます。
-   ツールとしてのエージェント: エージェントをツールとして扱い、ハンドオフせずに他のエージェントを呼び出せます。

## ホスト型ツール

[`OpenAIResponsesModel`][agents.models.openai_responses.OpenAIResponsesModel] を使用する場合、OpenAI はいくつかの組み込みツールを提供しています。

-   [`WebSearchTool`][agents.tool.WebSearchTool] はエージェントに Web 検索を行わせます。
-   [`FileSearchTool`][agents.tool.FileSearchTool] は OpenAI ベクトルストアから情報を取得します。
-   [`ComputerTool`][agents.tool.ComputerTool] はコンピュータ操作タスクを自動化します。
-   [`CodeInterpreterTool`][agents.tool.CodeInterpreterTool] は LLM にサンドボックス環境でコードを実行させます。
-   [`HostedMCPTool`][agents.tool.HostedMCPTool] はリモート MCP サーバーのツールをモデルに公開します。
-   [`ImageGenerationTool`][agents.tool.ImageGenerationTool] はプロンプトから画像を生成します。
-   [`LocalShellTool`][agents.tool.LocalShellTool] はローカルマシン上でシェルコマンドを実行します。

```python
from agents import Agent, FileSearchTool, Runner, WebSearchTool

agent = Agent(
    name="Assistant",
    tools=[
        WebSearchTool(),
        FileSearchTool(
            max_num_results=3,
            vector_store_ids=["VECTOR_STORE_ID"],
        ),
    ],
)

async def main():
    result = await Runner.run(agent, "Which coffee shop should I go to, taking into account my preferences and the weather today in SF?")
    print(result.final_output)
```

## 関数ツール

任意の Python 関数をツールとして利用できます。Agents SDK が自動的にセットアップを行います。

-   ツール名は Python 関数名になります（または任意で指定可能）
-   ツールの説明は関数の docstring から取得されます（または任意で指定可能）
-   関数の引数から入力スキーマを自動生成します
-   各入力の説明は docstring から取得されます（無効化も可能）

Python の `inspect` モジュールで関数シグネチャを抽出し、docstring 解析には [`griffe`](https://mkdocstrings.github.io/griffe/)、スキーマ生成には `pydantic` を使用しています。

```python
import json

from typing_extensions import TypedDict, Any

from agents import Agent, FunctionTool, RunContextWrapper, function_tool


class Location(TypedDict):
    lat: float
    long: float

@function_tool  # (1)!
async def fetch_weather(location: Location) -> str:
    # (2)!
    """Fetch the weather for a given location.

    Args:
        location: The location to fetch the weather for.
    """
    # In real life, we'd fetch the weather from a weather API
    return "sunny"


@function_tool(name_override="fetch_data")  # (3)!
def read_file(ctx: RunContextWrapper[Any], path: str, directory: str | None = None) -> str:
    """Read the contents of a file.

    Args:
        path: The path to the file to read.
        directory: The directory to read the file from.
    """
    # In real life, we'd read the file from the file system
    return "<file contents>"


agent = Agent(
    name="Assistant",
    tools=[fetch_weather, read_file],  # (4)!
)

for tool in agent.tools:
    if isinstance(tool, FunctionTool):
        print(tool.name)
        print(tool.description)
        print(json.dumps(tool.params_json_schema, indent=2))
        print()

```

1.  関数の引数には任意の Python 型を使用でき、同期関数・非同期関数のどちらでも構いません。  
2.  docstring が存在する場合、ツールおよび引数の説明として利用されます。  
3.  関数は任意で `context`（先頭の引数）を受け取れます。また、ツール名や説明、docstring スタイルなどの上書き設定も可能です。  
4.  デコレート済みの関数をツール一覧に渡すだけで使用できます。  

??? note "出力を表示するには展開"

    ```
    fetch_weather
    Fetch the weather for a given location.
    {
    "$defs": {
      "Location": {
        "properties": {
          "lat": {
            "title": "Lat",
            "type": "number"
          },
          "long": {
            "title": "Long",
            "type": "number"
          }
        },
        "required": [
          "lat",
          "long"
        ],
        "title": "Location",
        "type": "object"
      }
    },
    "properties": {
      "location": {
        "$ref": "#/$defs/Location",
        "description": "The location to fetch the weather for."
      }
    },
    "required": [
      "location"
    ],
    "title": "fetch_weather_args",
    "type": "object"
    }

    fetch_data
    Read the contents of a file.
    {
    "properties": {
      "path": {
        "description": "The path to the file to read.",
        "title": "Path",
        "type": "string"
      },
      "directory": {
        "anyOf": [
          {
            "type": "string"
          },
          {
            "type": "null"
          }
        ],
        "default": null,
        "description": "The directory to read the file from.",
        "title": "Directory"
      }
    },
    "required": [
      "path"
    ],
    "title": "fetch_data_args",
    "type": "object"
    }
    ```

### カスタム関数ツール

Python 関数をそのままツールにしたくない場合は、[`FunctionTool`][agents.tool.FunctionTool] を直接作成できます。必要な項目は以下のとおりです。

-   `name`
-   `description`
-   `params_json_schema`: 引数の JSON スキーマ
-   `on_invoke_tool`: [`ToolContext`][agents.tool_context.ToolContext] と JSON 文字列形式の引数を受け取り、ツール出力を文字列で返す非同期関数

```python
from typing import Any

from pydantic import BaseModel

from agents import RunContextWrapper, FunctionTool



def do_some_work(data: str) -> str:
    return "done"


class FunctionArgs(BaseModel):
    username: str
    age: int


async def run_function(ctx: RunContextWrapper[Any], args: str) -> str:
    parsed = FunctionArgs.model_validate_json(args)
    return do_some_work(data=f"{parsed.username} is {parsed.age} years old")


tool = FunctionTool(
    name="process_user",
    description="Processes extracted user data",
    params_json_schema=FunctionArgs.model_json_schema(),
    on_invoke_tool=run_function,
)
```

### 引数と docstring の自動解析

前述のとおり、ツール用スキーマを関数シグネチャから自動で抽出し、docstring からツールや各引数の説明を取得します。ポイントは次のとおりです。

1. `inspect` モジュールでシグネチャを解析し、型アノテーションから引数の型を判定して Pydantic モデルを動的に構築します。Python の基本型、Pydantic モデル、TypedDict など大半の型をサポートします。  
2. docstring 解析には `griffe` を使用します。対応フォーマットは `google`、`sphinx`、`numpy` です。フォーマットは自動推定しますが、`function_tool` 呼び出し時に明示的に指定もできます。`use_docstring_info` を `False` に設定すれば解析を無効化できます。  

スキーマ抽出のコードは [`agents.function_schema`][] にあります。

## ツールとしてのエージェント

ワークフローによっては、ハンドオフせずに中央のエージェントが複数の専門エージェントをオーケストレーションしたい場合があります。その際、エージェントをツールとしてモデル化できます。

```python
from agents import Agent, Runner
import asyncio

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You translate the user's message to Spanish",
)

french_agent = Agent(
    name="French agent",
    instructions="You translate the user's message to French",
)

orchestrator_agent = Agent(
    name="orchestrator_agent",
    instructions=(
        "You are a translation agent. You use the tools given to you to translate."
        "If asked for multiple translations, you call the relevant tools."
    ),
    tools=[
        spanish_agent.as_tool(
            tool_name="translate_to_spanish",
            tool_description="Translate the user's message to Spanish",
        ),
        french_agent.as_tool(
            tool_name="translate_to_french",
            tool_description="Translate the user's message to French",
        ),
    ],
)

async def main():
    result = await Runner.run(orchestrator_agent, input="Say 'Hello, how are you?' in Spanish.")
    print(result.final_output)
```

### ツール化エージェントのカスタマイズ

`agent.as_tool` はエージェントを簡単にツール化するためのヘルパーです。ただしすべての設定をサポートするわけではなく、たとえば `max_turns` は設定できません。高度なユースケースでは、ツール実装内で `Runner.run` を直接使用してください。

```python
@function_tool
async def run_my_agent() -> str:
    """A tool that runs the agent with custom configs"""

    agent = Agent(name="My agent", instructions="...")

    result = await Runner.run(
        agent,
        input="...",
        max_turns=5,
        run_config=...
    )

    return str(result.final_output)
```

### 出力のカスタム抽出

場合によっては、ツール化したエージェントの出力を中央エージェントへ返す前に加工したいことがあります。たとえば以下のようなケースです。

- サブエージェントのチャット履歴から特定情報（例: JSON ペイロード）のみを抽出する  
- エージェントの最終回答を変換・再フォーマットする（例: Markdown をプレーンテキストや CSV に変換）  
- 出力を検証し、不足または不正な場合にフォールバック値を返す  

これを行うには、`as_tool` メソッドに `custom_output_extractor` 引数を渡します。

```python
async def extract_json_payload(run_result: RunResult) -> str:
    # Scan the agent’s outputs in reverse order until we find a JSON-like message from a tool call.
    for item in reversed(run_result.new_items):
        if isinstance(item, ToolCallOutputItem) and item.output.strip().startswith("{"):
            return item.output.strip()
    # Fallback to an empty JSON object if nothing was found
    return "{}"


json_tool = data_agent.as_tool(
    tool_name="get_data_json",
    tool_description="Run the data agent and return only its JSON payload",
    custom_output_extractor=extract_json_payload,
)
```

## 関数ツールでのエラー処理

`@function_tool` で関数ツールを作成する際、`failure_error_function` を渡せます。これはツール呼び出しがクラッシュした場合に LLM へ返されるエラー応答を生成する関数です。

-   何も渡さなかった場合は、`default_tool_error_function` が実行され、LLM にエラーが発生したことを通知します。
-   独自のエラー関数を渡すと、それが実行され、その応答が LLM へ送信されます。
-   明示的に `None` を渡すと、ツール呼び出し時のエラーは再スローされます。モデルが無効な JSON を生成した場合は `ModelBehaviorError`、ユーザーコードがクラッシュした場合は `UserError` などになります。

`FunctionTool` オブジェクトを手動で作成する場合は、`on_invoke_tool` 内でエラーを処理する必要があります。