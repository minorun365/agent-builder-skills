# Deploy, CLI, and CodeZip pitfalls

## 目次

- update_agent_runtime の注意点
  - update_agent_runtime は全フィールド置換（パッチ更新ではない）
  - agentcore deploy で認証設定がリセットされる
  - Runtime を更新しても、続いている会話は古いコードのまま答える
- JWT 認証時の SDK/CLI 制約
  - JWT認証時に SDK/CLI で Authorization method mismatch
- CodeZip デプロイの罠
  - `--build CodeZip` は Playwright driver の実行権限で失敗する
  - Runtime の artifact type は in-place 更新できない
  - CodeZip の手動再パッケージングで UPDATE_FAILED
- トラブルシューティング
  - AgentCore Runtime を CLI から手動呼び出しする方法
- AgentCore CLI (`@aws/agentcore`)
  - インストール
  - プロジェクト作成（対話式 vs 非対話式）
  - デプロイの非対話化（aws-targets.json 直書き方式）
  - `aws-targets.json` のスキーマ（v1）
  - agentcore.json の runtimes 定義
  - 他の便利コマンド
  - TUI を完全自動化したいとき（最後の手段）

## update_agent_runtime の注意点

### update_agent_runtime は全フィールド置換（パッチ更新ではない）

症状：`bedrock-agentcore-control` の `update_agent_runtime` API でコードを更新したところ、環境変数と認証設定（authorizerConfiguration）が消えた。

原因：`update_agent_runtime` はパッチ更新ではなく全置換のため、省略したフィールドは空にリセットされる。

解決策：更新時は全パラメータを明示的に渡す。

```python
control.update_agent_runtime(
    agentRuntimeId='...',
    roleArn='...',                        # 必須
    networkConfiguration={...},           # 必須
    agentRuntimeArtifact={...},           # コード
    environmentVariables={                # 省略すると空になる！
        'AWS_DEFAULT_REGION': 'us-east-1',
        'MEMORY_ID': '...',
    },
    authorizerConfiguration={             # 省略すると認証なしになる！
        'customJWTAuthorizer': {
            'discoveryUrl': '...',
            'allowedClients': ['...'],
        }
    },
)
```

補足：`agentcore deploy` CLI はインタラクティブターミナル（Ink/raw mode）が必要なため、CI/CD やサブプロセスからは実行できない。非対話環境では boto3 の `update_agent_runtime` を使う。

### agentcore deploy で認証設定がリセットされる

症状：`agentcore deploy` を実行すると、マネコンで設定しておいた `authorizerConfiguration`（Cognito JWT 等）が空になる。

原因：AgentCore CLI の deploy は `authorizerConfiguration` を管理対象にしておらず、デプロイ時にランタイム設定が上書きされて認証設定が空になる。

解決策：デプロイ後にマネコンか API で認証設定を再適用する。環境変数は `agentcore.json` の `envVars` で管理すれば保持される。

### Runtime を更新しても、続いている会話は古いコードのまま答える

症状：Runtime を更新した直後にアプリから試すと、更新前のコードのまま応答する。確認用に新しい会話IDで呼ぶと新しい挙動になる。

原因：AgentCore Runtime は、同じ `runtimeSessionId` の呼び出しを同じ実行環境へ送り続ける。実行環境は使われない時間が `idleRuntimeSessionTimeout`（既定15分）に達するまで残り、呼ぶたびに期限が延びる。Runtime を更新しても残っている実行環境は古いコードで動くため、チャットのルームやスレッド単位などで会話IDを固定しているアプリでは、試すたびに古い実行環境が延命される。

確認の仕方：デプロイ後の確認は、アプリが実際に使う会話IDで行う。新しい会話IDで呼ぶと新しい実行環境が立ち上がるため、成功したように見えても確認にならない。

解決策：`bedrock-agentcore` の `stop_runtime_session` で古い実行環境を止める。恒久対策としては、CDK なら `Asset.assetHash` のようにコードの版を示す値を会話IDに含め、デプロイのたびに会話IDが変わるようにする。会話の記憶は実行環境の中ではなく、AgentCore Memory などの外部に置く作りが前提になる。

---

## JWT 認証時の SDK/CLI 制約

### JWT認証時に SDK/CLI で Authorization method mismatch

症状：`agentcore invoke` や boto3 の `invoke_agent_runtime` で呼び出すと、次のエラーが出る。
```
Authorization method mismatch. The agent is configured for a different authorization method than what was used in your request.
```

原因：Runtime に `customJWTAuthorizer` が設定されている場合、SigV4（IAM認証）での呼び出しは拒否される。

解決策：HTTPS エンドポイントへ直接リクエストを送り、`Authorization: Bearer {JWT}` ヘッダーで認証する。

```bash
curl -X POST \
  "https://bedrock-agentcore.${REGION}.amazonaws.com/runtimes/${ENCODED_ARN}/invocations?qualifier=DEFAULT" \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: ${SESSION_ID}" \
  -d '{"prompt": "hello"}'
```

注意：`runtimeSessionId` は33〜256文字にする必要があり、短いとバリデーションエラーになる。

---

## CodeZip デプロイの罠

### `--build CodeZip` は Playwright driver の実行権限で失敗する

症状：`agentcore add agent --build CodeZip` で作成したランタイムで `AgentCoreBrowser` / `strands_tools.browser` を使うと、Browser ツールの呼び出し時に次のエラーが出る。

```
Error: PermissionError - [Errno 13] Permission denied: '/var/task/playwright/driver/node'
```

原因：S3 ZIP デプロイの CodeZip は、zip アーカイブを展開するときにファイルの実行権限（+x）を保持しない。Playwright に同梱されている `driver/node` バイナリの実行ビットが落ちたまま展開されるため、ブラウザ自動化の裏側で node プロセスを起動できなくなる。

対策：Playwright を使うエージェントでは、ECR デプロイとなる `--build Container` を指定する。Dockerfile 内で `pip install playwright && playwright install` を実行すれば、実行権限が付いた状態でデプロイされる。

```bash
# ✅ Playwright / Browser tool を使うなら Container
agentcore add agent --name <name> --build Container --framework Strands ...

# ❌ NG: CodeZip で playwright を含むと driver/node の実行権限問題で失敗
```

他にも Chromium、Node.js ドライバ、ネイティブ拡張など、実行ビットが必要なバイナリを含むアプリでは同じ問題が起きる。

### Runtime の artifact type は in-place 更新できない

症状：一度 CodeZip で作成した AgentCore Runtime を CDK 経由で Container へ切り替えようとすると、CloudFormation が次のエラーを出して UPDATE_ROLLBACK になる。

```
Invalid request provided: Agent artifact type cannot be updated
(Service: BedrockAgentCoreControl, Status Code: 400)
```

原因：`AWS::BedrockAgentCore::Runtime` リソースの `AgentRuntimeArtifact.CodeConfiguration` と `ContainerConfiguration` は相互排他になっており、作成後には切り替えられない。

対策：スタックごと作り直す。CloudFormation スタックを `aws cloudformation delete-stack` で削除してから `agentcore deploy` を再実行する。Runtime ARN が変わるため、以下もやり直す必要がある。

- Amplify アプリの環境変数 `NEXT_PUBLIC_AGENT_ARN` を新しい ARN に更新して再ビルドする
- Workload Identity の許可URLを再設定する
- 実行ロールへの IAM ポリシーを再アタッチする
- `update-agent-runtime` で環境変数と JWT authorizer を再適用する

```bash
aws cloudformation delete-stack --stack-name <stack-name>
# DELETE_COMPLETE まで待機（通常数分）
agentcore deploy -y --verbose
```

教訓：**最初から正しい `--build` を選ぶ**。Playwright や Chromium を使うなら Container 一択。

### CodeZip の手動再パッケージングで UPDATE_FAILED

症状：`update_agent_runtime` で新しい ZIP をアップロードしてデプロイすると `UPDATE_FAILED` になる。コンテナが起動前に失敗しているため、CloudWatch にランタイムログが一切出力されない。

原因：agentcore CLI が作成した ZIP と、ステージングディレクトリから `zip -r` で新規作成した ZIP では内部構造が異なる。AgentCore のデプロイパイプラインが ZIP 構造を厳密にチェックしている可能性がある。

解決策：前回成功した ZIP をコピーし、`zip -u` で差分更新する。

```bash
# ❌ NG: ステージングディレクトリから新規作成 → UPDATE_FAILED
cd staging/ && zip -r /tmp/new.zip .

# ✅ OK: 前回成功の ZIP をベースに差分更新
cp /tmp/v21_success.zip /tmp/v24.zip
# main.py を修正してから:
cd /tmp/extract/ && zip -u /tmp/v24.zip main.py
```

補足：
- `UPDATE_FAILED` になっても `statusReason` フィールドがないため、API からエラーの詳細は取得できない
- CloudWatch にログが出ない場合は、ZIP 構造の問題やモジュール初期化タイムアウトなど、コンテナ起動前の失敗を疑う
- `get-agent-runtime` の `liveVersion` で現在稼働中のバージョンを確認できる

---

## トラブルシューティング

### AgentCore Runtime を CLI から手動呼び出しする方法

payload は base64 でエンコードして渡す。

```bash
PAYLOAD=$(echo -n '{"prompt": "Hello"}' | base64)
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:REGION:ACCOUNT:runtime/RUNTIME_ID" \
  --payload "$PAYLOAD" \
  --content-type "application/json" \
  --profile <profile> \
  --region us-east-1 \
  /tmp/response.txt
```

日本語の payload は使えない。

```
string argument should contain only ASCII characters
```

AWS CLI がマルチバイト文字を受け付けないためである。システムプロンプトが日本語なら、英語のプロンプトでも問題なく動く。

レスポンスはファイルに保存される。最後の引数が書き出し先のファイルパスで、省略できない。

---

## AgentCore CLI (`@aws/agentcore`)

CDK の生コードではなく公式 CLI でエージェントプロジェクトを作る場合のノウハウで、内容は v0.8.2 時点の検証結果に基づく。

### インストール

```bash
npm install -g @aws/agentcore
agentcore --version  # 0.8.2
```

プレビュー版のためバージョンは固定しない。古い CLI は動かなくなるリスクがあるので、常に最新を使う。

### プロジェクト作成（対話式 vs 非対話式）

対話式（ブラウザ Codespace 等の TTY がある環境）：
```bash
agentcore create   # TUI起動 → ガイド付きプロンプト
```

非対話式（CI/CD や SSH 経由）：
```bash
# ワンショットでフル生成（Python, Strands, Bedrock, memory なし）
agentcore create --name handson --defaults

# Agent 名をカスタマイズしたいとき
agentcore create --name handson --no-agent --skip-install
cd handson
agentcore add agent --name MyAgent --type create --language Python \
  --framework Strands --model-provider Bedrock --memory none \
  --build CodeZip --protocol HTTP
```

罠：
- `--skip-install` を付けると `agentcore/cdk/` の `npm install` がスキップされ、後の deploy 時に `tsc: not found` で失敗する。必要なら `cd agentcore/cdk && npm install` を手動で実行する
- `agentcore add agent` は `--language` が必須で、省略すると `--language is required` エラーになる

### デプロイの非対話化（aws-targets.json 直書き方式）

`agentcore deploy -y` は TTY なしでも動くが、`agentcore/aws-targets.json` が空 `[]` のままだと `Target "default" not found in aws-targets.json` で即座に失敗する。TUI での初回 deploy 時に自動で書き込まれるものの、CI/CD では次のように直書きしておく必要がある。

```bash
# AWS アカウントID を自動取得してファイルを作成
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
printf '[{"name":"default","account":"%s","region":"us-east-1"}]' "$ACCOUNT" \
  > agentcore/aws-targets.json

# 非対話 deploy
agentcore deploy -y --json
```

### `aws-targets.json` のスキーマ（v1）

`aws/agentcore-cli` の公式 `docs/configuration.md` と `schemas/agentcore.schema.v1.json` で確認したスキーマは次の通り。

```json
[
  {
    "name": "default",
    "account": "123456789012",
    "region": "us-east-1",
    "description": "optional"
  }
]
```

- ルートは配列で、空 `[]` のままだとデプロイに失敗する
- `name` は `--target <name>` で指定する識別子で、デフォルトは `"default"`
- `account` は12桁の AWS アカウントID（必須）
- `region` は必須
- v0.8.2 時点では、`AGENTCORE_*` 環境変数で初期値を渡す仕組みはない

### agentcore.json の runtimes 定義

`agentcore create --defaults` で生成される `agentcore/agentcore.json` の `runtimes` 要素は次の形になる。

```json
{
  "name": "MyAgent",
  "build": "CodeZip",
  "entrypoint": "main.py",
  "codeLocation": "app/MyAgent/",
  "runtimeVersion": "PYTHON_3_14",
  "networkMode": "PUBLIC",
  "protocol": "HTTP"
}
```

- `build` には `CodeZip` か `Container` を指定する。`CodeZip` は S3 直接デプロイで Python のみ・最大250MB、`Container` は ECR で任意言語・最大2GB に対応する
- `protocol` は `HTTP` / `MCP` / `A2A` から選べる

### 他の便利コマンド

- `agentcore invoke "<prompt>" --stream`：CLI からデプロイ済みエージェントを呼び出す
- `agentcore logs --follow`：CloudWatch Logs をストリーミング表示する
- `agentcore status`：全リソースの状態を確認する
- `agentcore dev`：ローカルの dev サーバーを起動し、localhost で `/invocations` を受ける
- `agentcore package`：デプロイ用アーティファクトのビルドのみを行う

### TUI を完全自動化したいとき（最後の手段）

1. `script -qec 'agentcore deploy' /dev/null` で PTY を疑似的に確保する
2. `sudo apt-get install -y expect` で expect を入れ、スクリプトで画面遷移を操作する
3. `gh codespace ssh --codespace <name> -- -tt "<command>"` で OpenSSH の PTY を強制的に割り当てる
4. `aws/agentcore-cli` の `src/mcp-harness/` にある公式 TUI Harness（`npm run build:harness` が必要）を使い、MCP 経由で `tui_launch` / `tui_send_keys` / `tui_wait_for` を実行する

ただし、ハンズオン手順の検証といった目的なら、「`aws-targets.json` 直書き＋`deploy -y --json`」で完結し、この方法が最も安定する。

---
