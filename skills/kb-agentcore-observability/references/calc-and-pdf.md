# LLMにやらせてはいけない計算・FPDF2でPDF生成

## LLM にやらせてはいけない計算

### 日付→曜日の変換

LLMは日付から曜日を計算するのが苦手で、これは既知の弱点だ。`strands_tools` の `current_time` は ISO 8601 形式（例: `2026-02-09T02:46:56+00:00`）を返すが、曜日情報が含まれないためLLMが誤認識する。

実例として、2026年2月9日（月曜日）をLLMが「日曜日」と誤認識した。

原則として、LLMには計算させずツール側で確定情報を返す。

```python
from datetime import datetime, timezone, timedelta
from strands import tool

JST = timezone(timedelta(hours=9))
WEEKDAY_JA = ["月", "火", "水", "木", "金", "土", "日"]

@tool
def current_time() -> str:
    """現在の日本時間（JST）を曜日付きで取得します。"""
    now = datetime.now(JST)
    weekday = WEEKDAY_JA[now.weekday()]
    return f"{now.year}年{now.month}月{now.day}日({weekday}) {now.strftime('%H:%M')} JST"
```

ポイントは以下の通り。
- システムプロンプトの「+9時間して」は不確実なため、タイムゾーン変換もツール側で完結させる
- `Python の weekday()` は月曜=0なので、日本語曜日配列のインデックスと一致する
- `strands_tools.current_time` の代わりにカスタムツールを使う

---

## FPDF2でPDF生成（日本語対応）

### 日本語フォント（NotoSansCJKjp）

日本語PDFを生成するにはCJKフォントが必要になる。ここではNotoSansCJKjpを使う。

```dockerfile
# Dockerfile: フォントをプロジェクトからコピー
COPY fonts/ /app/fonts/
```

フォントの入手先は以下の通り。
- https://github.com/minoryorg/Noto-Sans-CJK-JP
- `fonts/NotoSansCJKjp-Regular.ttf`
- `fonts/NotoSansCJKjp-Bold.ttf`

```python
# agent.py: FPDF2でフォント登録
from fpdf import FPDF

class MyPDF(FPDF):
    def __init__(self):
        super().__init__()
        self.add_font("NotoSansCJKjp", fname="/app/fonts/NotoSansCJKjp-Regular.ttf")
        self.add_font("NotoSansCJKjp", style="B", fname="/app/fonts/NotoSansCJKjp-Bold.ttf")
```

### S3への保存と署名付きURL

```python
from botocore.config import Config

# 署名付きURL用のクライアント（s3v4必須）
s3_presigned = boto3.client(
    "s3",
    region_name=AWS_REGION,
    config=Config(signature_version="s3v4"),
)

# PDFをS3にアップロード
pdf_bytes = pdf.output()
s3_client.put_object(
    Bucket=UPLOAD_BUCKET,
    Key=f"reports/{report_id}.pdf",
    Body=pdf_bytes,
    ContentType="application/pdf",
)

# 署名付きURL生成（1時間有効）
download_url = s3_presigned.generate_presigned_url(
    ClientMethod="get_object",
    Params={"Bucket": UPLOAD_BUCKET, "Key": s3_key},
    ExpiresIn=3600,
)
```

### 注意点
- GitHubからフォントをcurlでダウンロードする場合、リダイレクト対応のため`-L`オプションが必須になる
- apt-getでfonts-noto-cjkをインストールするとTTC形式になり、FPDF2で追加設定が必要になる
- **プロジェクトにフォントファイルを含めてCOPYするのが最も確実だ**

---
