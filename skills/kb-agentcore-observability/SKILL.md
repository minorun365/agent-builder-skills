---
name: kb-agentcore-observability
description: AgentCore の Observability ナレッジ。OpenTelemetry 必須4点セット（OTEL_EXPORTER設定・StrandsTelemetry・traceable デコレータ・Langfuse連携）、EMF メトリクス、Jaeger/コンソールエクスポーターでのトレース確認、トレースが出ない・繋がらない場合のデバッグ手順等
user-invocable: true
---
# AgentCore Observability ナレッジ

この `SKILL.md` は入口の役割に絞っている。作業内容に近い参照ファイルを1〜2個だけ読み、不要な大型ナレッジはまとめて読まない。

## 参照ルール

> **先に AWS Agent Toolkit を当たる。** この領域は `aws-agents:agents-optimize` と `aws-core:aws-observability` がカバーしている。
> このスキルは Toolkit の代わりではなく、公式どおりにやったのに動かなかったときの原因をためておく場所として使う。
> Toolkit 側が同じ内容をカバーしたと分かった項目は、このスキルから削る。両方に置くと公式側だけが更新されるため、こちらが古い情報の発生源になる。


1. まず依頼内容から必要な参照ファイルを選ぶ。
2. 最新仕様や外部サービスの仕様が関係する場合は、参照ファイルだけで断定せず、公式ドキュメント・MCP・実コードで確認する。
3. 複数領域にまたがる場合も、読み込むファイルは最小限から始める。

## 参照ファイル

- `references/overview.md`: 元の概要・前提
- `references/otel-setup.md`: Observability（トレース）のセットアップ、OTELログ形式
- `references/metrics-emf.md`: EMF（Embedded Metric Format）メトリクス
- `references/tracing.md`: トレースの確認、ローカル開発のコンソールエクスポーター、StrandsTelemetry API
- `references/langfuse.md`: Langfuse連携（サードパーティへのOTEL送信）
- `references/calc-and-pdf.md`: LLMにやらせてはいけない計算、FPDF2を使った日本語対応PDFの生成
- `references/troubleshooting.md`: トラブルシューティング、関連スキル、参考リンク
