---
name: kb-agentcore-identity
description: AgentCore Identity（アウトバウンド認証）のナレッジ。3LO/M2M/デコレータ分離/callback等
user-invocable: true
---

# AgentCore Identity（アウトバウンド認証）

> **先に AWS Agent Toolkit を当たる。** この領域はアウトバウンド認証を担当する `aws-agents:agents-connect` がカバーする。
> このスキルは Toolkit の代わりではなく、公式どおりにやったのに動かなかったときの原因をためておく場所として使う。
> Toolkit 側が同じ内容をカバーしたと分かった項目は、このスキルから削る。両方に置くと更新されるのは公式側だけになり、こちらが古い情報の発生源になる。


Bedrock AgentCore Identity を使った外部サービスとの OAuth2 連携パターンを記録する。

## `@requires_access_token` と `@tool` は必ず分離する

`@tool` と `@requires_access_token` を同じ関数に重ねると、パラメータ解析が干渉してツールが正しく動作しない。エージェントが `access_token` を入力パラメータとして要求してしまう。

```python
# ❌ NG: デコレータをスタック → パラメータ干渉
@tool
@requires_access_token(provider_name="my-provider", scopes=[...], auth_flow="USER_FEDERATION")
async def get_pages(*, access_token: str):
    ...

# ✅ OK: トークン取得とツールを分離
@requires_access_token(provider_name="my-provider", scopes=[...], auth_flow="USER_FEDERATION",
    on_auth_url=lambda url: print(f"認可URL: {url}"),
    callback_url="http://localhost:9090/oauth2/callback")
def get_token(access_token: str = ""):
    return access_token

@tool
def get_pages():
    """外部APIのページ一覧を取得する"""
    token = get_token()
    response = httpx.get("https://api.example.com/pages",
        headers={"Authorization": f"Bearer {token}"})
    return response.json()
```

### 内部関数パターン（推奨：よりコンパクト）

トークン取得関数をツール内部に定義すると、ファイルを分けずに済む。公式サンプルの `01_outbound.py` もこの方式をとっている。

```python
@tool
def add_calendar_event(
    summary: str,
    start_datetime: str,
    end_datetime: str,
    description: str = "",
):
    """Google Calendarに予定を追加する。"""

    @requires_access_token(
        provider_name=PROVIDER_NAME,
        scopes=["https://www.googleapis.com/auth/calendar"],
        auth_flow="USER_FEDERATION",
        on_auth_url=lambda url: print(f"認証URL: {url}"),
        callback_url="http://localhost:9090/oauth2/callback",
    )
    def call_api(access_token: str = ""):
        event = {
            "summary": summary,
            "start": {"dateTime": start_datetime, "timeZone": "Asia/Tokyo"},
            "end": {"dateTime": end_datetime, "timeZone": "Asia/Tokyo"},
        }
        resp = requests.post(
            "https://www.googleapis.com/calendar/v3/calendars/primary/events",
            headers={"Authorization": f"Bearer {access_token}"},
            json=event,
        )
        return resp.json()

    result = call_api()
    if "error" in result:
        return f"失敗: {result['error']['message']}"
    return f"登録完了: {result.get('summary')}"
```

外側の `@tool` が認識するスキーマには `access_token` が含まれないため、LLM がユーザーにトークンを直接要求する問題が解消される。

## USER_FEDERATION は同期関数でも動作する

`auth_flow="USER_FEDERATION"` でも同期関数の `def` で定義できる。SDK 内部でポーリングをブロッキング実行する。`async def` と `asyncio.run_until_complete()` の組み合わせは、BedrockAgentCoreApp の async コンテキストと競合するため避ける。

## 3LO (USER_FEDERATION) フローにはローカル callback サーバーが必要

3LO フローの callback は2段階で動作する。

1. Atlassian 等の IdP → AgentCore callback（`/identities/oauth2/callback/{providerId}?code=XXX&state=YYY`）→ code exchange
2. AgentCore → アプリの callback サーバー（`?session_id=...`）→ `CompleteResourceTokenAuth` を呼ぶ

`callback_url` に AgentCore の callback endpoint 自体を指定するとバリデーションエラーになる（GitHub Issue #801）。アプリ側で callback サーバーを立てる必要がある。

```python
# callback_server.py（FastAPI で localhost:9090 に立てる）
from fastapi import FastAPI
from bedrock_agentcore.services.identity import IdentityClient, UserIdIdentifier

app = FastAPI()
identity_client = IdentityClient(region="us-east-1")

@app.get("/oauth2/callback")
async def handle_callback(session_id: str):
    identity_client.complete_resource_token_auth(
        session_uri=session_id,
        user_identifier=UserIdIdentifier(user_id="<user_id>"),
    )
    return {"status": "success"}
```

```python
# エージェント側：callback_url を localhost に向ける
@requires_access_token(
    provider_name="my-provider",
    scopes=["read:confluence-content.all", "offline_access"],
    auth_flow="USER_FEDERATION",
    on_auth_url=lambda url: print(f"BROWSER: {url}"),
    callback_url="http://localhost:9090/oauth2/callback",
)
def get_token(access_token: str = ""):
    return access_token
```

セットアップ手順は以下のとおり。
1. Workload Identity の `allowedResourceOauth2ReturnUrls` に `http://localhost:9090/oauth2/callback` を追加する
2. `uv run python callback_server.py` で callback サーバーを起動する
3. エージェントを実行し、auth URL をブラウザで開いて認可する
4. callback サーバーが `CompleteResourceTokenAuth` を呼び、ポーリングでトークンを取得する

公式サンプル `awslabs/amazon-bedrock-agentcore-samples` の `05-Outbound_Auth_3lo/oauth2_callback_server.py` も参考になる。

## M2M (client_credentials) フローの互換性問題

AgentCore Identity の M2M フローは標準の `grant_type=client_credentials` を使うが、多くの外部プロバイダーが独自仕様を持つため互換性に注意がいる。

| プロバイダー | M2M結果 | 原因 |
|-------------|--------|------|
| Zoom | ❌ トークン取得成功、API拒否 | `grant_type=account_credentials` + `account_id` が必要 |
| GitHub | ❌ トークン取得失敗 | Token endpoint レスポンス形式が非標準 |
| Atlassian | △ サイトアクセス権なし | サービスアカウント設定が別途必要 |

実務上 M2M はイレギュラーであり、3LO (USER_FEDERATION) が主流になる。

## BedrockAgentCoreApp の初期化タイムアウト

AgentCore Runtime は初期化を30秒以内に完了する必要がある。モジュールレベルで `agent(...)` を実行すると、3LO のポーリングで初期化がブロックされてタイムアウトする。

```python
# ❌ NG: モジュールレベルで実行 → 30秒タイムアウト
agent = Agent(tools=[my_tool])
agent("処理を実行して")

# ✅ OK: BedrockAgentCoreApp + @app.entrypoint で invoke 時に実行
app = BedrockAgentCoreApp()

@app.entrypoint
def handle_request(request, context=None):
    agent = Agent(tools=[my_tool])
    result = agent(request.get("prompt", ""))
    return {"response": str(result)}

if __name__ == "__main__":
    app.run()
```

## 保管庫は「範囲」と「追加の指定」の組で鍵を引く

`GetResourceOauth2Token` は、`scopes` だけでなく `customParameters`（Google の `access_type`・`prompt`・`login_hint` など）まで、許可を取ったときと同じ頼み方でないと保管庫の鍵を返さない。1つでも違うと、鍵があっても `authorizationUrl` を返してくる。許可は通り `CompleteResourceTokenAuth` も成功しているのに取り出せない、という形で出る。

- 許可を取るスクリプトとエージェントで、`scopes` と `customParameters` を1か所の定数から読む
- Google は `access_type=offline` に加えて `prompt=consent` も入れる。入れないと2回目以降の許可で更新用トークンが返らない
- 範囲を広げたら、定数を変えてから許可を取り直す

## 利用者が1人のエージェントは、許可の戻り先を手元のマシンに置ける

鍵の持ち主が開発者本人1人だけのエージェントなら、公開の callback サーバーを立てなくてよい。手元で名前を決めて作ったワークロード ID と固定の userId を使って `GetWorkloadAccessTokenForUserId` を呼び、`GetResourceOauth2Token(forceAuthentication=True)` を経て `localhost` の待ち受けで `CompleteResourceTokenAuth` を1回実行すれば、以後はエージェントが同じワークロード ID と userId で鍵を受け取れる。

- ランタイムに自動で付くワークロード ID は使わず、名前を決めて作る。そうすれば許可スクリプトとエージェントが同じ名前を指せる。実行ロールには `GetWorkloadAccessTokenForUserId` と `GetResourceOauth2Token`（ワークロード ID・token-vault・発行元）、発行元の秘密 `bedrock-agentcore-identity!default/oauth2/<名前>-*` に対する `secretsmanager:GetSecretValue` を許可する
- 発行元の `CreateOauth2CredentialProvider` は client secret を持つため、テンプレートではなくスクリプトで作る。callback URL は作成後にしか分からないので、OAuth クライアント作成、発行元作成、クライアントへのリダイレクト URI 登録の順に進める。Google のデスクトップ型クライアントは任意のリダイレクト URI を登録できないため、ウェブ型を作る
- 許可の URL は1つのタブだけで開く。既定のブラウザを自動で開くと、別のプロファイルでも開いて片方が `Invalid request` になる

## Gateway を挟むか、コードの道具で直接使うか

Gateway の OpenAPI / MCP ターゲットに 3LO の発行元を付けると、API をそのままツールとして出せるため、新しい SaaS を足すのが設定だけで済む。ただし API の生の応答がモデルへ渡る。個人宛ての本文や個人情報を含むデータのようにモデルへ渡す項目を絞りたいものは、Identity から鍵だけ受け取るコードのツールにして、抜き出しをコードで行う。タスク管理やドキュメント検索のように生の応答を渡してよい SaaS が増えてきたら Gateway に寄せる。

## トラブルシューティング

### workload-identity ARN 不一致

症状: `@requires_access_token` で `GetResourceOauth2Token` の権限エラーが出る。IAMポリシーには `GetResourceOauth2Token` を設定済み。

原因: `agentcore deploy` で `.agentcore.json` がパッケージに含まれると、ローカルで自動生成された workload identity ID（例: `workload-xxxxxxxx`）が使われる。IAMポリシーのリソースARNが `workload-identity/sample_identity-*` のようなパターンだと一致しない。

解決策: IAMポリシーの workload-identity リソースをワイルドカードに広げる。
```json
{
  "Sid": "BedrockAgentCoreIdentityGetResourceOauth2Token",
  "Effect": "Allow",
  "Action": ["bedrock-agentcore:GetResourceOauth2Token"],
  "Resource": [
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:token-vault/default",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:token-vault/default/oauth2credentialprovider/*",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/*"
  ]
}
```

### 3LO callback で authorizationCode/state が null エラー

症状: 3LO (USER_FEDERATION) フローで、ブラウザの Atlassian 同意後に callback endpoint でバリデーションエラーが出る。
```
{"message":"2 validation errors detected: Value at 'authorizationCode' failed to satisfy constraint: Member must not be null; Value at 'state' failed to satisfy constraint: Member must not be null"}
```

原因: `callback_url` に AgentCore の callback endpoint を直接指定していた。3LO フローは2段階で動作するため、AgentCore callback からアプリの callback サーバーへのリダイレクトが必要になる。

解決策: アプリ側でローカル callback サーバーを立て、`callback_url` をそちらに向ける。詳細は上記の「3LO (USER_FEDERATION) フローにはローカル callback サーバーが必要」を参照。

```python
callback_url="http://localhost:9090/oauth2/callback"

identity_client.complete_resource_token_auth(
    session_uri=session_id,
    user_identifier=UserIdIdentifier(user_id="<user_id>"),
)
```

追加設定:
- Workload Identity の `allowedResourceOauth2ReturnUrls` に localhost の URL を追加する
- `fastapi` と `uvicorn` を依存関係に追加する

GitHub Issue #801 と公式サンプル `05-Outbound_Auth_3lo/oauth2_callback_server.py` も参考になる。

## Runtime の中で API キーを読むときの workload token

- Runtime が呼び出しに workload token を付けるのは、inbound の JWT 認証や `runtimeUserId` 付きの IAM 呼び出しのようにユーザーID付きで呼ばれたときに限られる。Lambda からの素の IAM 呼び出しや AgentCore Gateway 経由では付かず、`BedrockAgentCoreContext.get_workload_access_token()` が空になる
- 付かない呼ばれ方の場合は、CLI の `create-workload-identity` か CDK の `CfnWorkloadIdentity` で名前付きのワークロード ID を作り、`bedrock-agentcore:GetWorkloadAccessToken` を許可したうえで、`boto3.client("bedrock-agentcore").get_workload_access_token(workloadName=...)` で自分で取得する。名前は環境変数で渡す
- `IdentityClient.get_api_key` は bedrock-agentcore 1.25.0 時点で async 関数になっている。同期で呼ぶとキーではなく coroutine が返るため、同期のツールの中では `get_resource_api_key(resourceCredentialProviderName=..., workloadIdentityToken=...)["apiKey"]` を直接呼ぶ
- API キーの資格情報プロバイダはリージョンごとに管理される。別リージョンの Runtime で同じキーを使うなら、そのリージョンにも同じ名前で登録する
