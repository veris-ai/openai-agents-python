---
search:
  exclude: true
---
# ツール

ツールを使うと、エージェントはデータの取得、コードの実行、外部 API の呼び出し、さらにはコンピュータ操作などのアクションを実行できます。Agents SDK には次の 3 種類のツールがあります。

-   Hosted tools: LLM サーバー上で AI モデルと並行して動作するツールです。OpenAI は retrieval、web search、computer use を hosted tools として提供しています。
-   Function calling: 任意の Python 関数をツールとして利用できます。
-   Agents as tools: エージェントをツールとして扱い、ハンドオフせずに他のエージェントを呼び出せます。

## Hosted tools

[`OpenAIResponsesModel`][agents.models.openai_responses.OpenAIResponsesModel] を使用すると、OpenAI がいくつかの組み込みツールを提供します。

-   [`WebSearchTool`][agents.tool.WebSearchTool] はエージェントに Web 検索を行わせます。
-   [`FileSearchTool`][agents.tool.FileSearchTool] は OpenAI ベクトルストアから情報を取得します。
-   [`ComputerTool`][agents.tool.ComputerTool] はコンピュータ操作タスクを自動化します。
-   [`CodeInterpreterTool`][agents.tool.CodeInterpreterTool] は LLM にサンドボックス環境でコードを実行させます。
-   [`HostedMCPTool`][agents.tool.HostedMCPTool] はリモート MCP サーバーのツールをモデルに公開します。
-   [`ImageGenerationTool`][agents.tool.ImageGenerationTool] はプロンプトから画像を生成します。
-   [`LocalShellTool`][agents.tool.LocalShellTool] はローカルマシンでシェルコマンドを実行します。

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

## Function tools

任意の Python 関数をツールとして利用できます。Agents SDK が自動でセットアップします。

-   ツール名は Python 関数名になります（別名を指定することも可能）。
-   ツールの説明は関数の docstring から取得されます（別途指定も可能）。
-   関数引数から入力スキーマを自動生成します。
-   各入力の説明は、無効化しない限り docstring から取得します。

Python の `inspect` モジュールで関数シグネチャを取得し、[`griffe`](https://mkdocstrings.github.io/griffe/) で docstring を解析、`pydantic` でスキーマを生成します。

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

1.  任意の Python 型を引数に使用でき、関数は sync／async どちらでも構いません。  
2.  Docstring があれば、ツールと各引数の説明を取得します。  
3.  関数は任意で `context`（先頭の引数）を受け取れます。また、ツール名や説明、docstring スタイルなどを上書き設定できます。  
4.  デコレートした関数をツールのリストに渡せます。  

??? note "展開して出力を表示"

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

### カスタム function tool

Python 関数ではなく、直接 [`FunctionTool`][agents.tool.FunctionTool] を作成したい場合もあります。その場合は次を指定してください。

-   `name`
-   `description`
-   `params_json_schema`（引数の JSON スキーマ）
-   `on_invoke_tool`（[`ToolContext`][agents.tool_context.ToolContext] と引数の JSON 文字列を受け取り、ツールの出力文字列を返す async 関数）

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

前述のとおり、関数シグネチャを解析してスキーマを生成し、docstring から説明を抽出します。ポイントは次のとおりです。

1. シグネチャ解析は `inspect` を使用します。型アノテーションから引数型を理解し、動的に Pydantic モデルを組み立てます。Python プリミティブ型、Pydantic モデル、TypedDict など多くの型をサポートします。  
2. Docstring 解析には `griffe` を使用します。対応フォーマットは `google`、`sphinx`、`numpy` です。自動検出を試みますが、`function_tool` 呼び出し時に明示的に設定もできます。`use_docstring_info` を `False` にすると docstring 解析を無効化できます。  

スキーマ抽出のコードは [`agents.function_schema`][] にあります。

## Agents as tools

ワークフローによっては、ハンドオフせずに中央エージェントが専門エージェント群をオーケストレーションしたい場合があります。その際はエージェントをツールとしてモデル化できます。

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

### ツール化したエージェントのカスタマイズ

`agent.as_tool` はエージェントを簡単にツール化するための便利メソッドです。ただし全設定に対応しているわけではなく、たとえば `max_turns` は設定できません。高度なユースケースでは、ツール実装内で `Runner.run` を直接呼び出してください。

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

場合によっては、ツール化したエージェントの出力を中央エージェントへ返す前に加工したいことがあります。たとえば次のようなケースです。

- サブエージェントのチャット履歴から特定情報（例: JSON ペイロード）のみ抽出したい。  
- エージェントの最終回答を別形式に変換したい（例: Markdown をプレーンテキストや CSV に変換）。  
- 出力を検証し、不足または不正な場合にフォールバック値を返したい。  

その場合は `as_tool` メソッドに `custom_output_extractor` 引数を渡します。

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

## function tools のエラー処理

`@function_tool` で function tool を作成するとき、`failure_error_function` を指定できます。これはツール呼び出しがクラッシュした場合に LLM へ渡すエラーレスポンスを生成する関数です。

-   省略した場合は `default_tool_error_function` が実行され、LLM にエラー発生を通知します。
-   独自のエラーファンクションを渡すと、それが実行されて LLM に返されます。
-   明示的に `None` を渡すと、ツール呼び出しエラーは再スローされます。このとき、モデルが不正な JSON を生成した場合は `ModelBehaviorError`、ユーザーコードがクラッシュした場合は `UserError` などが発生します。

```python
from agents import function_tool, RunContextWrapper
from typing import Any

def my_custom_error_function(context: RunContextWrapper[Any], error: Exception) -> str:
    """A custom function to provide a user-friendly error message."""
    print(f"A tool call failed with the following error: {error}")
    return "An internal server error occurred. Please try again later."

@function_tool(failure_error_function=my_custom_error_function)
def get_user_profile(user_id: str) -> str:
    """Fetches a user profile from a mock API.
     This function demonstrates a 'flaky' or failing API call.
    """
    if user_id == "user_123":
        return "User profile for user_123 successfully retrieved."
    else:
        raise ValueError(f"Could not retrieve profile for user_id: {user_id}. API returned an error.")

```

`FunctionTool` オブジェクトを手動で作成する場合は、`on_invoke_tool` 内でエラーを処理する必要があります。