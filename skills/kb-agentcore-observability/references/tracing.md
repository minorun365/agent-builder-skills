# トレースの確認と StrandsTelemetry API

## トレースの確認

1. CloudWatch Console -> Bedrock AgentCore GenAI Observability
2. Agents View / Sessions View / Traces View で確認できる
3. トレースの確認画面は AgentCoreコンソールではなく **CloudWatchコンソール側** にあるので注意する

---

## ローカル開発でのトレース確認（コンソールエクスポーター）

Runtime にデプロイせずにローカルでトレースを確認するには、`StrandsTelemetry` のコンソールエクスポーターを使う。

```python
from strands import Agent
from strands.telemetry import StrandsTelemetry

# トレースのコンソール出力を有効化（1行）
StrandsTelemetry().setup_console_exporter()

agent = Agent()
response = agent("こんにちは")
```

出力されるスパンの階層は次のとおり。
```
invoke_agent Strands Agents    ← Agent Span（エージェント全体）
  └─ execute_event_loop_cycle  ← Cycle Span（推論サイクル）
       ├─ chat                 ← Model Invoke Span（LLM呼び出し）
       └─ execute_tool xxx     ← Tool Span（ツール呼び出し）
```

各スパンには主に次の属性が付与される。
- `gen_ai.usage.prompt_tokens` / `gen_ai.usage.completion_tokens`
- `gen_ai.server.time_to_first_token`（ms）
- `gen_ai.request.model`

注意: ローカルから CloudWatch X-Ray への直接送信は非推奨。ADOT 自動計装で SigV4 署名付き POST が 200 OK で成功しても、Runtime 環境外からのトレースは `aws/spans` に記録されないため、ローカル開発ではコンソールエクスポーターを使うこと。

---

## StrandsTelemetry API

```python
from strands.telemetry import StrandsTelemetry

t = StrandsTelemetry()
t.setup_console_exporter()     # コンソール出力（ローカル開発用）
t.setup_otlp_exporter(**kw)    # OTLPエンドポイントに送信（Langfuse等）
t.setup_meter(                 # メトリクス有効化
    enable_console_exporter=True,
    enable_otlp_exporter=True,
)
# メソッドチェーン可: t.setup_otlp_exporter().setup_console_exporter()
```

`setup_otlp_exporter` の kwargs は `OTLPSpanExporter` にそのまま渡される。
- `endpoint`: OTLPエンドポイントURL
- `headers`: 認証ヘッダーなど

---
