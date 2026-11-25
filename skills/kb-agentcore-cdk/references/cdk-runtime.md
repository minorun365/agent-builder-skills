# AgentCore CDK runtime patterns

## 目次

- AgentCore CDK
  - Runtime作成（推奨パターン）
  - JWT認証（Cognito統合）
    - DiscoveryUrl には `/.well-known/openid-configuration` が必須
    - allowedClients は client_id クレームを検証
  - IAM権限（Bedrockモデル呼び出し）
  - 環境変数渡し
  - リクエストヘッダーは許可リストに入れないとコンテナへ届かない（Authorization も既定では渡されない）
  - DEFAULTエンドポイント
  - SSEストリーミング
  - JWT認証時はHTTPS直接呼び出し（SDK非対応）

## AgentCore CDK

### Runtime作成（推奨パターン）

```typescript
import * as agentcore from '@aws-cdk/aws-bedrock-agentcore-alpha';

const artifact = agentcore.AgentRuntimeArtifact.fromAsset(
  path.join(__dirname, 'runtime')
);

const runtime = new agentcore.Runtime(stack, 'MyRuntime', {
  runtimeName: 'my-agent',
  agentRuntimeArtifact: artifact,
  authorizerConfiguration: agentcore.RuntimeAuthorizerConfiguration.usingJWT(
    discoveryUrl,
    [clientId],  // allowedClients - client_idクレームを検証
  ),
});

// エンドポイントはDEFAULTを使用（addEndpoint不要）
```

### JWT認証（Cognito統合）

#### DiscoveryUrl には `/.well-known/openid-configuration` が必須

AgentCore の `usingJWT` に渡す `discoveryUrl` は、末尾に `/.well-known/openid-configuration` を含める必要がある。`https://cognito-idp.us-east-1.amazonaws.com/{userPoolId}` のような issuer URL だけを渡すとバリデーションエラーになる。

```typescript
// NG: issuer URL のみ -> CFn バリデーションエラー
const discoveryUrl = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;

// OK: フルパス
const discoveryUrl = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/openid-configuration`;
```

エラーメッセージ: `DiscoveryUrl: string [...] does not match pattern ^.+/\.well-known/openid-configuration$`

#### allowedClients は client_id クレームを検証

AgentCore Runtime の JWT 認証では、`usingJWT` の `allowedClients` が `client_id` クレームを検証する。

| トークン種別 | クライアントIDの格納先 | AgentCore認証 |
|-------------|---------------------|--------------|
| IDトークン | `aud` クレーム | NG |
| アクセストークン | `client_id` クレーム | OK |

このため、Cognito と AgentCore を連携させるときはアクセストークンを使う。

```typescript
// フロントエンドでの実装例
const session = await fetchAuthSession();
const accessToken = session.tokens?.accessToken?.toString();  // IDトークンではなくアクセストークン
```

### IAM権限（Bedrockモデル呼び出し）

`us.anthropic.claude-*` 形式のモデルIDでクロスリージョン推論を使うときは、次の2種類のリソース両方に権限を付ける。

```typescript
runtime.addToRolePolicy(new iam.PolicyStatement({
  actions: [
    'bedrock:InvokeModel',
    'bedrock:InvokeModelWithResponseStream',
  ],
  resources: [
    'arn:aws:bedrock:*::foundation-model/*',      // 基盤モデル
    'arn:aws:bedrock:*:*:inference-profile/*',    // 推論プロファイル（クロスリージョン推論）
  ],
}));
```

`foundation-model/*` だけでは `AccessDeniedException` が発生する。

### 環境変数渡し

```typescript
const runtime = new agentcore.Runtime(stack, 'MyRuntime', {
  runtimeName: 'my-agent',
  agentRuntimeArtifact: artifact,
  environmentVariables: {
    TAVILY_API_KEY: process.env.TAVILY_API_KEY || '',
    OTHER_SECRET: process.env.OTHER_SECRET || '',
  },
});
```

sandbox 起動時に環境変数を設定する必要がある。
```bash
export TAVILY_API_KEY=$(grep TAVILY_API_KEY .env | cut -d= -f2) && npx ampx sandbox
```

### リクエストヘッダーは許可リストに入れないとコンテナへ届かない

**AgentCore Runtime は、既定でどのリクエストヘッダーもコンテナへ渡さない。** `Authorization` も例外ではない。
JWT オーソライザーによる検証は AgentCore 側で完結するため、トークンはエージェントコードへ渡されない。
渡したいヘッダーは `requestHeaderConfiguration.requestHeaderAllowlist` に明示する。

```typescript
new agentcore.CfnRuntime(this, 'Runtime', {
  authorizerConfiguration: { customJwtAuthorizer: { discoveryUrl, allowedClients } },
  // これが無いと context.request_headers が空になる
  requestHeaderConfiguration: { requestHeaderAllowlist: ['Authorization'] },
  // ...
});
```

エラーにならないのが厄介で、`context.request_headers` が `None` になる。利用者識別やテナント判定を JWT クレームから行っていると、例外も警告も出ないまま統計だけが空になる。

- ローカル実行では再現しない。`app.run()` で SDK が直接 HTTP を受けるときは許可リストの制御が働かないため、ヘッダーはそのまま渡される。「ローカルで動いたから実装は正しい」は本番の根拠にならない。切り分けには、ローカルで受信ヘッダーのキー名を出す最小アプリを立てて、本番のログと突き合わせる
- AgentCore 側で検証済みのため、署名の再検証は不要。`jwt.decode(token, options={"verify_signature": False})` でクレームだけを取り出す。公式ドキュメントもこの前提で書かれている
- 設定が落ちたことを検知できるテストを1本置く。CDK のソースに `requestHeaderAllowlist` と `'Authorization'` が含まれるかを静的検査するテストで足りる。壊れても無症状なので、テストが無いと次に気づくのは「統計を見ようとしたとき」になる

> 実装もフロントのヘッダー送信も正しいのに、記録した利用者IDが全件「識別できず」になっていたら、Runtime 側の許可リスト未設定を疑う。
> 公式ドキュメントは「Pass custom headers to AgentCore Runtime」および runtime-oauth.html の Step 7 が該当する。

### DEFAULTエンドポイント

Runtime を作成すると DEFAULT エンドポイントが自動で作成される。特別な理由がなければ `addEndpoint()` は不要。

```typescript
// NG: 不要なエンドポイントが増える
const endpoint = runtime.addEndpoint('my-endpoint');  // DEFAULT + my-endpoint の2つになる

// OK: DEFAULTエンドポイントを使う
// addEndpoint() を呼ばない -> DEFAULTのみ
```

### SSEストリーミング

エンドポイント URL の形式は次のとおり。
```
POST https://bedrock-agentcore.{region}.amazonaws.com/runtimes/{URLエンコードARN}/invocations?qualifier={endpointName}
```

ARN は必ず `encodeURIComponent()` で完全に URL エンコードする。

### JWT認証時はHTTPS直接呼び出し（SDK非対応）

customJWTAuthorizer を設定したランタイムに AWS SDK / CLI の `invoke_agent_runtime` を使うと `Authorization method mismatch` エラーになる。HTTPS エンドポイントへ直接リクエストを送る。

```typescript
// フロントエンドでの呼び出し例
const session = await fetchAuthSession();
const token = session.tokens?.accessToken?.toString();
const region = AGENT_ARN.split(':')[3];
const url = `https://bedrock-agentcore.${region}.amazonaws.com/runtimes/${encodeURIComponent(AGENT_ARN)}/invocations?qualifier=DEFAULT`;

const res = await fetch(url, {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
    'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': sessionId,  // 33文字以上必須
  },
  body: JSON.stringify({ prompt: userText, session_id: sessionId }),
});
```

制約は次の2点。
- `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` は 33-256 文字にする必要がある。短いとバリデーションエラーになる
- JWT を使うときは IAM SigV4 認証と併用できず、どちらか一方を選ぶ

レスポンス形式は次のとおり。
```
data: {"type": "text", "data": "テキストチャンク"}
data: {"type": "tool_use", "data": "ツール名"}
data: {"type": "markdown", "data": "生成されたコンテンツ"}
data: {"type": "error", "error": "エラーメッセージ"}
data: [DONE]
```

イベントペイロードは `content` または `data` フィールドに入るため、両方に対応する。
```typescript
const textValue = event.content || event.data;
```

### Runtime を V2 へ移すときに確かめること

- CDKD は Runtime の `platformVersion: V2` を Cloud Control API 経由で適用する。デプロイの成功だけで済ませず、GetAgentRuntime で `platformVersion` を読み、API やフロントエンドといった公開入口がどの Runtime を指しているかも照合する
- AgentCore Memory に、会話の控え（blob）と conversational イベントを混ぜて保存している場合、最新のイベント1件だけを読むと控えを取りこぼす。保存形式を判別し、全ページを読んで最新の控えを選ぶ。セッションを閉じたあとの会話の復元を、ログインあり・なしの両方の経路で確かめる
- Managed Knowledge Base を Retrieve するときは `managedSearchConfiguration` を使う。通常の `vectorSearchConfiguration` を渡すと ValidationException になる

---
