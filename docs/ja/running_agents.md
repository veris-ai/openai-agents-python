---
search:
  exclude: true
---
# エージェントの実行

エージェントは [`Runner`][agents.run.Runner] クラスを介して実行できます。オプションは 3 つあります。

1. [`Runner.run()`][agents.run.Runner.run]  
   非同期で実行され、[`RunResult`][agents.result.RunResult] を返します。  
2. [`Runner.run_sync()`][agents.run.Runner.run_sync]  
   同期メソッドで、内部では `.run()` を呼び出します。  
3. [`Runner.run_streamed()`][agents.run.Runner.run_streamed]  
   非同期で実行され、[`RunResultStreaming`][agents.result.RunResultStreaming] を返します。LLM をストリーミング モードで呼び出し、受信したイベントをそのままストリーム配信します。

```python
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="You are a helpful assistant")

    result = await Runner.run(agent, "Write a haiku about recursion in programming.")
    print(result.final_output)
    # Code within the code,
    # Functions calling themselves,
    # Infinite loop's dance
```

詳細は [結果ガイド](results.md) を参照してください。

## エージェントループ

`Runner` の run メソッドを使用する際は、開始エージェントと入力を渡します。入力は文字列（ユーザー メッセージと見なされます）か、OpenAI Responses API のアイテムである入力アイテムのリストのいずれかです。

Runner は次のループを実行します。

1. 現在のエージェントに対して現在の入力を用い、LLM を呼び出します。  
2. LLM が出力を生成します。  
    1. LLM が `final_output` を返した場合、ループを終了して結果を返します。  
    2. LLM がハンドオフを行った場合、現在のエージェントと入力を更新し、ループを再実行します。  
    3. LLM がツール呼び出しを生成した場合、それらを実行し結果を追加して、ループを再実行します。  
3. 渡された `max_turns` を超えた場合、[`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded] 例外を送出します。

!!! note

    LLM 出力が「最終出力」と見なされるルールは、望ましい型のテキスト出力を生成し、ツール呼び出しが存在しない場合です。

## ストリーミング

ストリーミングを使用すると、LLM 実行中のストリーミング イベントを受け取れます。ストリームが完了すると、[`RunResultStreaming`][agents.result.RunResultStreaming] に実行に関する完全な情報（新しく生成されたすべての出力を含む）が格納されます。`.stream_events()` を呼び出してストリーミング イベントを取得できます。詳細は [ストリーミング ガイド](streaming.md) を参照してください。

## 実行設定

`run_config` パラメーターでは、エージェント実行のグローバル設定を指定できます。

- [`model`][agents.run.RunConfig.model]: 各 Agent の `model` 設定に関わらず、グローバルで使用する LLM モデルを指定します。  
- [`model_provider`][agents.run.RunConfig.model_provider]: モデル名を解決するモデル プロバイダー。デフォルトは OpenAI です。  
- [`model_settings`][agents.run.RunConfig.model_settings]: エージェント固有の設定を上書きします。たとえば、グローバルな `temperature` や `top_p` を指定できます。  
- [`input_guardrails`][agents.run.RunConfig.input_guardrails], [`output_guardrails`][agents.run.RunConfig.output_guardrails]: すべての実行に適用する入力または出力ガードレールの一覧です。  
- [`handoff_input_filter`][agents.run.RunConfig.handoff_input_filter]: すでにハンドオフ側にフィルターがない場合に適用される、すべてのハンドオフに対するグローバル入力フィルターです。新しいエージェントへ送信する入力を編集できます。詳細は [`Handoff.input_filter`][agents.handoffs.Handoff.input_filter] のドキュメントを参照してください。  
- [`tracing_disabled`][agents.run.RunConfig.tracing_disabled]: 実行全体の [トレーシング](tracing.md) を無効にします。  
- [`trace_include_sensitive_data`][agents.run.RunConfig.trace_include_sensitive_data]: トレースに LLM やツール呼び出しの入出力など、機微なデータを含めるかどうかを設定します。  
- [`workflow_name`][agents.run.RunConfig.workflow_name], [`trace_id`][agents.run.RunConfig.trace_id], [`group_id`][agents.run.RunConfig.group_id]: 実行のトレーシング ワークフロー名、トレース ID、トレース グループ ID を設定します。少なくとも `workflow_name` を設定することを推奨します。`group_id` は任意で、複数の実行間でトレースを関連付ける際に使用します。  
- [`trace_metadata`][agents.run.RunConfig.trace_metadata]: すべてのトレースに付与するメタデータです。  

## 会話 / チャットスレッド

いずれかの run メソッド呼び出しにより、1 つ以上のエージェントが実行され（つまり 1 回以上の LLM 呼び出しが行われ）ますが、チャット会話における 1 つの論理ターンとして扱われます。例:

1. ユーザーターン: ユーザーがテキストを入力  
2. Runner 実行: 最初のエージェントが LLM を呼び出し、ツールを実行し、2 番目のエージェントへハンドオフ。2 番目のエージェントがさらにツールを実行し、出力を生成。  

エージェント実行の終了時に、ユーザーへ何を表示するかを選択できます。たとえば、エージェントが生成したすべての新規アイテムを表示するか、最終出力のみを表示するかを決められます。いずれの場合でも、ユーザーが追質問を行ったら、再度 run メソッドを呼び出せます。

### 会話の手動管理

[`RunResultBase.to_input_list()`][agents.result.RunResultBase.to_input_list] メソッドを使用して次ターンの入力を取得し、会話履歴を手動で管理できます。

```python
async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    thread_id = "thread_123"  # Example thread ID
    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?")
        print(result.final_output)
        # San Francisco

        # Second turn
        new_input = result.to_input_list() + [{"role": "user", "content": "What state is it in?"}]
        result = await Runner.run(agent, new_input)
        print(result.final_output)
        # California
```

### Sessions での自動会話管理

より簡単な方法として、[Sessions](sessions.md) を利用すると `.to_input_list()` を手動で呼び出さずに会話履歴を自動管理できます。

```python
from agents import Agent, Runner, SQLiteSession

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    # Create session instance
    session = SQLiteSession("conversation_123")

    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?", session=session)
        print(result.final_output)
        # San Francisco

        # Second turn - agent automatically remembers previous context
        result = await Runner.run(agent, "What state is it in?", session=session)
        print(result.final_output)
        # California
```

Sessions は以下を自動で行います。

- 各実行前に会話履歴を取得  
- 各実行後に新規メッセージを保存  
- 異なるセッション ID ごとに個別の会話を維持  

詳細は [Sessions のドキュメント](sessions.md) を参照してください。

## 例外

SDK は特定のケースで例外を送出します。完全な一覧は [`agents.exceptions`][] にあります。概要は以下のとおりです。

- [`AgentsException`][agents.exceptions.AgentsException]: SDK 内で送出されるすべての例外の基底クラス。  
- [`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded]: 実行が run メソッドに渡した `max_turns` を超えた場合に送出。  
- [`ModelBehaviorError`][agents.exceptions.ModelBehaviorError]: モデルが不正な出力（壊れた JSON や存在しないツールの使用など）を生成した場合に送出。  
- [`UserError`][agents.exceptions.UserError]: SDK を使用する際に開発者が誤った使い方をした場合に送出。  
- [`InputGuardrailTripwireTriggered`][agents.exceptions.InputGuardrailTripwireTriggered], [`OutputGuardrailTripwireTriggered`][agents.exceptions.OutputGuardrailTripwireTriggered]: [ガードレール](guardrails.md) が発動した場合に送出。