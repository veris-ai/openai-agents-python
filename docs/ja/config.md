---
search:
  exclude: true
---
# SDK の設定

## API キーとクライアント

デフォルトでは、 SDK はインポートされるとすぐに LLM リクエストとトレーシングのために `OPENAI_API_KEY` 環境変数を探します。アプリ起動前にこの環境変数を設定できない場合は、 [`set_default_openai_key()`][agents.set_default_openai_key] 関数でキーを設定できます。

```python
from agents import set_default_openai_key

set_default_openai_key("sk-...")
```

また、使用する OpenAI クライアントを設定することも可能です。デフォルトでは、 SDK は環境変数もしくは前述のデフォルトキーを用いて `AsyncOpenAI` インスタンスを作成します。これを変更したい場合は、 [`set_default_openai_client()`][agents.set_default_openai_client] 関数を使用してください。

```python
from openai import AsyncOpenAI
from agents import set_default_openai_client

custom_client = AsyncOpenAI(base_url="...", api_key="...")
set_default_openai_client(custom_client)
```

さらに、利用する OpenAI API をカスタマイズすることもできます。デフォルトでは OpenAI Responses API を使用しますが、 [`set_default_openai_api()`][agents.set_default_openai_api] 関数を用いれば Chat Completions API を利用するように上書きできます。

```python
from agents import set_default_openai_api

set_default_openai_api("chat_completions")
```

## トレーシング

トレーシングはデフォルトで有効になっています。前節の OpenAI API キー（環境変数または設定したデフォルトキー）をそのまま使用します。トレーシングに使用する API キーを個別に設定したい場合は、 [`set_tracing_export_api_key()`][agents.set_tracing_export_api_key] 関数をご利用ください。

```python
from agents import set_tracing_export_api_key

set_tracing_export_api_key("sk-...")
```

トレーシングを完全に無効化したい場合は、 [`set_tracing_disabled()`][agents.set_tracing_disabled] 関数を呼び出してください。

```python
from agents import set_tracing_disabled

set_tracing_disabled(True)
```

## デバッグログ

SDK にはハンドラーが設定されていない Python ロガーが 2 つあります。デフォルトでは warning と error が `stdout` に送られ、それ以外のログは抑制されます。

詳細なログを有効にするには、 [`enable_verbose_stdout_logging()`][agents.enable_verbose_stdout_logging] 関数を使用してください。

```python
from agents import enable_verbose_stdout_logging

enable_verbose_stdout_logging()
```

ログをカスタマイズしたい場合は、ハンドラー・フィルター・フォーマッターなどを追加できます。詳細は [Python logging guide](https://docs.python.org/3/howto/logging.html) をご覧ください。

```python
import logging

logger = logging.getLogger("openai.agents") # or openai.agents.tracing for the Tracing logger

# To make all logs show up
logger.setLevel(logging.DEBUG)
# To make info and above show up
logger.setLevel(logging.INFO)
# To make warning and above show up
logger.setLevel(logging.WARNING)
# etc

# You can customize this as needed, but this will output to `stderr` by default
logger.addHandler(logging.StreamHandler())
```

### ログにおける機密データ

一部のログには機密データ（たとえばユーザーデータ）が含まれる場合があります。これらを記録しないようにするには、以下の環境変数を設定してください。

LLM への入力および出力の記録を無効化する:

```bash
export OPENAI_AGENTS_DONT_LOG_MODEL_DATA=1
```

ツールへの入力および出力の記録を無効化する:

```bash
export OPENAI_AGENTS_DONT_LOG_TOOL_DATA=1
```