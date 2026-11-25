# Bedrock AgentCore との統合

## Bedrock AgentCore との統合

### 基本構造
```python
from bedrock_agentcore import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()
agent = Agent(model="us.anthropic.claude-sonnet-4-5-20250929-v1:0")

@app.entrypoint
async def invoke(payload):
    prompt = payload.get("prompt", "")
    stream = agent.stream_async(prompt)
    async for event in stream:
        yield event

if __name__ == "__main__":
    app.run()  # ポート8080でリッスン
```

### エンドポイント
- `POST /invocations` はエージェント実行に使う
- `GET /ping` はヘルスチェックに使う

### 必要な依存関係
```
# requirements.txt
bedrock-agentcore
strands-agents
tavily-python  # Web検索が必要な場合
```

注意点として、fastapi/uvicorn は bedrock-agentcore SDK に内包されているため、別途入れなくてよい。

### セッションIDでAgentを管理（複数ユーザー対応）

AgentCoreで複数ユーザーの会話履歴を保持する場合は、セッションIDごとにAgentインスタンスを管理する。

```python
from strands import Agent

# セッションごとのAgentインスタンスを管理
_agent_sessions: dict[str, Agent] = {}

def get_or_create_agent(session_id: str | None) -> Agent:
    """セッションIDに対応するAgentを取得または作成"""
    # セッションIDがない場合は新規Agentを作成（履歴なし）
    if not session_id:
        return Agent(
            model="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
            system_prompt="...",
            tools=[...],
        )

    # 既存のセッションがあればそのAgentを返す
    if session_id in _agent_sessions:
        return _agent_sessions[session_id]

    # 新規セッションの場合はAgentを作成して保存
    agent = Agent(
        model="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
        system_prompt="...",
        tools=[...],
    )
    _agent_sessions[session_id] = agent
    return agent

@app.entrypoint
async def invoke(payload):
    session_id = payload.get("session_id")
    agent = get_or_create_agent(session_id)
    # ...
```

セッションはメモリ上で管理しているため、コンテナを再起動すると消える点に注意する。永続化が必要な場合はDynamoDB等の利用を検討する。

### SSE keep-alive パターン（長時間処理のコネクション維持）

ファイル変換や外部API呼び出し等の同期的な重い処理をSSEで返す場合は、処理中にkeep-aliveイベントを送信してコネクションを維持する。実装には `asyncio.run_in_executor` と `asyncio.shield`、タイムアウトを組み合わせる。

```python
import asyncio

async def _wait_with_keepalive(task, format_name):
    """タスク完了を待ちつつ、5秒ごとにSSE keep-aliveイベントをyield"""
    while not task.done():
        try:
            await asyncio.wait_for(asyncio.shield(task), timeout=5.0)
        except asyncio.TimeoutError:
            yield {"type": "progress", "message": f"{format_name}変換中..."}

@app.entrypoint
async def invoke(payload, context=None):
    if action == "export_pptx" and markdown:
        try:
            print(f"[INFO] PPTX export started")
            loop = asyncio.get_event_loop()
            task = loop.run_in_executor(None, generate_pptx, markdown, theme)
            async for event in _wait_with_keepalive(task, "PPTX"):
                yield event  # 5秒ごとにprogressイベント送信
            result_bytes = task.result()
            yield {"type": "pptx", "data": base64.b64encode(result_bytes).decode()}
        except Exception as e:
            print(f"[ERROR] PPTX export failed: {e}")
            yield {"type": "error", "message": str(e)}
        return
```

実装時のポイントは以下の通り。
- `asyncio.shield(task)` を使っているため、TimeoutError 時もタスクがキャンセルされない
- `task.done()` でループを抜けるタイミングを判定し、`task.result()` で結果を取得する
- フロントエンドのSSEパーサーは未知の `type` を無視するため、既存コードの変更は不要

### ツール使用イベント送信
```python
@app.entrypoint
async def invoke(payload):
    global _generated_markdown
    _generated_markdown = None

    stream = agent.stream_async(payload.get("prompt", ""))
    async for event in stream:
        if "data" in event:
            yield {"type": "text", "data": event["data"]}
        elif "current_tool_use" in event:
            tool_name = event["current_tool_use"].get("name", "unknown")
            yield {"type": "tool_use", "data": tool_name}

    if _generated_markdown:
        yield {"type": "markdown", "data": _generated_markdown}
```

---

## Bedrockプロンプトキャッシュが突然停止する

症状としては、特定のコミット以降、Cost ExplorerのCacheWrite/CacheRead費用がずっと0になる。

原因は、Bedrockのprompt cachingで**ツール定義の合計トークンが1024以上**必要なことにある。ツールのdocstringを短くしたり、`strands_tools.http_request` 等のリッチなビルトインツールをシンプルなカスタムツールに置き換えたりすると、合計が1024を下回ってキャッシュが止まる。

| 状態 | http_requestのトークン数 | ツール合計 | キャッシュ |
|------|------------------------|-----------|----------|
| strands_tools版（21パラメータ） | ~884 tokens | ~1096 | 動作 |
| カスタム版（2パラメータ、docstringが短い） | ~90 tokens | ~302 | 停止 |

次の手順で診断する。
1. Cost Explorerでモデル別にCacheWrite/CacheReadを確認し、0になり始めた日付を特定する
2. その日付のコミットで、`@tool` 付き関数のdocstringが短くなっていないか確認する

解決するには、docstringを充実させて合計を1024トークン以上に戻す。パラメータの詳細説明や使用例、注意事項を追加すると大幅に増やせる。

`@tool` デコレータとトークン数の関係は次の通り。
- Strands Agentsの `@tool` デコレータは、Python関数のdocstringをBedrockへのツール説明（tool spec）として送信する
- パラメータ数やdocstringの文字数が、そのまま毎リクエストのトークンコストになる
- docstringが増えてもキャッシュリード時は90%オフになるため、キャッシュが有効な状態であれば、長いdocstringでもコスト増にはならない

Sonnet 4.6 でのBedrockキャッシュの課金構造は下の表の通り。
| 種別 | 単価 | 備考 |
|------|------|------|
| 通常インプット | $3.00/MTok | キャッシュなし時 |
| キャッシュライト | $3.75/MTok | 初回（25%高い） |
| キャッシュリード | $0.30/MTok | 2回目以降（90%オフ） |

---
