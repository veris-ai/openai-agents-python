---
search:
  exclude: true
---
# SDK の設定

## API キーとクライアント

デフォルトでは、SDK はインポートされ次第、LLM リクエストとトレーシングのために `OPENAI_API_KEY` 環境変数を参照します。アプリの起動前にその環境変数を設定できない場合は、[set_default_openai_key()][agents.set_default_openai_key] 関数でキーを設定できます。

```python
from agents import set_default_openai_key

set_default_openai_key("sk-...")
```

あるいは、使用する OpenAI クライアントを設定することもできます。デフォルトでは、SDK は環境変数の API キー、または上で設定したデフォルト キーを使って `AsyncOpenAI` インスタンスを作成します。これは [set_default_openai_client()][agents.set_default_openai_client] 関数で変更できます。

```python
from openai import AsyncOpenAI
from agents import set_default_openai_client

custom_client = AsyncOpenAI(base_url="...", api_key="...")
set_default_openai_client(custom_client)
```

最後に、使用する OpenAI API を変更することもできます。デフォルトでは、OpenAI Responses API を使用します。これを [set_default_openai_api()][agents.set_default_openai_api] 関数で上書きして、Chat Completions API を使うようにできます。

```python
from agents import set_default_openai_api

set_default_openai_api("chat_completions")
```

## トレーシング

トレーシングはデフォルトで有効です。既定では、上のセクションの OpenAI API キー（つまり、環境変数またはあなたが設定したデフォルト キー）を使用します。トレーシングに使用する API キーを個別に設定するには、[`set_tracing_export_api_key`][agents.set_tracing_export_api_key] 関数を使用します。

```python
from agents import set_tracing_export_api_key

set_tracing_export_api_key("sk-...")
```

また、[`set_tracing_disabled()`][agents.set_tracing_disabled] 関数でトレーシングを完全に無効化できます。

```python
from agents import set_tracing_disabled

set_tracing_disabled(True)
```

## デバッグ ロギング

SDK には、ハンドラーが設定されていない Python のロガーが 2 つあります。デフォルトでは、警告とエラーは `stdout` に送られますが、その他のログは抑制されます。

詳細なロギングを有効にするには、[`enable_verbose_stdout_logging()`][agents.enable_verbose_stdout_logging] 関数を使用します。

```python
from agents import enable_verbose_stdout_logging

enable_verbose_stdout_logging()
```

また、ハンドラー、フィルター、フォーマッターなどを追加してログをカスタマイズできます。詳しくは [Python のロギングガイド](https://docs.python.org/3/howto/logging.html) を参照してください。

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

### ログ内の機微データ

一部のログには機微データ（例: ユーザー データ）が含まれる場合があります。このデータが記録されないようにするには、次の環境変数を設定してください。

LLM の入力と出力のロギングを無効にするには:

```bash
export OPENAI_AGENTS_DONT_LOG_MODEL_DATA=1
```

ツールの入力と出力のロギングを無効にするには:

```bash
export OPENAI_AGENTS_DONT_LOG_TOOL_DATA=1
```