# EMF メトリクス

## EMF（Embedded Metric Format）メトリクス

Strands Agents はトークンメトリクスを EMF 形式で自動出力する。ログストリーム `otel-rt-logs` に JSON 形式で記録される。

メトリクス名の一覧は次のとおり。
- `strands.event_loop.input.tokens` / `strands.event_loop.output.tokens`
- `strands.event_loop.cache_read.input.tokens` / `strands.event_loop.cache_write.input.tokens`
- `strands.event_loop.latency` / `strands.model.time_to_first_token`

EMF のデータはヒストグラム形式で記録される。

```json
{
  "strands.event_loop.input.tokens": {
    "Values": [576.81, 2571.17, 7272.37],
    "Counts": [1.0, 1.0, 1.0],
    "Count": 3,
    "Sum": 10344,
    "Max": 7197,
    "Min": 582
  }
}
```

- `Count`: 1回のエージェント呼び出しでのモデルコール回数（≒ターン数）
- `Sum`: セッションの合計トークン数
- `Max`/`Min`: 単一ターンの最大値と最小値
- `Values`: ヒストグラムのバケット境界による近似値。**正確な値は Sum/Max/Min を使う**

CloudWatch Log Insights で分析するときは、次のように検索する。

```
# EMFログの検索
filter @message like /strands.event_loop.input.tokens/
| fields @message, @timestamp
| limit 200
```

Log Insights の `parse` + `stats pct()` では EMF ヒストグラムの値を正しく集計できない点に注意する。Python などで JSON をパースして `Sum`/`Max`/`Min` を直接抽出する方が正確だ。

---
