# Bedrock AgentCore runtime integration

## 目次

- Bedrock AgentCore との統合
  - 基本構造
  - エンドポイント
  - 必要な依存関係
  - import パスの罠: runtime サブモジュール経由だとトレースが出ない
  - セッションIDでAgentを管理（複数ユーザー対応）
  - SSE keep-alive パターン（長時間処理のコネクション維持）
  - ツール使用イベント送信

## Bedrock AgentCore との統合

### 基本構造
```python
from bedrock_agentcore import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()
agent = Agent(model="us.anthropic.claude-sonnet-4-5-20250929-v1:0")

@app.entrypoint
async def invoke(payload):
    prompt = payload.get("prompt", "")
    stream = agent.stream_async(prompt)
    async for event in stream:
        yield event

if __name__ == "__main__":
    app.run()  # ポート8080でリッスン
```

### エンドポイント
- `POST /invocations`: エージェント実行
- `GET /ping`: ヘルスチェック

### 必要な依存関係
```
# requirements.txt
bedrock-agentcore
strands-agents
tavily-python  # Web検索が必要な場合
```

注意: fastapi/uvicorn は bedrock-agentcore SDK に内包されているため、別途指定する必要はない。

### import パスの罠: runtime サブモジュール経由だとトレースが出ない

```python
# OK: トレースが出力される
from bedrock_agentcore import BedrockAgentCoreApp

# NG: トレースが出力されない（ログ・メトリクスは出るがトレースだけ欠落）
from bedrock_agentcore.runtime import BedrockAgentCoreApp
```

SDK のトップレベル `__init__.py` にある Observability の初期化フックを通らないことが原因と推測される。

### セッションIDでAgentを管理（複数ユーザー対応）

AgentCore で複数ユーザーの会話履歴を保持する場合は、セッションIDごとに Agent インスタンスを管理する。

```python
from strands import Agent

# セッションごとのAgentインスタンスを管理
_agent_sessions: dict[str, Agent] = {}

def get_or_create_agent(session_id: str | None) -> Agent:
    """セッションIDに対応するAgentを取得または作成"""
    # セッションIDがない場合は新規Agentを作成（履歴なし）
    if not session_id:
        return Agent(
            model="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
            system_prompt="...",
            tools=[...],
        )

    # 既存のセッションがあればそのAgentを返す
    if session_id in _agent_sessions:
        return _agent_sessions[session_id]

    # 新規セッションの場合はAgentを作成して保存
    agent = Agent(
        model="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
        system_prompt="...",
        tools=[...],
    )
    _agent_sessions[session_id] = agent
    return agent

@app.entrypoint
async def invoke(payload):
    session_id = payload.get("session_id")
    agent = get_or_create_agent(session_id)
    # ...
```

注意: メモリ上で管理しているため、コンテナを再起動するとセッションは消える。永続化が必要な場合は DynamoDB などの利用を検討する。

### SSE keep-alive パターン（長時間処理のコネクション維持）

ファイル変換や外部API呼び出しなどの同期的な重い処理を SSE で返す場合は、処理中に keep-alive イベントを送信してコネクションを維持する。`asyncio.run_in_executor` と `asyncio.shield`、タイムアウトを組み合わせて実装する。

```python
import asyncio

async def _wait_with_keepalive(task, format_name):
    """タスク完了を待ちつつ、5秒ごとにSSE keep-aliveイベントをyield"""
    while not task.done():
        try:
            await asyncio.wait_for(asyncio.shield(task), timeout=5.0)
        except asyncio.TimeoutError:
            yield {"type": "progress", "message": f"{format_name}変換中..."}

@app.entrypoint
async def invoke(payload, context=None):
    if action == "export_pptx" and markdown:
        try:
            print(f"[INFO] PPTX export started")
            loop = asyncio.get_event_loop()
            task = loop.run_in_executor(None, generate_pptx, markdown, theme)
            async for event in _wait_with_keepalive(task, "PPTX"):
                yield event  # 5秒ごとにprogressイベント送信
            result_bytes = task.result()
            yield {"type": "pptx", "data": base64.b64encode(result_bytes).decode()}
        except Exception as e:
            print(f"[ERROR] PPTX export failed: {e}")
            yield {"type": "error", "message": str(e)}
        return
```

ポイント:
- `asyncio.shield(task)` を使うことで、TimeoutError が起きてもタスクがキャンセルされない。
- `task.done()` でループを抜けるタイミングを判定し、`task.result()` で結果を取得する。
- フロントエンドの SSE パーサーは未知の `type` を無視するため、既存コードを変更せずに使える。

### ツール使用イベント送信
```python
@app.entrypoint
async def invoke(payload):
    global _generated_markdown
    _generated_markdown = None

    stream = agent.stream_async(payload.get("prompt", ""))
    async for event in stream:
        if "data" in event:
            yield {"type": "text", "data": event["data"]}
        elif "current_tool_use" in event:
            tool_name = event["current_tool_use"].get("name", "unknown")
            yield {"type": "tool_use", "data": tool_name}

    if _generated_markdown:
        yield {"type": "markdown", "data": _generated_markdown}
```

---

## Slack → Lambda → AgentCore の実装パターン

Slack Bot を API Gateway と Lambda 経由で AgentCore Runtime に繋ぐ構成で踏んだ罠をまとめる。

### Slack の 3 秒 ACK 制約は Lambda 自己 invoke で回避

Slack Events API は 3 秒以内に 200 を返さないと最大3回までリトライしてくる。ACK を返す前に数十秒かかる AgentCore を同期 invoke すると、毎回タイムアウトしてリトライが起き、重複排除で握りつぶして延命する不安定な状態になる。

対策として、署名検証と DynamoDB の条件付き Put などによる重複排除までを同期で済ませ、本処理は Lambda が自分自身を `InvocationType: Event` の非同期で invoke して即座に 200 を返す。ハンドラ冒頭で `mode` を確認し、「非同期本処理」の分岐に入れる。

- 自己 invoke には `lambda:InvokeFunction` 権限が必要だが、`handler.functionArn`（GetAtt）を自身のロールポリシーに入れると Function → DefaultPolicy → Function の循環依存になる。関数名を固定すると Lambda の置き換えが起き、他スタックが参照する ARN エクスポートをロックして deploy が詰まる。`formatArn` でリテラル生成した `スタック名-論理ID*` へのワイルドカード ARN を許可するのが安全な落とし所だ。
- 注意: 自己 invoke 後に本処理が失敗しても Slack からはリトライされない。エラー時のスレッド返信が唯一の通知手段になるため、重複排除は「受付側で先に」済ませる順序を保つ。

### Slack Web API は読み取り系が form-urlencoded 必須

`chat.postMessage` などの書き込み系は JSON body で通るが、`conversations.replies` / `conversations.history` / `conversations.info` / `files.getUploadURLExternal` などは JSON で送ると `invalid_arguments` になる。これらは `application/x-www-form-urlencoded` で送る。メソッドごとに Content-Type を出し分け、urlencoded 必須のメソッドを Set で一元管理しておけば、対象が増えても 1 行の追加で済む。

- スレッドの文脈取得に使う `conversations.replies` を JSON で送っていて実は一度も成功していなかったという潜在バグが、「同一チャンネルのメンションだけ」の運用では表面化せず、DM や複数チャンネル対応で初めて露見した事例がある。

### 非同期パイプラインの DynamoDB ゾンビレコード対策

「登録 → 数分後に別ジョブ（画像生成等）が `update_item` で結果を書き戻す」構成では、その間にレコードが削除されると `update_item` の upsert 動作によって部分レコードが復活し、`title` などの必須属性が無い不完全な項目が生まれる。これが KB 同期で `record["title"]` を参照するような後段のバッチを `KeyError` で全滅させることがある。

対策:
- 更新系では `ConditionExpression="attribute_exists(pk)"` を基本とし、削除済みキーへの書き戻しを弾く。
- バッチ側では不完全なレコードを見つけても 1 件で全体を止めず、警告ログを出してスキップする。
- 非同期パイプラインを含む設計では、`update_item` を条件なしの upsert のまま使わないことを原則にする。

---

## AgentCore Runtime のコスト（メモリ従量課金）の罠

AgentCore Runtime の請求で `USE1-Runtime:Consumption-based:Memory` が突出していたら、まずこの罠を疑う。モデル呼び出し料ではなく、「セッションがアイドルで居座る空課金」が主因であることが多い。短い間隔で定期実行するボットでは、請求の8割以上をメモリ従量が占め、モデル料は1割に満たないこともある。

### 課金の仕組み（一次情報・2026-07 時点）

- **メモリ課金は「セッションが生存している間ずっと」発生し、アイドル中も止まらない**。CPU は I/O 待ちで消費がゼロなら課金されないが、メモリは「その秒までのピークメモリ × 生存秒数」で毎秒課金され続ける。us-east-1 の単価はメモリが $0.00945/GB-Hour、CPU が $0.0895/vCPU-Hour で、128MB・1秒が下限である。
- セッションの状態は Active → Idle → Terminated の順に遷移する。処理を返したあとも、デフォルトでは 15分（`idleRuntimeSessionTimeout`=900秒）の間、アイドルのまま provisioned で生存し、その間もメモリ課金が続く。`maxLifetime` の最大値は 8時間である。
- 非同期処理やバックグラウンド処理がヘルスチェックに `HealthyBusy` を返している間は Active 扱いとして生き続けるため、意図せず長時間メモリを確保し続けることがある。
- Runtime を「デプロイしているだけ（セッション0件）」の状態ではコンピュート課金は発生しない。課金はセッション単位で計算される。
- 逆算の仕方: メモリの請求額を $0.00945 で割ると GB-Hours が求まる。たとえば $75 なら約 7,900 GB-Hours となり、3週間にわたり平均15GBを24時間確保し続けたのと同等になる。

### やりがちな構造（犯人）

- `rate(2 minutes)` などの高頻度ポーリング Scheduler でバッチ mode を叩くと、1回の処理が数秒で終わっても新規セッションが15分居座る。結果としてセッションが重なって常時7〜8個が生存し、空ポーリングだけで固定費が積み上がる。試験運用で人間トラフィックがゼロでもコストが発生する。
- 定期実行の処理が呼び出しごとに新規 UUID のセッションを作りながら、どこからも `StopRuntimeSession` を呼ばない構造になっている。

### lifecycle パラメータの範囲と適用単位（設計の分岐点）

`CreateAgentRuntime` と `UpdateAgentRuntime` の入力である `LifecycleConfiguration` の設定範囲は次の通りである。

| パラメータ | 有効範囲 | デフォルト |
|---|---|---|
| `idleRuntimeSessionTimeout` | 60〜28800秒（1秒単位） | 900（15分） |
| `maxLifetime` | 60〜28800秒 | 28800（8時間・上限固定、超過不可） |

- idle の設定値は最短60秒で、0秒や数秒にはできない。「極限まで短く」した場合でも下限は60秒になる。
- lifecycle の設定は Runtime 単位で固定される。データプレーンの `InvokeAgentRuntime` には lifecycle 系の入力パラメータが無く、呼び出しやセッションごとの出し分けはできない。1つの Runtime で「対話呼び出しはidle長め・バッチ呼び出しはidle極短」のように使い分けることはできない。
- セッション種別ごとに終わらせ方を変えたいときの選択肢は2つある。①用途別に Runtime を分けてデプロイし、バッチ用 Runtime だけ idle=60秒にする方法と、②後述の `StopRuntimeSession` を使ってアプリ側から早期終了する方法だ。Runtime を分割するとデプロイ面が増えるため、非対話バッチでは②が第一候補になる。

### `StopRuntimeSession` の呼び出し作法

- 呼び出すのは SSE 応答のストリームを完全に読み切ってからにする。早すぎると「stops any ongoing streaming responses」の仕様により、自分の応答が途中で切れる。
- 呼び出し主体はクライアントや Lambda など、呼び出し側にするのが定石である。`invoke_agent_runtime` を呼んだ側で `runtimeSessionId` を追跡し、応答の受信が完了したあとに finally 節で `stop_runtime_session(agentRuntimeArn=..., runtimeSessionId=..., qualifier='DEFAULT')` を呼ぶ。
- 既に終了したセッションを二重に stop すると `ResourceNotFoundException`(404) が返るため、このエラーは握りつぶしてよい。必要な IAM 権限は `bedrock-agentcore:StopRuntimeSession` である。

### CDK での lifecycle 指定（L2・stable）

```python
from aws_cdk import Duration, aws_bedrockagentcore as agentcore
agentcore.Runtime(self, "runtime",
    runtime_name="...",
    agent_runtime_artifact=artifact,
    lifecycle_configuration=agentcore.LifecycleConfiguration(
        idle_runtime_session_timeout=Duration.minutes(1),  # 最小60秒
        max_lifetime=Duration.hours(4),
    ),
)
```

- L1 の `CfnRuntime.LifecycleConfigurationProperty` では `IdleRuntimeSessionTimeout` と `MaxLifetime` を秒数で指定する。
- 旧 alpha モジュール `aws_cdk.aws_bedrock_agentcore_alpha` の `LifecycleConfiguration` は Deprecated のため、安定版の `aws_cdk.aws_bedrockagentcore` を使う。

### 削減の主戦場（効果の大きい順）

1. 会話の継続が不要な単発処理やバッチ処理では、応答を返した直後に `StopRuntimeSession` を呼ぶのを最優先にする。15分のアイドルによる空課金を丸ごと消せるうえ、idle 短縮と違って実質0秒で畳めるため、これが「セッション種別ごとに極短」を実現する正攻法になる。
2. Runtime 単位の安全網として `idleRuntimeSessionTimeout` を60秒へ短縮する。1のフォールバックとして機能し、対話系の follow-up 後に残る待ち時間も短く抑えられる。
3. ポーリング頻度を下げるか、イベント駆動へ切り替える。「2分毎に空ポーリング」する仕組みを「書き込みイベントで起こす」構成に変えるのが最も効果が大きい。締切 epoch 方式のようなデバウンスが既にあるなら、SQS の `DelaySeconds`（最大900秒）で遅延 enqueue すると既存の冪等ロジックをそのまま流用でき、即時発火で遅延を表現できない DynamoDB Streams よりも配線を単純にできる。
4. 暴走して生存し続ける上限として `maxLifetime` を絞り、`HealthyBusy` を返し続けていないか点検するとともに、常駐ライブラリやインメモリキャッシュによるピークメモリを削る。
5. 状態は「セッションを生かして保持」するのではなく DynamoDB などの外部に逃がし、セッション自体は短命に保つのが公式の設計指針である。

### 「削除」か「idle短縮」かの判断（コスト最適化の勘所）

- Runtime は「存在するだけ」では課金されず、セッションが起動したときだけメモリ GB-Hours や vCPU の consumption 課金が発生する。ECR やコードのストレージ課金は別途少額かかる。
- したがって、呼ばれていない放置 Runtime の compute 費用は実質 $0 である。誤呼び出し防止や ECR 整理、混乱回避のために棚卸しで消す価値はあるが、コスト削減の主効果にはならない。
- メモリ課金が積み上がっているのは、実際に呼ばれている Runtime である。トークン課金の有無が実利用の判定材料になり、この対象には削除ではなく idle 短縮と StopRuntimeSession でアイドル時間の尾引きを削る対策が有効だ。
- 原則として、使わないデモや検証用に名前を変えて作った重複の枝は削除し、現役の本体は短縮で対処する。UsageType の内訳を見て「Memoryが支配的かつトークン課金ゼロ＝純アイドル」なら削除の効果が最も大きく、「トークン課金あり＝実利用＋尾引き」なら短縮が本命になる。

### コスト調査の手順（月額タグに慌てず「継続 vs 一過性」を切り分ける）

「コスト配分タグの月額が高い」という事実だけで対策に走らない。存在課金ではないため、高い月額が既に止まった一過性のバーストによるものだったケースがある。月初に手動の検証が集中しただけでその後はゼロになり、現在の run-rate は保管料だけになっている状態だ。切り分け手順は次の通りである。

1. Cost Explorer を DAILY で取得する（`get-cost-and-usage` で Granularity=DAILY を指定）。毎日平坦に課金されていれば継続、山型で最近ゼロになっていれば一過性と判断できる。
2. CloudWatch の `AWS/Bedrock-AgentCore` から `Invocations` と `SessionCount` を Runtime ARN 別に取得する。ゼロなら呼ばれておらず compute 課金も発生していない。先に `list-metrics` で実際のメトリクス名とディメンションを確認しておき、SEARCH 式で全ディメンションを網羅すると取りこぼしを防げる。
3. EventBridge Scheduler と Rule を `scheduler list-schedules` と `events list-rules` で確認する。意図しない定期呼び出し元が見つかった場合は、idle 短縮より先にそちらを止めるのが本命になる。
4. 一過性と分かったら対応は不要である。Secrets Manager や ECR などの保管料だけ気になる場合は棚卸しして削除する。

### 一次情報（2026-07-23 取得）

- 料金（課金対象の原文・単価）: https://aws.amazon.com/bedrock/agentcore/pricing/
- ライフサイクル Quota（idle 15分 / max 8時間）: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/bedrock-agentcore-limits.html
- StopRuntimeSession: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-stop-session.html
- 請求可視化メトリクス `MemoryUsed-GBHours` / `CPUUsed-vCPUHours`: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-runtime-metrics.html
