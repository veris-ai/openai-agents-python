---
search:
  exclude: true
---
# ツール

ツールは エージェント がアクションを実行できるようにします。たとえばデータの取得、コード実行、外部 API 呼び出し、さらにはコンピュータ操作 などです。Agents SDK にはツールのクラスが 3 つあります。

-   ホスト型ツール: これらは AI モデルと同じ LLM サーバー 上で動作します。OpenAI は ファイル検索、Web 検索、コンピュータ操作 をホスト型ツールとして提供しています。
-   関数呼び出し: 任意の Python 関数をツールとして使用できます。
-   ツールとしてのエージェント: エージェント をツールとして使用でき、ハンドオフ せずに他の エージェント を呼び出せます。

## ホスト型ツール

OpenAI は [`OpenAIResponsesModel`][agents.models.openai_responses.OpenAIResponsesModel] を使用する際に、いくつかの組み込みツールを提供しています。

-   [`WebSearchTool`][agents.tool.WebSearchTool] は エージェント に Web を検索させます。
-   [`FileSearchTool`][agents.tool.FileSearchTool] は OpenAI の ベクトルストア から情報を取得します。
-   [`ComputerTool`][agents.tool.ComputerTool] は コンピュータ操作 の自動化を可能にします。
-   [`CodeInterpreterTool`][agents.tool.CodeInterpreterTool] は LLM がサンドボックス環境でコードを実行できるようにします。
-   [`HostedMCPTool`][agents.tool.HostedMCPTool] はリモートの MCP サーバー のツールをモデルに公開します。
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

## 関数ツール

任意の Python 関数をツールとして使用できます。Agents SDK が自動でツールを設定します。

-   ツール名は Python 関数名になります（または任意の名前を指定できます）
-   ツールの説明は関数の docstring から取得されます（または説明を指定できます）
-   関数入力のスキーマは関数の引数から自動生成されます
-   各入力の説明は、無効化しない限り、関数の docstring から取得されます

関数シグネチャの抽出には Python の `inspect` モジュール、docstring の解析には [`griffe`](https://mkdocstrings.github.io/griffe/)、スキーマ作成には `pydantic` を使用します。

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

1.  関数の引数には任意の Python 型を使用でき、関数は同期・非同期どちらでも構いません。
2.  docstring がある場合、説明および引数の説明の取得に使用します。
3.  関数は任意で `context` を最初の引数として受け取れます。ツール名や説明、docstring スタイルなどの上書きも設定できます。
4.  デコレートした関数はツールのリストに渡せます。

??? note "展開して出力を見る"

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

Python 関数をツールとして使いたくない場合もあります。その場合は、[`FunctionTool`][agents.tool.FunctionTool] を直接作成できます。以下を指定する必要があります。

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

前述のとおり、関数シグネチャを自動解析してツールのスキーマを抽出し、docstring を解析してツールと各引数の説明を抽出します。補足:

1.  シグネチャ解析は `inspect` モジュールで行います。型アノテーションから引数の型を把握し、全体スキーマを表す Pydantic モデル を動的に構築します。Python の基本型、Pydantic モデル、TypedDicts など大半の型をサポートします。
2.  docstring の解析には `griffe` を使用します。サポートする docstring 形式は `google`、`sphinx`、`numpy` です。形式の自動検出も試みますがベストエフォートのため、`function_tool` 呼び出し時に明示的に指定できます。`use_docstring_info` を `False` に設定して docstring 解析を無効化することもできます。

スキーマ抽出のコードは [`agents.function_schema`][] にあります。

## ツールとしてのエージェント

一部のワークフローでは、制御をハンドオフ する代わりに、中央の エージェント が専門 エージェント のネットワークをオーケストレーションしたい場合があります。エージェント をツールとしてモデル化することで実現できます。

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

`agent.as_tool` 関数は、エージェント を簡単にツール化するための補助メソッドです。ただし、すべての設定をサポートしているわけではありません。たとえば `max_turns` は設定できません。高度なユースケースでは、ツール実装内で `Runner.run` を直接使用してください。

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

### カスタム出力抽出

場合によっては、中央の エージェント に返す前にツール化した エージェント の出力を変更したいことがあります。たとえば次のような場合に有用です。

- サブエージェント のチャット履歴から特定の情報（例: JSON ペイロード）を抽出する。
- エージェント の最終回答を変換・再整形する（例: Markdown をプレーンテキストや CSV に変換）。
- エージェント の応答が欠落または不正な場合に、出力を検証したりフォールバック値を提供したりする。

これは、`as_tool` メソッドに `custom_output_extractor` 引数を渡すことで行えます。

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

## 関数ツールにおけるエラー処理

`@function_tool` で関数ツールを作成する際、`failure_error_function` を渡せます。これは、ツール呼び出しがクラッシュした場合に LLM へ返すエラー応答を提供する関数です。

-   既定（何も渡さない場合）では、エラーが発生したことを LLM に伝える `default_tool_error_function` が実行されます。
-   独自のエラー関数を渡した場合はそれが実行され、その応答が LLM に送信されます。
-   明示的に `None` を渡した場合、ツール呼び出しのエラーは再スローされ、呼び出し元で処理する必要があります。モデルが不正な JSON を生成した場合は `ModelBehaviorError`、コードがクラッシュした場合は `UserError` などになり得ます。

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

`FunctionTool` オブジェクトを手動で作成する場合は、`on_invoke_tool` 関数の内部でエラー処理を行う必要があります。