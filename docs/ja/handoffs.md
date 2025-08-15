---
search:
  exclude: true
---
# ハンドオフ

ハンドオフを使用すると、ある エージェント がタスクを別の エージェント に委任できます。これは、異なる エージェント がそれぞれ異なる分野を専門とするシナリオで特に便利です。たとえば、カスタマーサポート アプリには、注文状況、返金、 FAQ などのタスクをそれぞれ処理する エージェント が存在するかもしれません。

ハンドオフは LLM に対してはツールとして表現されます。そのため、`Refund Agent` という エージェント へのハンドオフがある場合、ツール名は `transfer_to_refund_agent` となります。

## ハンドオフの作成

すべての エージェント には [`handoffs`][agents.agent.Agent.handoffs] パラメーターがあります。ここには `Agent` を直接指定することも、ハンドオフをカスタマイズする `Handoff` オブジェクトを指定することもできます。

Agents SDK が提供する [`handoff()`][agents.handoffs.handoff] 関数を使用してハンドオフを作成できます。この関数では、ハンドオフ先の エージェント を指定できるほか、オーバーライドや入力フィルターも設定できます。

### 基本的な使い方

シンプルなハンドオフを作成する方法は次のとおりです:

```python
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

# (1)!
triage_agent = Agent(name="Triage agent", handoffs=[billing_agent, handoff(refund_agent)])
```

1. `billing_agent` のように エージェント を直接指定することも、`handoff()` 関数を使用することもできます。

### `handoff()` 関数によるハンドオフのカスタマイズ

[`handoff()`][agents.handoffs.handoff] 関数では、以下の項目をカスタマイズできます。

-   `agent`: ハンドオフ先となる エージェント です。  
-   `tool_name_override`: 既定では `Handoff.default_tool_name()` 関数が使用され、`transfer_to_<agent_name>` に解決されます。これを上書きできます。  
-   `tool_description_override`: `Handoff.default_tool_description()` の既定のツール説明を上書きします。  
-   `on_handoff`: ハンドオフが呼び出されたときに実行されるコールバック関数です。ハンドオフが呼び出されたことが分かった時点でデータフェッチを開始するなどに便利です。この関数は エージェント コンテキストを受け取り、オプションで LLM が生成した入力も受け取れます。入力データは `input_type` パラメーターで制御します。  
-   `input_type`: ハンドオフが受け取る入力の型です (オプション)。  
-   `input_filter`: 次の エージェント が受け取る入力をフィルタリングします。詳細は下記を参照してください。  

```python
from agents import Agent, handoff, RunContextWrapper

def on_handoff(ctx: RunContextWrapper[None]):
    print("Handoff called")

agent = Agent(name="My agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    tool_name_override="custom_handoff_tool",
    tool_description_override="Custom description",
)
```

## ハンドオフ入力

状況によっては、ハンドオフを呼び出す際に LLM に何らかのデータを渡してほしいことがあります。たとえば “Escalation agent” にハンドオフする場合、ログ用に理由 (reason) を受け取りたいなどが考えられます。

```python
from pydantic import BaseModel

from agents import Agent, handoff, RunContextWrapper

class EscalationData(BaseModel):
    reason: str

async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    print(f"Escalation agent called with reason: {input_data.reason}")

agent = Agent(name="Escalation agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    input_type=EscalationData,
)
```

## 入力フィルター

ハンドオフが発生すると、新しい エージェント が会話を引き継ぎ、これまでの会話履歴をすべて閲覧できる状態になります。これを変更したい場合は [`input_filter`][agents.handoffs.Handoff.input_filter] を設定できます。入力フィルターは、既存の入力を [`HandoffInputData`][agents.handoffs.HandoffInputData] 経由で受け取り、新しい `HandoffInputData` を返す関数です。

よくあるパターン (たとえば履歴からすべてのツール呼び出しを削除する) は [`agents.extensions.handoff_filters`][] に実装済みです。

```python
from agents import Agent, handoff
from agents.extensions import handoff_filters

agent = Agent(name="FAQ agent")

handoff_obj = handoff(
    agent=agent,
    input_filter=handoff_filters.remove_all_tools, # (1)!
)
```

1. これにより、`FAQ agent` が呼び出された際に履歴からすべてのツールが自動的に削除されます。

## 推奨プロンプト

LLM がハンドオフを正しく理解できるよう、エージェント にハンドオフに関する情報を含めることを推奨します。[`agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX`][] に推奨のプレフィックスが用意されているほか、[`agents.extensions.handoff_prompt.prompt_with_handoff_instructions`][] を呼び出してプロンプトに推奨データを自動的に追加することもできます。

```python
from agents import Agent
from agents.extensions.handoff_prompt import RECOMMENDED_PROMPT_PREFIX

billing_agent = Agent(
    name="Billing agent",
    instructions=f"""{RECOMMENDED_PROMPT_PREFIX}
    <Fill in the rest of your prompt here>.""",
)
```