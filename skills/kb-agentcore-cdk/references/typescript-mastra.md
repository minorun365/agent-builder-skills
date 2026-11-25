# TypeScript and Mastra container deployment

## 目次

- TypeScript エージェントのコンテナデプロイ（Mastra等）
  - 必要な依存関係（Mastra 構成）
  - 最小限の agent.ts
  - 動作する Dockerfile（マルチステージ）
  - tsconfig.json（ESM 必須設定）
  - ハマりポイント一覧
    - useradd -u 1000 でビルド失敗
    - agentcore dev は AWS_REGION を渡さない
    - @ai-sdk/amazon-bedrock のクレデンシャル問題
    - モデル ID のプレフィックスと AWS_REGION は揃える
    - bedrock-agentcore@0.2.3 のエラーが不透明な 500 になる
    - リクエストキーは prompt 固定
    - agentcore.json に Python のプレースホルダが残る（Container では無視）
    - agentcore deploy のコンテナビルドは CodeBuild ARM64
  - デバッグの基本動作（TypeScript Container）
- 関連スキル

## TypeScript エージェントのコンテナデプロイ（Mastra等）

agentcore CLI の公式チュートリアルは Python / Strands が前提になっている。ここでは TypeScript + Mastra でコンテナデプロイする際に固有のハマりポイントをまとめる。CLI 全般は上記の「AgentCore CLI」セクションを参照すること。

検証バージョン: `@aws/agentcore` 最新版、`bedrock-agentcore@0.2.3`、Dockerfile 側 Node.js 20、ホスト側 v24

### 必要な依存関係（Mastra 構成）

```sh
# ランタイム依存（dependencies）
npm install zod @mastra/core @ai-sdk/amazon-bedrock @aws-sdk/credential-providers bedrock-agentcore

# 開発依存（devDependencies）
npm install --save-dev typescript @types/node tsx
```

公式チュートリアルは `@strands-agents/sdk` と `@opentelemetry/auto-instrumentations-node` を入れるが、Mastra では不要だ。Dockerfile が `npm ci` を使うため lock ファイルとズレるとビルドエラーになる。`npm install` のたびに `package-lock.json` を必ず更新すること。

### 最小限の agent.ts

```typescript
import { z } from "zod";
import { Agent } from "@mastra/core/agent";
import { createAmazonBedrock } from "@ai-sdk/amazon-bedrock";
import { fromNodeProviderChain } from "@aws-sdk/credential-providers";
import { BedrockAgentCoreApp } from "bedrock-agentcore/runtime";  // ← /runtime サブパス（npm TS版）

const bedrock = createAmazonBedrock({
  credentialProvider: fromNodeProviderChain()  // ← 明示必須（下記「クレデンシャル」参照）
});

const agent = new Agent({
  id: "assistant",
  name: "Assistant",
  instructions: "あなたは親切なアシスタントです。",
  model: bedrock("jp.anthropic.claude-sonnet-4-6"),
});

const app = new BedrockAgentCoreApp({
  invocationHandler: {
    requestSchema: z.object({ prompt: z.string() }),  // ← キーは prompt 固定
    process: async ({ prompt }) => {
      const result = await agent.generate(prompt);
      return result.text;  // ← 文字列で返す（オブジェクトは 500 を誘発）
    },
  },
});
app.run();
```

Python SDK では `from bedrock_agentcore import BedrockAgentCoreApp` を使い、`.runtime` サブモジュールからインポートするとトレースが出なくなる。一方、TypeScript npm 版は `bedrock-agentcore/runtime` サブパスが正しく、言語によって扱いが逆になる点に気をつける。

### 動作する Dockerfile（マルチステージ）

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # = tsc → dist/agent.js を生成

FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
ENV AWS_REGION=ap-northeast-1   # ← リージョンを焼き込む（下記参照）
RUN useradd -m bedrock_agentcore  # ← -u 1000 は付けない（下記参照）
USER bedrock_agentcore
EXPOSE 8080
CMD ["node", "dist/agent.js"]
```

### tsconfig.json（ESM 必須設定）

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": ".",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

`package.json` には `"type": "module"` を設定する必要がある。

### ハマりポイント一覧

#### useradd -u 1000 でビルド失敗

公式ドキュメントの Dockerfile には `useradd -m -u 1000 bedrock_agentcore` と書いてあるが、`node:20-slim` には UID 1000 の `node` ユーザーがすでにある。そのため「UID 1000 is not unique」で `exit code 1` となり、「Container build failed」で落ちる。Python チュートリアルの Dockerfile をそのまま流用したことによるドキュメント側のバグだ。

対処として `-u 1000` を外し、`RUN useradd -m bedrock_agentcore` と書く。

#### agentcore dev は AWS_REGION を渡さない

`agentcore dev` はローカルの AWS クレデンシャルをコンテナへ渡すものの、`AWS_REGION` は渡さず、`~/.aws/config` もコンテナに入らない。その結果 `@ai-sdk/amazon-bedrock` がリージョンを解決できず、次のエラーが出る。

```
AI_LoadSettingError: AWS region setting is missing.
```

対処として Dockerfile の実行ステージに `ENV AWS_REGION=<region>` を設定しておけば、`agent.ts` を変えなくても SDK が自動で読み込む。

#### @ai-sdk/amazon-bedrock のクレデンシャル問題

`@ai-sdk/amazon-bedrock` はデフォルトだと `AWS_ACCESS_KEY_ID` などの環境変数からしかクレデンシャルを読まない。`agentcore dev` は `~/.aws` のマウントと `AWS_CONFIG_FILE` でクレデンシャルを渡すため、デフォルトのままではローカルと本番の両方で `AWS credentials` 系のエラーになる。

対処として、`fromNodeProviderChain()` を `credentialProvider` に渡す。「環境変数 → 設定ファイル → SSO → コンテナロール」の順で解決されるため、ローカルでも本番でも動く。

```typescript
import { fromNodeProviderChain } from "@aws-sdk/credential-providers";
const bedrock = createAmazonBedrock({ credentialProvider: fromNodeProviderChain() });
```

#### モデル ID のプレフィックスと AWS_REGION は揃える

| モデル ID | Dockerfile の ENV AWS_REGION |
|-----------|------------------------------|
| `jp.anthropic.claude-sonnet-4-6` | `ap-northeast-1` |
| `us.anthropic.claude-sonnet-4-6` | `us-east-1` または `us-west-2` |

ここがズレると `ValidationException: inference profile not found` 系のエラーで落ちる。invoke ログで「最初数回 500、最後だけ成功」と出ていたら、たいていこの不一致が原因だ。

#### bedrock-agentcore@0.2.3 のエラーが不透明な 500 になる

ハンドラやバリデーションでエラーが起きると、SDK 内蔵の `@fastify/sse` プラグインと競合して `Attempted to send payload of invalid type 'object'` が発生し、UI には「Error: 500」としか表示されない。

次のように対処する。
- `process` の戻り値は**文字列**にする。オブジェクトを返すとこのバグ経路を踏む。
- エラー診断は UI ではなくログを見る。ローカルは `docker logs <devコンテナ名>`、デプロイ後は `agentcore logs --runtime <名前> --since 30m` を確認する。

#### リクエストキーは prompt 固定

`agentcore invoke "..."` も Inspector の Chat UI も `{"prompt": "..."}` でペイロードを送る。`requestSchema` を `message` などにすると zod バリデーションが失敗し、上記の不透明な 500 になる。

#### agentcore.json に Python のプレースホルダが残る（Container では無視）

`agentcore add agent --build Container --language TypeScript` で生成しても、`agentcore.json` の `runtimes[0].entrypoint` は `"main.py"`、`runtimeVersion` は `"PYTHON_3_14"` のままになる。Container ビルドでは起動が Dockerfile の `CMD`、ランタイムがベースイメージで決まるため、これらの項目は無視される。変更不要で、変えても影響はないため「`main.py` に直さなきゃ？」と焦る必要はない。

#### agentcore deploy のコンテナビルドは CodeBuild ARM64

本番ビルドは CodeBuild の ARM64 環境で走り、ローカルの `docker build` はあくまで事前確認用になる。`node:20-slim` はマルチアーキ対応なのでそのままで問題なく動くが、アーキテクチャ固有のネイティブバイナリを含む場合は注意が必要だ。

### デバッグの基本動作（TypeScript Container）

| 症状 | 見るもの |
|------|---------|
| 「Container build failed」 | コードディレクトリで `docker build .` を直接実行 |
| ローカルでの「Error: 500」 | `docker logs <devコンテナ名>` を確認。`"level":50` の行が真因 |
| デプロイ後の「Received error (500)」 | `agentcore logs --runtime <名前> --since 30m` |
| TypeScript ビルドエラー | `npm run build`（`tsc`）をローカルで実行 |

---

## 関連スキル

- `/kb-strands-agentcore`：Strands Agents フレームワーク（Agent作成、ツール定義、イベント処理）
- `/kb-agentcore-observability`：OpenTelemetry、ログ、メトリクス、トレース
- `/kb-agentcore-identity`：アウトバウンド認証（3LO/M2M/デコレータ分離/callback等）
