---
name: kb-agent-webapp-cdk
description: AIエージェントのWebアプリを「AWS CDK（デプロイはCDKD）＋ S3 ＋ CloudFront ＋ AgentCore Runtime」で組むときの全体構成。画面は S3 から静的に配信し、サーバー側の処理が要るときだけ Lambda Web Adapter（または Lambda Web Functions）を足す判断基準。スタックの分け方、BucketDeployment での配置とキャッシュ指定、SPA の振り分け、runtime-config.json で環境値を渡す型、Lambda を足すときの Function URL と CloudFront OAC の権限、フロントから AgentCore を直接呼ぶ配線、ローカル開発、デプロイ前後の検証。新しくエージェントのWebアプリを作るとき、Amplify Hosting 以外の配信基盤を選ぶとき、画面の配信に Lambda が要るか迷ったとき、CloudFront 経由の Function URL が 403 になるときに読む。
user-invocable: true
---

# CDK で組むエージェントWebアプリ（S3 配信＋AgentCore Runtime）

画面は S3 から静的に配信し、エージェントは AgentCore Runtime で動かし、ブラウザから直接呼ぶ構成。
すべて CDK で一括管理する。サーバー側の処理が要るときだけ、Lambda をパス単位で足す。

関連スキル：Runtime の細部は `kb-agentcore-cdk`、エージェント本体は `kb-strands-agentcore`、
認証は `kb-demo-app-auth`、ストリーミングの受け方は `kb-frontend-sse`、デプロイコマンドは `cdkd-deploy`。

## 1. 全体像

```
ブラウザ
  │  ① 画面（HTML/JS）と runtime-config.json
  ▼
CloudFront ──OAC──▶ S3（Vite のビルド結果と設定 JSON）
  │            └ /api/* など、サーバー処理が要るパスだけ ──OAC──▶ Lambda（6節）
  │
  │  ② Cognito でサインイン（ID/アクセストークンを取得）
  │
  │  ③ エージェント呼び出し（Authorization: Bearer <JWT>、SSE で受信）
  ▼
AgentCore Runtime（コンテナ。JWT オーソライザーが Cognito のトークンを検証）
  └ Strands Agents ＋ Bedrock のモデル
```

- **エージェントの呼び出しは CloudFront も Lambda も通さない。** ブラウザから AgentCore のエンドポイントへ直接つなぐ。
  Lambda を挟むと、応答サイズと実行時間の上限がエージェントの長い応答にそのまま効いてしまう
- **AI の応答待ちが AgentCore 側で起きるなら、画面の配信は静的ファイルで足りる。** React（Vite）の SPA は、
  ビルドすればただのファイルの集まりになる。S3 に置いて CloudFront から返せば、起動待ちも保守するサーバーも無い

### 画面の配信に Lambda が要るかの判定

判定はひとつ、**「リクエストごとに中身が変わる応答を、サーバー側で作る必要があるか」**。

| 要る（6節へ） | 要らない（S3 で足りる） |
|---|---|
| Next.js などの SSR | Vite でビルドした SPA |
| LLM の応答をサーバーで中継・加工する API | ブラウザから AgentCore を直接呼ぶ |
| 決済・外部サービスからの Webhook の受け口 | Cognito の ID や Runtime の ARN を画面へ渡す（デプロイ時に決まる値。4節） |
| 利用者ごとに値が変わる設定を返す | SPA の画面遷移（CloudFront Function で index.html へ振り分ける） |

次の理由は、Lambda で画面を配信する理由にならない。どれも S3 構成で満たせる。

- **「ローカルで本番と同じコンテナを動かせる」** … CDKD のローカル実行は S3 オリジンの CloudFront も再現できる（7節）
- **「環境ごとの設定を実行時に差し込める」** … デプロイ時に決まる値なら `BucketDeployment` が JSON として置ける（4節）
- **「あとからサーバー処理を足しやすい」** … CloudFront のパス振り分けで、必要になった時点で `/api/*` だけ Lambda へ向ければよい

Lambda で静的ファイルを返すと、デプロイのたびのコンテナビルド、配信サーバー（パスの正規化・MIME・SPA の振り分け・圧縮）の保守、
アクセスが途切れたあとの起動待ちを払うことになる。料金はどちらも小さいので、差が出るのは手間と表示の速さ。

### この構成を選ぶ理由

- インフラが CDK の1系統にまとまる。認証・エージェント・配信のあいだの値の受け渡しが、スタック間参照だけで済む
- 画面の更新はファイルのアップロードだけで、コンテナのビルドが無い
- Git への push と本番反映が切り離される。反映は明示的にコマンドを打ったときだけ起きる

## 2. スタックの分け方

役割ごとに分け、依存の向きを一方向に保つ。

| スタック | 持つもの | 作り直しの頻度 |
|---|---|---|
| Foundation | ドメイン（Route 53・ACM 証明書）、永続データ、シークレット | ほぼ変えない |
| Access（実行ロール） | AgentCore（と、足すなら Lambda）の実行ロール、ロググループ | まれ |
| Auth | Cognito の User Pool・アプリクライアント・ドメイン | まれ |
| Agent | AgentCore Runtime、エージェントのコンテナイメージ | 頻繁 |
| Web | 画面用の S3 バケット、CloudFront、（必要なら）API の Lambda | 頻繁 |

```ts
auth.addStackDependency(foundation);
agent.addStackDependency(auth);      // JWT オーソライザーが User Pool を参照する
web.addStackDependency(agent);       // runtime-config.json が Runtime の ARN を参照する
web.addStackDependency(auth);
```

- **消えると困るもの（データ・ドメイン・利用者）を、頻繁に作り直すもの（Agent・Web）から離す。**
  `removalPolicy: RETAIN` も Foundation 側へ寄せる
- `cdk.Tags.of(app).add('Project', '<名前>')` をアプリ全体へ付ける。コストの切り分けに効く

## 3. 画面を S3 から配信する

### バケットと CloudFront

```ts
const siteBucket = new s3.Bucket(this, 'SiteBucket', {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,   // 直接は見せない。CloudFront の OAC だけ通す
  enforceSSL: true,
});
const siteOrigin = origins.S3BucketOrigin.withOriginAccessControl(siteBucket);

// SPA の振り分け：拡張子の無いパス（/settings など）は index.html を返す
const spaFallback = new cloudfront.Function(this, 'SpaFallback', {
  runtime: cloudfront.FunctionRuntime.JS_2_0,
  code: cloudfront.FunctionCode.fromInline(`function handler(event) {
  var request = event.request;
  if (request.uri.indexOf('.') === -1) { request.uri = '/index.html'; }
  return request;
}`),
});

// オリジン（S3 のオブジェクト）の Cache-Control に従うポリシー
const originAwareCache = new cloudfront.CachePolicy(this, 'OriginAwareCachePolicy', {
  minTtl: cdk.Duration.seconds(0),
  defaultTtl: cdk.Duration.seconds(0),
  maxTtl: cdk.Duration.days(365),
  enableAcceptEncodingBrotli: true,
  enableAcceptEncodingGzip: true,
});

const distribution = new cloudfront.Distribution(this, 'Distribution', {
  defaultRootObject: 'index.html',
  defaultBehavior: {
    origin: siteOrigin,
    cachePolicy: originAwareCache,
    compress: true,
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    functionAssociations: [{ eventType: cloudfront.FunctionEventType.VIEWER_REQUEST, function: spaFallback }],
  },
  additionalBehaviors: {
    'runtime-config.json': {
      origin: siteOrigin,
      cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
      viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    },
  },
});
```

- **圧縮は CloudFront に任せる（`compress: true`）。** S3 は `Content-Length` を必ず返すので、CloudFront の自動圧縮がそのまま効く
- SPA の振り分けは、S3 の 403／404 を `index.html` へ差し替える `errorResponses` でも書ける。
  ただし `errorResponses` はディストリビューション全体に効くので、API など他のオリジンの 404 まで書き換わる。パスを見て振り分ける CloudFront Function のほうが安全
- 利用者が上げるファイルや共有用の成果物は、別の S3 バケットを別オリジンにしてパスで振り分ける

### ファイルの配置とキャッシュ指定

Vite のハッシュつきファイルは1年キャッシュ、`index.html` と `runtime-config.json` はキャッシュ禁止。
`BucketDeployment` を2つに分けて、`Cache-Control` を出し分ける。

```ts
// ハッシュつきの資産（index.html 以外のすべて）
new s3deploy.BucketDeployment(this, 'DeployAssets', {
  sources: [s3deploy.Source.asset(distDir, { exclude: ['index.html'] })],
  destinationBucket: siteBucket,
  cacheControl: [s3deploy.CacheControl.fromString('public, max-age=31536000, immutable')],
  prune: false,
});

// 入口の index.html と設定 JSON（4節）
new s3deploy.BucketDeployment(this, 'DeployEntry', {
  sources: [
    s3deploy.Source.asset(distDir, { exclude: ['*', '!index.html'] }),
    s3deploy.Source.jsonData('runtime-config.json', runtimeConfig),
  ],
  destinationBucket: siteBucket,
  cacheControl: [s3deploy.CacheControl.fromString('no-cache, no-store, must-revalidate')],
  prune: false,
  distribution,
  distributionPaths: ['/index.html', '/runtime-config.json'],
});
```

- **同じバケットへ `BucketDeployment` を複数置くときは、どれも `prune: false` にする。** 既定の `true` だと、互いに相手のファイルを消し合う
- `distDir` は CDK の実行前に `npm run build` で作っておく。CDK はビルドしない
- 古いハッシュつきファイルは `prune: false` のため残る。開いたままの古い画面が古い JS を取りに来ても壊れないので、むしろ都合がよい。
  増えすぎが気になるならライフサイクルルールで消す

## 4. runtime-config.json：環境の値をビルドへ焼き込まない

Cognito の ID や Runtime の ARN を `VITE_*` でビルド時に埋めると、環境ごとにビルドし直すことになる。
**CDK が JSON を組み立てて S3 へ置き、画面は起動時に取りに行く。** `Source.jsonData` は、
CDK のトークン（スタックをまたぐ ID や ARN）をデプロイ時に実際の値へ置き換えてくれる。

```ts
const runtimeConfig = {
  auth: {
    region: this.region,
    userPoolId: props.auth.userPool.userPoolId,
    userPoolClientId: props.auth.userPoolClient.userPoolClientId,
  },
  agent: { runtimeArn: props.agent.runtime.attrAgentRuntimeArn, protocol: 'HTTP' },
  environment: 'production',
};
// → 3節の DeployEntry で s3deploy.Source.jsonData('runtime-config.json', runtimeConfig)
```

```ts
// 画面側（main.tsx）。Amplify.configure より前に読む
const response = await fetch('/runtime-config.json', { cache: 'no-store' });
if (!response.ok) throw new Error(`runtime-config.json: ${response.status}`);
```

- **秘密の値は入れない。** このJSONは誰でも取得できる。入れてよいのは ID・ARN・URL のような公開前提の値だけ
- CloudFront 側は、このパスだけキャッシュを無効にする（3節）
- ローカル開発では同じ形の `runtime-config.local.json` を置き、`environment` だけ `local` に変えて取り違えを防ぐ
- 利用者ごと・リクエストごとに変わる値が要るなら、それはサーバー処理なので6節の Lambda で返す

## 5. フロントから AgentCore Runtime を呼ぶ配線

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

- **AgentCore は既定でリクエストヘッダーをコンテナへ渡さない。** 利用者ごとの処理（利用統計、利用者別のデータ）が要るなら
  `requestHeaderAllowlist` に `Authorization` を入れる。入れないと、コンテナ側では誰が呼んだのか分からない。
  署名の検証はオーソライザーが済ませているので、コンテナ側は `sub` を読むだけでよい
- **Runtime のロググループは CDK で先に作る。** AgentCore が自動で作るロググループは保持期間が短い。
  `/aws/bedrock-agentcore/runtimes/<RuntimeId>-DEFAULT` を保持期間つきで定義しておく。
  Runtime を作り直すと ID が変わるので、同じスタックに置いて一緒に作り直されるようにする
- エージェントのコンテナも `Platform.LINUX_ARM64` でビルドする（AgentCore Runtime は ARM64）

## 6. サーバー処理が要るときだけ Lambda を足す

1節の判定で「要る」になったら、**そのパスだけ** Lambda へ振り分ける。画面は S3 に残す。
ふつうの HTTP サーバー（Express・Hono・Next.js など）をそのまま Lambda で動かす手段は2つある。

| 手段 | 中身 | 向いているとき |
|---|---|---|
| Lambda Web Adapter（以下 LWA） | 普通の Lambda に拡張機能を載せ、HTTP をイベントに変換する | 言語・ランタイムを問わず使いたい |
| Lambda Web Functions | ポートで待ち受ける HTTP サーバーをそのまま動かす新しい Lambda の形。1つの実行環境で複数のリクエストを同時に処理し、I/O 待ちの間は CPU 代がかからない | LLM の応答を中継するなど待ち時間が長い Node.js の API。対応ランタイム・リージョン・IaC の対応状況が限られるので、使う前に公式ドキュメントで確かめる |

LWA と書くべきところを「Lambda」と略さない。ハンドラ関数を書く普通の Lambda とは作りが別物なので、構成図でも LWA と明記する。

### LWA のコンテナ

```dockerfile
FROM public.ecr.aws/docker/library/node:22-slim
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.1 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
ENV PORT=8080 AWS_LWA_PORT=8080 AWS_LWA_READINESS_CHECK_PATH=/health
CMD ["node", "server.mjs"]
```

- LWA は **`/opt/extensions/` へ1ファイル置くだけ**で有効になる。アプリ側のコードは Lambda を意識しない
- `AWS_LWA_PORT` は HTTP サーバーが待ち受けるポート、`AWS_LWA_READINESS_CHECK_PATH` は起動確認に使うパス。
  **このパスが 200 を返すまで LWA はリクエストを流さない**ので、サーバーに必ず実装する
- ベースイメージは Docker Hub ではなく ECR Public（`public.ecr.aws/docker/library/…`）から取る。
  ビルド環境での取得回数制限を踏まない
- LWA のバージョンは変わるので、使うときに [awslabs/aws-lambda-web-adapter](https://github.com/awslabs/aws-lambda-web-adapter) で確かめる
- ビルドコンテキストは API のディレクトリ（下の例では `api/`）だけにする。リポジトリ全体を渡すと `node_modules` や `.git` が入り、イメージ作成が極端に遅くなる

### Function URL を CloudFront の OAC で守る

```ts
const apiFunction = new lambda.DockerImageFunction(this, 'ApiFunction', {
  code: lambda.DockerImageCode.fromImageAsset(path.join(currentDir, '../../api'), {
    platform: cdk.aws_ecr_assets.Platform.LINUX_ARM64,
  }),
  architecture: lambda.Architecture.ARM_64,
  memorySize: 1024,
  timeout: cdk.Duration.seconds(30),
  environment: { AWS_LWA_PORT: '8080' },
});

const functionUrl = apiFunction.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.AWS_IAM,      // URL を直接叩かれても通さない
  invokeMode: lambda.InvokeMode.BUFFERED,
});
const apiOrigin = origins.FunctionUrlOrigin.withOriginAccessControl(functionUrl);

// 3節の Distribution の additionalBehaviors に足す
// 'api/*': { origin: apiOrigin, cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
//   allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
//   originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER,
//   viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS },
```

**OAC だけでは 403 になる。権限がもう1つ要る。** `AWS_IAM` で保護した Function URL は、
`lambda:InvokeFunctionUrl` に加えて `lambda:InvokeFunction` も要求する。
`FunctionUrlOrigin.withOriginAccessControl` が作るのは前者だけなので、後者をディストリビューション単位で足す。

```ts
apiFunction.addPermission('CloudFrontInvokeFunction', {
  principal: new iam.ServicePrincipal('cloudfront.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceArn: cdk.Stack.of(this).formatArn({
    service: 'cloudfront', region: '', resource: 'distribution',
    resourceName: distribution.distributionId,
  }),
  invokedViaFunctionUrl: true,
});
```

- **OAC で守った Function URL へ POST・PUT を送るときは、ブラウザ側で本文の SHA-256 を計算して `x-amz-content-sha256` ヘッダーに付ける。**
  付けないと署名が合わず 403 になる（Lambda は署名なしの本文を受け付けない）。GET だけなら不要。
  Lambda Web Functions をオリジンにした場合はこのヘッダーは要らない
- `BUFFERED` は応答が 6MB までという Lambda の上限を受ける
- エージェントの呼び出しをこの Lambda に中継させない（1節）。中継が要るのは、応答をサーバー側で加工・記録する必要があるときだけ

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
- CloudFront 相当の配信経路まで手元で再現したいときは、CDKD のローカル実行に CloudFront 用のコマンドがある（S3 と Lambda Function URL のオリジン、CloudFront Function を扱える）。
  フラグは変わりやすいので、`cdkd-deploy` スキルの手順で最新のヘルプを確かめてから使う

## 8. npm scripts

デプロイの入口を npm scripts へ固定して、人もエージェントも同じ手順を踏むようにする。
画面のビルドを `infra:*` の前に挟み、古い `dist/` をデプロイしないようにする。

```json
{
  "dev": "node infra/scripts/start-dev.mjs",
  "infra:synth": "npm run build && cdkd synth",
  "infra:diff": "npm run build && cdkd diff",
  "infra:dry-run": "npm run build && cdkd deploy --all --dry-run",
  "infra:deploy": "npm run build && cdkd deploy --full-wait",
  "infra:drift": "cdkd drift --all"
}
```

反映の順序は **synth → diff → dry-run → deploy**。CDKD はコミュニティ製で、CloudFormation を介さず AWS の API を直接呼ぶ。
本番で使うかどうかの線引きと、既存の CloudFormation スタックへ当ててはいけない理由は `cdkd-deploy` スキルにある。
素の `cdk deploy` でも、この構成はそのまま動く。

## 9. デプロイ後の検証

「デプロイが通った」は動作確認にならない。次を外から確かめる。

```bash
# 画面と設定が返る。SPA の画面遷移先も 200
curl -sS -o /dev/null -w '%{http_code}\n' https://<配信ドメイン>/
curl -sS -o /dev/null -w '%{http_code}\n' https://<配信ドメイン>/settings
curl -sS https://<配信ドメイン>/runtime-config.json

# キャッシュ指定（index.html は no-cache、ハッシュつき資産は immutable）
curl -sS -I https://<配信ドメイン>/ | grep -i cache-control
curl -sS -I https://<配信ドメイン>/assets/<ハッシュつきJS> | grep -i cache-control

# 圧縮が効いている（content-encoding: br か gzip）
curl -sS -I -H 'Accept-Encoding: br' https://<配信ドメイン>/assets/<ハッシュつきJS> | grep -i content-encoding

# バケットを直接は見られない（403 なら正しい）
curl -sS -o /dev/null -w '%{http_code}\n' https://<バケット名>.s3.amazonaws.com/index.html

# トークンなしのエージェント呼び出しは拒否される（401/403 なら正しい）
curl -sS -o /dev/null -w '%{http_code}\n' -X POST '<Runtime の invocations URL>'
```

Lambda を足した場合は、Function URL を直接叩くと 403 になることも確かめる。
そのうえで、サインインからエージェントの応答が返るまでを、PC とスマホの両方で実際に操作して通す。

## 10. よくある詰まりどころ

| 症状 | 原因と対処 |
|---|---|
| リロードすると 403 か 404 | SPA の振り分けが無い。拡張子の無いパスを `index.html` へ向ける CloudFront Function を付ける（3節） |
| デプロイしたのに画面が古いまま | `index.html` に `no-cache` が付いていない、または `distributionPaths` で `index.html` を無効化していない |
| デプロイのたびにファイルが消える | 同じバケットへの `BucketDeployment` が `prune: true` のまま（3節） |
| 古いビルドがデプロイされる | `dist/` を作り直していない。`infra:*` の前に `npm run build` を挟む（8節） |
| エージェント側で利用者を識別できない | `requestHeaderAllowlist` に `Authorization` が無い（5節） |
| Lambda を足したら CloudFront 経由で 403 | `lambda:InvokeFunction` の権限が無い（6節）。`invokedViaFunctionUrl: true` を付けて足す。POST・PUT なら `x-amz-content-sha256` も確かめる |
| Lambda の初回アクセスだけ 502・タイムアウト | `AWS_LWA_READINESS_CHECK_PATH` のパスが 200 を返していない、または `AWS_LWA_PORT` とサーバーの待ち受けポートが違う |
