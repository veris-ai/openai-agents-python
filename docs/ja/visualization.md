---
search:
  exclude: true
---
# エージェント可視化

エージェントの可視化を使用すると、 **Graphviz** を利用してエージェントとその関係を構造化されたグラフィカル表現として生成できます。これは、アプリケーション内でエージェント、ツール、およびハンドオフがどのように相互作用するかを理解するのに役立ちます。

## インストール

オプションの `viz` 依存グループをインストールします:

```bash
pip install "openai-agents[viz]"
```

## グラフの生成

`draw_graph` 関数を使用してエージェントの可視化を生成できます。この関数は以下のような有向グラフを作成します:

- **エージェント** は黄色のボックスで表されます。
- **MCP サーバー** は灰色のボックスで表されます。
- **ツール** は緑色の楕円で表されます。
- **ハンドオフ** は一方のエージェントから別のエージェントへ向かう有向エッジで表されます。

### 使用例

```python
import os

from agents import Agent, function_tool
from agents.mcp.server import MCPServerStdio
from agents.extensions.visualization import draw_graph

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You only speak Spanish.",
)

english_agent = Agent(
    name="English agent",
    instructions="You only speak English",
)

current_dir = os.path.dirname(os.path.abspath(__file__))
samples_dir = os.path.join(current_dir, "sample_files")
mcp_server = MCPServerStdio(
    name="Filesystem Server, via npx",
    params={
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", samples_dir],
    },
)

triage_agent = Agent(
    name="Triage agent",
    instructions="Handoff to the appropriate agent based on the language of the request.",
    handoffs=[spanish_agent, english_agent],
    tools=[get_weather],
    mcp_servers=[mcp_server],
)

draw_graph(triage_agent)
```

![Agent Graph](../assets/images/graph.png)

これにより、 **triage エージェント** の構造とそのサブエージェントおよびツールへの接続を視覚的に表すグラフが生成されます。

## 可視化の理解

生成されたグラフには以下が含まれます:

- エントリーポイントを示す **開始ノード** (`__start__`)
- 黄色で塗りつぶされた **長方形** で表されるエージェント
- 緑色で塗りつぶされた **楕円** で表されるツール
- 灰色で塗りつぶされた **長方形** で表される **MCP サーバー**
- 相互作用を示す有向エッジ  
  - **実線の矢印** はエージェント間のハンドオフ  
  - **点線の矢印** はツール呼び出し  
  - **破線の矢印** は MCP サーバー呼び出し
- 実行の終了地点を示す **終了ノード** (`__end__`)

## グラフのカスタマイズ

### グラフの表示
既定では、`draw_graph` はグラフをインラインで表示します。別ウィンドウで表示したい場合は、次のようにします:

```python
draw_graph(triage_agent).view()
```

### グラフの保存
既定では、`draw_graph` はグラフをインラインで表示します。ファイルとして保存するには、ファイル名を指定します:

```python
draw_graph(triage_agent, filename="agent_graph")
```

これにより、作業ディレクトリに `agent_graph.png` が生成されます。