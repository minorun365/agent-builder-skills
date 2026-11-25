# 会話履歴の管理

## 会話履歴の管理

```python
from strands import Agent

agent = Agent(model="us.anthropic.claude-sonnet-4-5-20250929-v1:0")

# 会話を継続
response1 = agent("私の名前は太郎です")
response2 = agent("私の名前は何ですか？")  # 「太郎」と答える

# 履歴をクリア
agent.clear_history()
```

### SlidingWindowConversationManager（履歴トリミング）

トークンコストを抑えるため、古いメッセージを自動で削除する組み込み機能である。

```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=tools,
    conversation_manager=SlidingWindowConversationManager(window_size=6),
)
```

#### パラメータ

| パラメータ | デフォルト | 説明 |
|-----------|----------|------|
| `window_size` | 40 | 保持する最大メッセージ数（ターン数ではない） |
| `should_truncate_results` | True | ツール結果の圧縮を有効にする |
| `per_turn` | False | `False`: 完了後のみ / `True`: 毎回 / `int(N)`: N回ごとにトリミング |

#### window_size の意味

`window_size` は Bedrock APIのメッセージ配列の要素数を指す。1つのメッセージが複数の content block（text, toolUse, toolResult）を持ちうるため、実際のテキスト量は window_size だけでは決まらない。

ツールを2つ使う典型的な1リクエストでは、メッセージ数は次のようになる。

```
[user]      ユーザーメッセージ          ... 1
[assistant] toolUse: web_search         ... 2
[user]      toolResult: 検索結果        ... 3
[assistant] toolUse: output_slide       ... 4
[user]      toolResult: スライド出力    ... 5
[assistant] 最終テキスト応答           ... 6
```

1リクエストが6メッセージになるため、`window_size=6` で約1リクエスト分を保持する。

#### トリミングアルゴリズム（2段階）

1. フェーズ1では、優先してツール結果を圧縮する
   - 古い toolResult の内容を `"The tool result was too large!"` に置き換える
   - メッセージ数は減らさない
   - 同じ toolResult を2回圧縮しない

2. フェーズ2では、最終手段としてメッセージを削除する
   - フェーズ1で足りない場合に、古いメッセージを削除する
   - toolUse/toolResult ペアの整合性を保ち、ペアが壊れない位置で削除する

#### per_turn パラメータの使い分け

```python
# デフォルト: エージェントループ完了後にのみトリミング
# -> 処理中は全メッセージが利用可能（品質重視）
SlidingWindowConversationManager(window_size=6)

# 毎回のモデル呼び出し前にトリミング
# -> ループが多いユースケースでトークン爆発を防止
SlidingWindowConversationManager(window_size=10, per_turn=True)

# N回ごとにトリミング（パフォーマンスバランス）
SlidingWindowConversationManager(window_size=10, per_turn=5)
```

#### per_turn=True と並列ツール実行の非互換性

**`per_turn=True` は、エージェントが複数のツールを並列発行するユースケースでは使用禁止である。**

Strands Agents は、`web_search` x2 のような並列ツールの結果を1件ずつ個別にセッション履歴へ追加する。`per_turn=True` だと各LLMコールの前にトリミングが実行され、次の正のフィードバックループが起きる。

1. LLMが `web_search` x2 を並列発行する
2. ツール結果の1件目が履歴に追加され、トリミングが発生する
3. 「検索結果が1件しかない」状態でLLMが呼ばれる
4. LLMが「情報不足」と判断し、追加の `web_search` を発行する
5. 1-4が連鎖し、1セッションで web_search が16-20回に増殖する

さらに `output_slide` などのツール結果が "The tool result was too large!" に圧縮され、ツールが正常に動かなくなる。

デフォルトの `per_turn=False` のまま使うことを推奨する。コスト削減には、要約などでツール結果のサイズを制限して対応する。

#### window_size チューニングの考え方

- フロントエンドが毎回最新のコンテキスト（生成済みMarkdownなど）を送る設計なら、古い履歴は不要なため、小さい window_size でよい
- デフォルトの `per_turn=False` であれば、処理中のメッセージはトリミングされないため品質に影響しない
- トリミングの効果は多ターンセッションで最も大きくなる

---

## 検索結果と質問の対象を取り違えない

検索して答えるエージェントは、ヒットした記述を、質問された対象の事実として答えてしまうことがある。製品Aについて聞かれて、製品Aと製品Bを比べた記事から製品Bの仕様を答える、という形である。直すときに、間違えた対象ごとの例外をプロンプトへ足していかない。通常の回答・追加の調査・会話の要約のどれでも、「質問の対象・知りたい属性・適用条件」と根拠の対応をそろえる。

- 検索にヒットしたことや、記事の書き手が誰かだけでは、その記述が質問の対象についての事実だとは言えない
- 会話の要約には、対象の切り替え・条件・未確認の点・訂正を残す。過去の回答を検証済みの事実へ格上げしない
- 知識ベースの見出しやラベルも照合する。出典の種類（製品ページ、比較記事など）を記述の対象と同一視すると、別の対象についての正しい本文まで取り違える。出典は出典として示し、各行の対象・属性・条件を保つ

回帰テストには、別の対象の情報を混ぜた検索結果、途中で対象を切り替える会話、情報が足りない質問、要約のあとも訂正が保たれるか、の4種類を入れる。単語が出たかどうかだけで取り違えを判定すると、別の対象だと明示した比較や否定まで不合格になる。誰の・何の属性として述べたかで判定する。
