---
search:
  exclude: true
---
# ガードレール

ガードレールはエージェントと _並列で_ 実行され、ユーザー入力のチェックと検証を行えます。たとえば、非常に賢い（そのため遅く／高価な）モデルを使ってカスタマーリクエストを処理するエージェントがあるとします。悪意のあるユーザーにそのモデルを使って数学の宿題を手伝わせたくはありません。そのため、速くて安価なモデルでガードレールを実行できます。ガードレールが悪意のある使用を検知すると、すぐにエラーを送出し、高価なモデルの実行を止めて時間とコストを節約できます。

ガードレールには 2 種類あります:

1. 入力ガードレール: 初期のユーザー入力に対して実行されます  
2. 出力ガードレール: 最終的なエージェント出力に対して実行されます  

## 入力ガードレール

入力ガードレールは次の 3 段階で実行されます:

1. まず、ガードレールはエージェントに渡されたものと同一の入力を受け取ります。  
2. 次に、ガードレール関数が実行され `GuardrailFunctionOutput` を生成し、それが `InputGuardrailResult` でラップされます。  
3. 最後に `.tripwire_triggered` が true かどうかを確認します。true の場合、`InputGuardrailTripwireTriggered` 例外が送出され、ユーザーへの応答や例外処理を適切に行えます。  

!!! Note

    入力ガードレールはユーザー入力に対して実行されることを想定しているため、エージェントが *最初の* エージェントである場合にのみ実行されます。「`guardrails` プロパティがエージェント上にあるのはなぜで、`Runner.run` に渡さないのか？」と疑問に思うかもしれません。ガードレールは実際のエージェントに密接に関連する傾向があるためです。エージェントごとに異なるガードレールを実行することになるので、コードを同じ場所に置くことで可読性が向上します。

## 出力ガードレール

出力ガードレールは次の 3 段階で実行されます:

1. まず、ガードレールはエージェントが生成した出力を受け取ります。  
2. 次に、ガードレール関数が実行され `GuardrailFunctionOutput` を生成し、それが `OutputGuardrailResult` でラップされます。  
3. 最後に `.tripwire_triggered` が true かどうかを確認します。true の場合、`OutputGuardrailTripwireTriggered` 例外が送出され、ユーザーへの応答や例外処理を適切に行えます。  

!!! Note

    出力ガードレールは最終的なエージェント出力に対して実行されることを想定しているため、エージェントが *最後の* エージェントである場合にのみ実行されます。入力ガードレールと同様に、ガードレールは実際のエージェントに密接に関連しているため、コードを同じ場所に置くことで可読性が向上します。

## トリップワイヤ

入力または出力がガードレールを通過できなかった場合、ガードレールはトリップワイヤでこれを通知できます。トリップワイヤが発動したガードレールを検知すると、ただちに `{Input,Output}GuardrailTripwireTriggered` 例外を送出し、エージェントの実行を停止します。

## ガードレールの実装

入力を受け取り、`GuardrailFunctionOutput` を返す関数を用意する必要があります。以下の例では、内部でエージェントを実行してこれを行います。

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    TResponseInputItem,
    input_guardrail,
)

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent( # (1)!
    name="Guardrail check",
    instructions="Check if the user is asking you to do their math homework.",
    output_type=MathHomeworkOutput,
)


@input_guardrail
async def math_guardrail( # (2)!
    ctx: RunContextWrapper[None], agent: Agent, input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output, # (3)!
        tripwire_triggered=result.final_output.is_math_homework,
    )


agent = Agent(  # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    input_guardrails=[math_guardrail],
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except InputGuardrailTripwireTriggered:
        print("Math homework guardrail tripped")
```

1. このエージェントをガードレール関数内で使用します。  
2. これはエージェントの入力／コンテキストを受け取り、結果を返すガードレール関数です。  
3. ガードレール結果に追加情報を含めることができます。  
4. これがワークフローを定義する実際のエージェントです。  

出力ガードレールも同様です。

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    OutputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    output_guardrail,
)
class MessageOutput(BaseModel): # (1)!
    response: str

class MathOutput(BaseModel): # (2)!
    reasoning: str
    is_math: bool

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the output includes any math.",
    output_type=MathOutput,
)

@output_guardrail
async def math_guardrail(  # (3)!
    ctx: RunContextWrapper, agent: Agent, output: MessageOutput
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, output.response, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math,
    )

agent = Agent( # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    output_guardrails=[math_guardrail],
    output_type=MessageOutput,
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except OutputGuardrailTripwireTriggered:
        print("Math output guardrail tripped")
```

1. これは実際のエージェントの出力型です。  
2. これはガードレールの出力型です。  
3. これはエージェントの出力を受け取り、結果を返すガードレール関数です。  
4. これがワークフローを定義する実際のエージェントです。