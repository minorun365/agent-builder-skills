# トラブルシューティング・関連スキル・参考リンク

## トラブルシューティング

### AWS認証エラー
`aws login` で認証した場合、`botocore[crt]` が必要になる。
```bash
uv add 'botocore[crt]'
```

### モデルが見つからない
クロスリージョン推論のモデルID（`us.` プレフィックス）を使っているか確認する。
リージョンによって利用可能なモデルが異なる。

### ストリーミングが動かない
`stream()` と `stream_async()` を環境に合わせて使い分ける。
- 同期コンテキスト -> `stream()`
- 非同期コンテキスト（async/await） -> `stream_async()`

### 複数ツールの連続呼び出しで `MaxTokensReachedException`

1回の依頼で同じ種類のツール（1件ずつ登録するツールなど）を何度も呼ばせる設計だと、モデルが複数の tool use を1つの応答にまとめて生成しようとして出力トークン上限に達し、`MaxTokensReachedException` で応答が途中で切れる。資料本文の要約を渡すなどして tool use の引数が長くなるほど起きやすい。1件のダミー入力では偶然収まってしまい、実データの規模になって初めて発生することがある。

対策は2つをセットで行う。

1. `BedrockModel` の `max_tokens` はデフォルト値が小さいため、明示的に引き上げる。
   ```python
   model = BedrockModel(region_name=REGION, model_id=MODEL_ID, max_tokens=16384)
   ```
2. システムプロンプトで「ツールは1回の応答につき1つだけ呼び、結果を確認してから次を呼ぶ。全部終わったら最後にまとめ返信ツールを呼ぶ」と明示し、1応答につき1ツールになるよう誘導する。

### 検索基盤の同期ラグがある重複チェックはプロンプトでなくコードで

「登録前に既存を検索して重複を防ぐ」という処理を Knowledge Base 検索ツール（`retrieve_*`）で LLM にやらせると、KB の同期が数分〜十数分遅れるため直前に登録したばかりの項目を見逃し、重複したまま登録されてしまう。LLM は指示どおり検索して「重複なし」と正しく判断していても、索引が古いことが原因だ。

整合性が必要なガードはプロンプト任せにせず、**書き込み先（DynamoDB 等）を即時照合するコード**を登録ツール自身に持たせる。人が明示的に指示したときだけバイパスする `allow_duplicate` 相当のフラグを用意しておけば、誤検知が起きたときも回避できる。背後の基盤に結果整合性がある場合、「プロンプトで指示したから安全」は成立しない。

---

## 関連スキル

- `/kb-agentcore-cdk`：AgentCore CDK、デプロイ、ランタイム統合、コンテナ構成
- `/kb-agentcore-observability`：OpenTelemetry、ログ、メトリクス、トレース
- `/kb-agentcore-identity`：アウトバウンド認証（3LO/M2M/デコレータ分離/callback等）

## 参考リンク

- [Strands Agents 公式ドキュメント](https://strandsagents.com/)
- [GitHub リポジトリ](https://github.com/strands-agents/strands-agents)
- [Bedrock AgentCore 統合ガイド](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-agentcore.html)
