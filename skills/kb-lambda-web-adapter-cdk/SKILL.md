---
name: kb-lambda-web-adapter-cdk
description: AIエージェントのWebアプリを「AWS CDK（デプロイはCDKD）＋ Lambda Web Adapter ＋ CloudFront ＋ AgentCore Runtime」で組むときの全体構成。スタックの分け方、画面を配信するLambdaコンテナのDockerfileと配信サーバー、Function URL を CloudFront の OAC で守る設定と追加で要る権限、runtime-config.json で環境値を渡す型、フロントから AgentCore を直接呼ぶ配線、ローカル開発、デプロイ前後の検証。新しくエージェントのWebアプリを作るとき、Amplify Hosting 以外の配信基盤を選ぶとき、CloudFront 経由の Function URL が 403 になるときに読む。
user-invocable: true
---

# Lambda Web Adapter ＋ CDK で組むエージェントWebアプリ

画面の配信と、エージェントの実行を、どちらもコンテナにして CDK で一括管理する構成。
動く実物は [minorun365/marp-agent](https://github.com/minorun365/marp-agent)（`infra/` 配下）。迷ったら実物を読む。

関連スキル：Runtime の細部は `kb-agentcore-cdk`、エージェント本体は `kb-strands-agentcore`、
認証は `kb-demo-app-auth`、ストリーミングの受け方は `kb-frontend-sse`、デプロイコマンドは `cdkd-deploy`。

## 1. 全体像

```
ブラウザ
  │  ① 画面（HTML/JS）と runtime-config.json
  ▼
CloudFront ──OAC(SigV4)──▶ Lambda Function URL（AWS_IAM）
                              └ Lambdaコンテナ：Lambda Web Adapter ＋ Node の配信サーバー
  │
  │  ② Cognito でサインイン（ID/アクセストークンを取得）
  │
  │  ③ エージェント呼び出し（Authorization: Bearer <JWT>、SSE で受信）
  ▼
AgentCore Runtime（コンテナ。JWT オーソライザーが Cognito のトークンを検証）
  └ Strands Agents ＋ Bedrock のモデル
```

- **画面の配信とエージェントの実行は別のコンテナに分ける。** 画面は Lambda、エージェントは AgentCore Runtime
- **エージェントの呼び出しは CloudFront も Lambda も通さない。** ブラウザから AgentCore のエンドポイントへ直接つなぐ。
  Lambda を挟むと、応答サイズと実行時間の上限がエージェントの長い応答にそのまま効いてしまう
- Lambda Web Adapter（以下 LWA）は Lambda の拡張機能。ふつうの HTTP サーバーを、コードを変えずに Lambda で動かせる。
  **「Lambda」と略さない**——ハンドラ関数を書く普通の Lambda とは作りが別物なので、構成図でも LWA と明記する

### この構成を選ぶ理由

- インフラが CDK の1系統にまとまる。認証・エージェント・配信のあいだの値の受け渡しが、スタック間参照だけで済む
- 画面の配信に API の処理を足したくなったとき、同じ HTTP サーバーへルートを足すだけで済む
- Git への push と本番反映が切り離される。反映は明示的にコマンドを打ったときだけ起きる

## 2. スタックの分け方

役割ごとに分け、依存の向きを一方向に保つ。

| スタック | 持つもの | 作り直しの頻度 |
|---|---|---|
| Foundation | ドメイン（Route 53・ACM 証明書）、永続データ、シークレット | ほぼ変えない |
| Access（実行ロール） | Lambda・AgentCore の実行ロール、ロググループ | まれ |
| Auth | Cognito の User Pool・アプリクライアント・ドメイン | まれ |
| Agent | AgentCore Runtime、エージェントのコンテナイメージ | 頻繁 |
| Web | 画面の Lambda コンテナ、Function URL、CloudFront | 頻繁 |

```ts
auth.addStackDependency(foundation);
agent.addStackDependency(auth);      // JWT オーソライザーが User Pool を参照する
web.addStackDependency(agent);       // runtime-config.json が Runtime の ARN を参照する
web.addStackDependency(auth);
```

- **消えると困るもの（データ・ドメイン・利用者）を、頻繁に作り直すもの（Agent・Web）から離す。**
  `removalPolicy: RETAIN` も Foundation 側へ寄せる
- `cdk.Tags.of(app).add('Project', '<名前>')` をアプリ全体へ付ける。コストの切り分けに効く

## 3. 画面を配信する Lambda コンテナ

### Dockerfile

```dockerfile
FROM public.ecr.aws/docker/library/node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM public.ecr.aws/docker/library/node:22-slim
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.1 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY infra/web/server.mjs ./infra/web/server.mjs
ENV PORT=8080 AWS_LWA_PORT=8080 AWS_LWA_READINESS_CHECK_PATH=/health
CMD ["node", "infra/web/server.mjs"]
```

- LWA は **`/opt/extensions/` へ1ファイル置くだけ**で有効になる。アプリ側のコードは Lambda を意識しない
- `AWS_LWA_PORT` は HTTP サーバーが待ち受けるポート、`AWS_LWA_READINESS_CHECK_PATH` は起動確認に使うパス。
  **このパスが 200 を返すまで LWA はリクエストを流さない**ので、配信サーバーに必ず実装する
- ベースイメージは Docker Hub ではなく ECR Public（`public.ecr.aws/docker/library/…`）から取る。
  ビルド環境での取得回数制限を踏まない
- LWA のバージョンは変わるので、使うときに [awslabs/aws-lambda-web-adapter](https://github.com/awslabs/aws-lambda-web-adapter) で確かめる

### 配信サーバーに持たせる5つの役目

Node 標準の `node:http` だけで80行ほど。フレームワークは要らない。

1. **`/health`** … 200 を返すだけ。LWA の起動確認用
2. **`/runtime-config.json`** … 環境変数 `RUNTIME_CONFIG_JSON` の中身をそのまま返す（次節）
3. **静的ファイル** … `dist/` から返す。パスは `normalize` して `..` を落とす
4. **SPA フォールバック** … ファイルが無ければ `index.html` を返す
5. **圧縮とキャッシュヘッダー** … 下の2点

```js
response.setHeader('Cache-Control',
  filePath.endsWith('index.html')
    ? 'no-cache, no-store, must-revalidate'
    : 'public, max-age=31536000, immutable');   // Vite のハッシュつきファイル名が前提
```

⚠️ **圧縮は配信サーバー側でやる。** 応答をストリームで返すと `Content-Length` が付かず、
CloudFront は自動圧縮を諦める。数MBの JS が無圧縮で流れて初期表示が秒単位で遅くなる。
`Accept-Encoding` を見て Brotli か gzip で圧縮し、`Content-Encoding` と `Vary: Accept-Encoding` を付ければ
CloudFront はそのまま通す。品質は Brotli 4・gzip 5 程度で足りる（既定の最高品質は CPU 時間が伸びるだけ）。

## 4. runtime-config.json：環境の値をビルドへ焼き込まない

Cognito の ID や Runtime の ARN を `VITE_*` でビルド時に埋めると、環境ごとにイメージを作り分けることになる。
**CDK が JSON を組み立てて Lambda の環境変数へ入れ、配信サーバーがそれを返し、画面は起動時に取りに行く。**

```ts
const runtimeConfig = cdk.Fn.toJsonString({
  auth: {
    region: this.region,
    userPoolId: props.auth.userPool.userPoolId,
    userPoolClientId: props.auth.userPoolClient.userPoolClientId,
  },
  agent: { runtimeArn: props.agent.runtime.attrAgentRuntimeArn, protocol: 'HTTP' },
  environment: 'production',
});
// → DockerImageFunction の environment: { AWS_LWA_PORT: '8080', RUNTIME_CONFIG_JSON: runtimeConfig }
```

```ts
// 画面側（main.tsx）。Amplify.configure より前に読む
const response = await fetch('/runtime-config.json', { cache: 'no-store' });
if (!response.ok) throw new Error(`runtime-config.json: ${response.status}`);
```

- **秘密の値は入れない。** このJSONは誰でも取得できる。入れてよいのは ID・ARN・URL のような公開前提の値だけ
- CloudFront 側は、このパスだけキャッシュを無効にする（次節）
- ローカル開発では同じ形の `runtime-config.local.json` を置き、`environment` だけ `local` に変えて取り違えを防ぐ

## 5. CDK：Function URL を CloudFront の OAC で守る

```ts
const webFunction = new lambda.DockerImageFunction(this, 'WebFunction', {
  code: lambda.DockerImageCode.fromImageAsset(path.join(currentDir, '../..'), {
    file: 'infra/web/Dockerfile',
    platform: cdk.aws_ecr_assets.Platform.LINUX_ARM64,
    exclude: ['.git', 'node_modules', 'cdk.out', 'dist', 'docs', 'tests', 'agent', '**/__pycache__'],
    ignoreMode: cdk.IgnoreMode.GLOB,
  }),
  architecture: lambda.Architecture.ARM_64,
  memorySize: 1024,
  timeout: cdk.Duration.seconds(30),
  environment: { AWS_LWA_PORT: '8080', RUNTIME_CONFIG_JSON: runtimeConfig },
});

const functionUrl = webFunction.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.AWS_IAM,      // URL を直接叩かれても通さない
  invokeMode: lambda.InvokeMode.BUFFERED,
});
const webOrigin = origins.FunctionUrlOrigin.withOriginAccessControl(functionUrl);
```

⚠️ **OAC だけでは 403 になる。権限がもう1つ要る。** `AWS_IAM` で保護した Function URL は、
`lambda:InvokeFunctionUrl` に加えて `lambda:InvokeFunction` も要求する。
`FunctionUrlOrigin.withOriginAccessControl` が作るのは前者だけなので、後者をディストリビューション単位で足す。

```ts
webFunction.addPermission('CloudFrontInvokeFunction', {
  principal: new iam.ServicePrincipal('cloudfront.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceArn: cdk.Stack.of(this).formatArn({
    service: 'cloudfront', region: '', resource: 'distribution',
    resourceName: distribution.distributionId,
  }),
  invokedViaFunctionUrl: true,
});
```

キャッシュは「オリジンの `Cache-Control` に従う」ポリシーを1つ作り、`runtime-config.json` だけ無効にする。

```ts
const originAwareCache = new cloudfront.CachePolicy(this, 'OriginAwareCachePolicy', {
  minTtl: cdk.Duration.seconds(0),
  defaultTtl: cdk.Duration.seconds(0),
  maxTtl: cdk.Duration.days(365),
  enableAcceptEncodingBrotli: true,
  enableAcceptEncodingGzip: true,       // Accept-Encoding をキャッシュキーへ入れる
});

new cloudfront.Distribution(this, 'Distribution', {
  defaultBehavior: { origin: webOrigin, cachePolicy: originAwareCache, compress: true,
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS },
  additionalBehaviors: {
    'runtime-config.json': { origin: webOrigin,
      cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
      viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS },
  },
});
```

- `exclude` を書かないと、`node_modules` や `.git` がビルドコンテキストへ入ってイメージ作成が極端に遅くなる。
  エージェント側のディレクトリ（`agent/`）も画面のイメージには要らないので外す
- `BUFFERED` は応答が 6MB までという Lambda の上限を受ける。圧縮後に超えるファイルを作らない
  （超えそうなら、その大きな資産だけ S3 オリジンへ逃がす）
- 利用者が上げるファイルや共有用の成果物は、S3 バケットを別オリジンにしてパスで振り分ける

## 6. フロントから AgentCore Runtime を呼ぶ配線

Runtime は Cognito の JWT を検証するオーソライザーで守り、画面からはトークンを付けて直接呼ぶ。

```ts
new agentcore.CfnRuntime(this, 'Runtime', {
  agentRuntimeName: 'my_agent',
  agentRuntimeArtifact: { containerConfiguration: { containerUri: runtimeImage.imageUri } },
  authorizerConfiguration: {
    customJwtAuthorizer: {
      discoveryUrl: `https://cognito-idp.${this.region}.amazonaws.com/${userPoolId}/.well-known/openid-configuration`,
      allowedClients: [userPoolClientId],
    },
  },
  networkConfiguration: { networkMode: 'PUBLIC' },
  protocolConfiguration: 'HTTP',
  requestHeaderConfiguration: { requestHeaderAllowlist: ['Authorization'] },
  roleArn: runtimeRole.roleArn,
});
```

```ts
// 呼び出し先。ARN は URL エンコードする
const url = `https://bedrock-agentcore.${region}.amazonaws.com/runtimes/${encodeURIComponent(runtimeArn)}/invocations?qualifier=DEFAULT`;
```

- ⚠️ **AgentCore は既定でリクエストヘッダーをコンテナへ渡さない。** 利用者ごとの処理（利用統計、利用者別のデータ）が要るなら
  `requestHeaderAllowlist` に `Authorization` を入れる。入れないと、コンテナ側では誰が呼んだのか分からない。
  署名の検証はオーソライザーが済ませているので、コンテナ側は `sub` を読むだけでよい
- ⚠️ **Runtime のロググループは CDK で先に作る。** AgentCore が自動で作るロググループは保持期間が短い。
  `/aws/bedrock-agentcore/runtimes/<RuntimeId>-DEFAULT` を保持期間つきで定義しておく。
  Runtime を作り直すと ID が変わるので、同じスタックに置いて一緒に作り直されるようにする
- エージェントのコンテナも `Platform.LINUX_ARM64` でビルドする（AgentCore Runtime は ARM64）

## 7. ローカル開発

CDKD のローカル実行で、エージェントのコンテナを手元で動かしながら Vite で画面を開発する。

```js
// infra/scripts/start-dev.mjs の要点：2つのプロセスを同時に起動する
start('./node_modules/.bin/cdkd', ['local', 'start-agentcore', '<AgentStack>/Runtime',
  '--watch', '--port', '8081', '--no-verify-auth']);
start('./node_modules/.bin/vite', [], { VITE_AGENT_ENDPOINT: '/local-agent' });
```

- 画面は `VITE_AGENT_ENDPOINT` があるときだけ、呼び出し先をローカルのエージェントへ切り替える（Vite のプロキシで `/local-agent` → `127.0.0.1:8081`）
- 認証は本番と同じ Cognito を使う。アプリクライアントのコールバック URL に `http://localhost:5173/` を入れておく
- CloudFront 相当の配信経路まで手元で再現したいときは、CDKD のローカル実行に CloudFront 用のコマンドがある。
  フラグは変わりやすいので、`cdkd-deploy` スキルの手順で最新のヘルプを確かめてから使う

## 8. npm scripts

デプロイの入口を npm scripts へ固定して、人もエージェントも同じ手順を踏むようにする。

```json
{
  "dev": "node infra/scripts/start-dev.mjs",
  "infra:synth": "cdkd synth",
  "infra:diff": "cdkd diff",
  "infra:dry-run": "cdkd deploy --all --dry-run",
  "infra:deploy": "cdkd deploy --full-wait",
  "infra:drift": "cdkd drift --all"
}
```

反映の順序は **synth → diff → dry-run → deploy**。CDKD はコミュニティ製で、CloudFormation を介さず AWS の API を直接呼ぶ。
本番で使うかどうかの線引きと、既存の CloudFormation スタックへ当ててはいけない理由は `cdkd-deploy` スキルにある。
素の `cdk deploy` でも、この構成はそのまま動く。

## 9. デプロイ後の検証

「デプロイが通った」は動作確認にならない。次を外から確かめる。

```bash
# 画面と設定が返る
curl -sS -o /dev/null -w '%{http_code}\n' https://<配信ドメイン>/
curl -sS https://<配信ドメイン>/runtime-config.json

# 圧縮が効いている（content-encoding: br か gzip）
curl -sS -I -H 'Accept-Encoding: br' https://<配信ドメイン>/assets/<ハッシュつきJS> | grep -i content-encoding

# Function URL を直接叩くと拒否される（403 なら正しい）
curl -sS -o /dev/null -w '%{http_code}\n' https://<id>.lambda-url.<region>.on.aws/

# トークンなしのエージェント呼び出しは拒否される（401/403 なら正しい）
curl -sS -o /dev/null -w '%{http_code}\n' -X POST '<Runtime の invocations URL>'
```

そのうえで、サインインからエージェントの応答が返るまでを、PC とスマホの両方で実際に操作して通す。

## 10. よくある詰まりどころ

| 症状 | 原因と対処 |
|---|---|
| CloudFront 経由で 403 | `lambda:InvokeFunction` の権限が無い（5節）。`invokedViaFunctionUrl: true` を付けて足す |
| 初回アクセスだけ 502・タイムアウト | `AWS_LWA_READINESS_CHECK_PATH` のパスが 200 を返していない、または `AWS_LWA_PORT` とサーバーの待ち受けポートが違う |
| 画面が古いまま | `index.html` に `no-cache` が付いていない。ハッシュつき資産だけを長期キャッシュにする |
| 初期表示が遅い | 圧縮が効いていない（3節）。`content-encoding` を実際に確かめる |
| イメージのビルドが遅い | `exclude` が足りない。`node_modules`・`.git`・`cdk.out` を外す |
| エージェント側で利用者を識別できない | `requestHeaderAllowlist` に `Authorization` が無い（6節） |
| リロードすると 404 | SPA フォールバックが無い。ファイルが無いパスには `index.html` を返す |
