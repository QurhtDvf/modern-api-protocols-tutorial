# 🤖 Local Responses API Server (CPU)

OpenAI Responses API 互換のローカルLLMサーバーを **Google Colab の CPU のみ** で動かすJupyter Notebookです。

**llama-cpp-python + Qwen2.5-1.5B + FastAPI + cloudflared** の構成で、テキスト生成と Function calling に対応します。

---

## 📋 目次

- [Responses API とは](#responses-api-とは)
- [Function Calling とは](#function-calling-とは)
- [cloudflared とは](#cloudflared-とは)
- [uvicorn とは](#uvicorn-とは)
- [セットアップ](#セットアップ)
- [使い方](#使い方)
- [構成](#構成)
- [トラブルシューティング](#トラブルシューティング)

---

## Responses API とは

### 概要

Responses API は、OpenAI が 2025年3月に発表した新しいAPIインターフェースです。従来の Chat Completions API (`/v1/chat/completions`) を置き換えることを意図しており、テキスト生成・ツール呼び出し・マルチモーダル処理を統一的なインターフェースで扱えるように設計されています。

エンドポイントは `/v1/responses` で、リクエストの `input` フィールドに文字列またはメッセージ配列を渡し、`output` 配列でレスポンスを受け取ります。

### 歴史

| 年 | 出来事 |
|---|---|
| 2020年 | OpenAI が GPT-3 を発表。APIは `/v1/completions`（テキスト補完形式）を採用 |
| 2022年 | ChatGPT の公開に合わせ `/v1/chat/completions` が登場。ロールベースのメッセージ形式が普及 |
| 2023年 | Function calling が Chat Completions API に追加（GPT-3.5-turbo / GPT-4） |
| 2024年 | Assistants API が登場。ファイル・スレッド・ツールを統合管理する高レベルAPIとして提供 |
| 2025年3月 | **Responses API が発表**。Chat Completions と Assistants API の設計思想を統合し、シンプルかつ拡張性の高い新標準として位置づけられる |

### Chat Completions API との主な違い

| 項目 | Chat Completions API | Responses API |
|---|---|---|
| エンドポイント | `/v1/chat/completions` | `/v1/responses` |
| 入力フィールド | `messages` | `input` |
| 出力フィールド | `choices[].message` | `output[]` |
| ツール出力タイプ | `tool_calls` | `function_call` |
| システムプロンプト | `messages` 内の `system` ロール | `instructions` フィールド |
| ステート管理 | なし | `previous_response_id` で連鎖可能 |

---

## Function Calling とは

### 概要

Function calling（ツール呼び出し）は、LLMがユーザーの要求に応じて外部関数を呼び出すための仕組みです。モデルは関数を直接実行するのではなく、「この関数をこの引数で呼び出すべき」という構造化された出力を返し、実際の実行はアプリケーション側で行います。

これにより、LLMをデータベース検索・API呼び出し・計算処理などと組み合わせた**エージェント**を構築できます。

### 歴史

| 年 | 出来事 |
|---|---|
| 2023年6月 | OpenAI が GPT-3.5-turbo / GPT-4 に **Function calling を追加**。JSON スキーマでツールを定義し、モデルが呼び出しを判断する仕組みを提供 |
| 2023年11月 | OpenAI DevDay にて Parallel function calling（複数ツールの同時呼び出し）を発表 |
| 2024年 | Anthropic（Claude）・Google（Gemini）・Mistral など主要LLMが相次いでFunction callingに対応。業界標準の機能となる |
| 2024年後半 | ローカルLLM（llama.cpp / Ollama 等）でも対応モデルが増加。Qwen2.5・Llama-3.2 など小型モデルでも利用可能に |
| 2025年 | OpenAI Responses API にて `function_call` タイプとして再設計。エージェントループとの親和性が向上 |

### 動作の流れ

```
1. ユーザーが質問を送信
        ↓
2. モデルがツール呼び出しを判断
        ↓
3. モデルが function_call を返す（引数付き）
        ↓
4. アプリが実際の関数を実行
        ↓
5. 結果をモデルに返す
        ↓
6. モデルが最終回答を生成
```

### Qwen2.5 系モデルの特徴

Qwen2.5 系は Function calling を `tool_calls` フィールドではなく、テキスト本文中に以下の形式で出力する場合があります：

```
<tool_call>
{"name": "get_weather", "arguments": {"city": "Tokyo"}}
</tool_call>
```

本リポジトリのサーバーはこの形式を正規表現でパースし、Responses API の `function_call` 形式に自動変換します。

---

## cloudflared とは

### 概要

cloudflared は、Cloudflare が提供するトンネリングクライアントです。ローカルで動いているサーバーをインターネットに公開するための安全なトンネルを作成します。

### 特徴

- **アカウント不要・無料**（Quick Tunnels 機能）でHTTPSのパブリックURLを発行できます
- Cloudflare のグローバルネットワークを経由するため、DDoS 対策や TLS 終端が自動で行われます
- ngrok と異なり、無料プランでも認証トークン不要で使えます（`trycloudflare.com` サブドメインが払い出されます）

### ngrok との比較

| 項目 | cloudflared (Quick Tunnels) | ngrok 無料プラン |
|---|---|---|
| アカウント | 不要 | 必要 |
| 認証トークン | 不要 | 必要 |
| URL形式 | `*.trycloudflare.com` | `*.ngrok-free.app` |
| セッション制限 | なし | あり |
| 速度 | 高速（Cloudflareエッジ経由） | 普通 |

### 使い方（本リポジトリ）

```bash
# インストール
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
  -O /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared

# トンネル起動
cloudflared tunnel --url http://localhost:8000
```

起動後、標準出力に `https://xxxx.trycloudflare.com` の形式でURLが表示されます。

---

## uvicorn とは

### 概要

uvicorn は、Python の ASGI（Asynchronous Server Gateway Interface）対応の高速 Web サーバーです。FastAPI アプリケーションを本番・開発問わず起動するための標準的なサーバーとして広く使われています。

### 特徴

- **非同期処理対応**：`asyncio` ベースで高い並行性を実現します
- **高速**：純粋な Python 実装でありながら、uvloop（オプション）を使うことで Node.js に匹敵するパフォーマンスを発揮します
- **ASGI 標準**：FastAPI・Starlette・Django（ASGI モード）など主要フレームワークをサポートします

### WSGI との違い

従来の Flask・Django（WSGI）はリクエストを同期的に処理するため、1リクエストが終わるまで次を待つ必要がありました。uvicorn + FastAPI は非同期処理により、LLM の推論待ち中でも他のリクエストを受け付けられます。

### 本リポジトリでの使い方

```bash
uvicorn responses_api_server:app --host 0.0.0.0 --port 8000
```

| オプション | 説明 |
|---|---|
| `responses_api_server:app` | `responses_api_server.py` 内の `app` オブジェクトを指定 |
| `--host 0.0.0.0` | 全インターフェースでリクエストを受け付ける（cloudflaredからのアクセスに必要） |
| `--port 8000` | ポート番号 |

---

## セットアップ

### 必要なもの

- Google アカウント（Google Colab 利用のため）
- それだけです。ngrok アカウントも Hugging Face トークンも不要です。

### 手順

1. `responses_api_server_final.ipynb` を [Google Colab](https://colab.research.google.com/) で開く
2. ランタイム → 「ランタイムのタイプを変更」→ **CPU** を選択
3. 上から順にセルを実行する

---

## 使い方

### エンドポイント

| メソッド | エンドポイント | 説明 |
|---|---|---|
| GET | `/health` | サーバーの状態確認 |
| GET | `/v1/models` | 利用可能なモデル一覧 |
| POST | `/v1/responses` | テキスト生成・ツール呼び出し |

### テキスト生成

```python
import requests

response = requests.post("https://xxxx.trycloudflare.com/v1/responses", json={
    "model": "qwen2.5-1.5b-instruct-q4_k_m.gguf",
    "input": "Pythonでフィボナッチ数列を書いてください。",
    "instructions": "あなたは優秀なプログラミングアシスタントです。",
    "max_output_tokens": 400,
    "temperature": 0.5,
})

print(response.json()["output"][0]["content"][0]["text"])
```

### Function Calling

```python
tools = [
    {
        "type": "function",
        "name": "get_weather",
        "description": "指定した都市の天気を取得する",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "都市名"}
            },
            "required": ["city"]
        }
    }
]

response = requests.post("https://xxxx.trycloudflare.com/v1/responses", json={
    "model": "qwen2.5-1.5b-instruct-q4_k_m.gguf",
    "input": "東京の天気を教えてください。",
    "tools": tools,
    "tool_choice": "auto",
    "temperature": 0.1,
})

output = response.json()["output"][0]
if output["type"] == "function_call":
    print(f"ツール: {output['name']}, 引数: {output['arguments']}")
```

---

## 構成

```
.
├── responses_api_server_final.ipynb   # メインノートブック
└── README.md
```

ノートブック内で `/content/responses_api_server.py` が自動生成されます。

---

## トラブルシューティング

### `RepositoryNotFoundError: 401`

Gemma 系モデルはライセンス同意が必要なため認証エラーになります。本リポジトリでは認証不要の **Qwen2.5** を使用しています。

### サーバーを再起動しても変更が反映されない

`pkill -f uvicorn` で旧プロセスを確実に終了してから再起動してください。

```python
import subprocess, time
subprocess.run(["pkill", "-f", "uvicorn"])
time.sleep(3)
# → 改めて uvicorn を起動
```

### Function calling で `KeyError: 'name'`

`output[0]["type"]` が `"function_call"` ではなく `"message"` の場合に発生します。サーバーコード（Step 3）を再実行後、uvicorn を再起動してください。

### cloudflared の URL が取得できない

以下で再試行してください：

```python
import subprocess, re
cf_proc = subprocess.Popen(
    ["cloudflared", "tunnel", "--url", "http://localhost:8000"],
    stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
)
for _ in range(60):
    line = cf_proc.stdout.readline().decode()
    match = re.search(r"https://[a-z0-9\-]+\.trycloudflare\.com", line)
    if match:
        print(match.group(0))
        break
```

---

## ライセンス

MIT

---

## 使用モデル

[Qwen2.5-1.5B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF) — Alibaba Cloud / Apache 2.0 License
