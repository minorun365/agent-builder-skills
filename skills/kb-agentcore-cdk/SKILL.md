---
name: kb-agentcore-cdk
description: AgentCore CDK・デプロイ・ランタイム統合のナレッジ。Runtime作成、JWT認証、SSE、Dockerfile、コンテナ管理、Browser Tool等
user-invocable: true
---
# AgentCore CDK・ランタイム統合ナレッジ

この `SKILL.md` は入口にとどめている。作業内容に近い参照ファイルを1〜2個だけ読み、不要な大型ナレッジはまとめて読まない。

## 参照ルール

> **先に AWS Agent Toolkit を当たる。** この領域は `aws-agents:agents-deploy` / `agents-build`、`aws-core:aws-cdk` がカバーする。
> このスキルは Toolkit の代わりではなく、公式どおりにやったのに動かなかったときの原因をためておく場所として使う。
> Toolkit 側が同じ内容に対応したと分かった項目は、このスキルから削る。両方に置くと公式側しか更新されないため、こちらが古い情報の発生源になる。


1. まず依頼内容から必要な参照ファイルを選ぶ。
2. 最新の仕様や外部サービスの仕様が関係する場合は、参照ファイルだけで断定せず、公式ドキュメント・MCP・実コードで確認する。
3. 複数の領域にまたがる場合も、読み込む参照ファイルは最小限から始める。

## 参照ファイル

- `references/overview.md`: 元の概要・前提
- `references/runtime-integration.md`: Bedrock AgentCore ランタイム統合、SSE、セッション管理
- `references/cdk-runtime.md`: CDK Runtime、JWT、IAM、環境変数、DEFAULTエンドポイント
- `references/container-browser-gateway.md`: Dockerfile、コンテナライフサイクル、Browser Tool、Gateway、WebSocket
- `references/deploy-cli-codezip.md`: update_agent_runtime、SDK/CLI制約、CodeZip、CLI運用
- `references/typescript-mastra.md`: TypeScript/Mastra のコンテナデプロイ例とハマりどころ
