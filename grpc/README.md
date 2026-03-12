# gRPC 入門チュートリアル（Google Colab 対応）

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/your-username/grpc-tutorial/blob/main/grpc_tutorial.ipynb)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?logo=python)](https://www.python.org/)
[![gRPC](https://img.shields.io/badge/gRPC-1.50%2B-green?logo=grpc)](https://grpc.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

gRPC の基本概念から実装まで、Google Colab 上で動かしながら学べるチュートリアルです。  
4 種類の通信パターン・エラーハンドリング・認証メタデータ・Protocol Buffers のシリアライズ比較まで網羅しています。

---

## 目次

1. [gRPC とは](#grpc-とは)
2. [開発の背景](#開発の背景)
3. [アーキテクチャ概要](#アーキテクチャ概要)
4. [Protocol Buffers](#protocol-buffers)
5. [4 種類の通信パターン](#4-種類の通信パターン)
6. [gRPC vs REST API](#grpc-vs-rest-api)
7. [チュートリアルの内容](#チュートリアルの内容)
8. [クイックスタート](#クイックスタート)
9. [ディレクトリ構成](#ディレクトリ構成)
10. [参考リンク](#参考リンク)

---

## gRPC とは

**gRPC**（gRPC Remote Procedure Calls）は、Google が開発したオープンソースの高性能 RPC（Remote Procedure Call）フレームワークです。2016 年に公開され、現在は [Cloud Native Computing Foundation（CNCF）](https://www.cncf.io/) のインキュベーションプロジェクトとして管理されています。

> **RPC（Remote Procedure Call）とは？**  
> ネットワーク越しにある別のプロセスやサーバー上の関数を、あたかもローカルの関数を呼び出すかのように実行する仕組みです。gRPC はその現代的な実装であり、HTTP/2 と Protocol Buffers を採用することで高速かつ型安全な通信を実現します。

```
┌──────────────────────────────────────────────────────┐
│                    gRPC の全体像                       │
│                                                      │
│  ┌─────────────┐   .proto 定義   ┌─────────────────┐ │
│  │   Client    │◄──────────────►│     Server      │ │
│  │             │                │                 │ │
│  │  Stub       │── HTTP/2 ─────►│  Servicer       │ │
│  │  (自動生成)  │◄── protobuf ───│  (自動生成ベース) │ │
│  └─────────────┘                └─────────────────┘ │
│                                                      │
│        言語をまたいだ通信が可能（多言語対応）              │
└──────────────────────────────────────────────────────┘
```

---

## 開発の背景

### Google 社内の巨大な分散システム

gRPC の前身となった **Stubby** は、2000 年代初頭から Google 社内で運用されていた社内 RPC フレームワークです。Google のインフラは何万ものマイクロサービスで構成されており、毎秒数十億回ものサービス間通信が発生していました。このような規模では、通信の効率性・速度・信頼性が直接サービス品質に影響します。Stubby はその課題を解決するために設計・運用され続けてきました。

### HTTP/1.1 と JSON の限界

2010 年代に入り、REST + JSON が Web API の事実上の標準として広まりました。しかし、以下のような問題が顕在化していきました。

- **パフォーマンス**: JSON はテキストベースであるため、データサイズが大きくパースコストも高い
- **HTTP/1.1 の非効率性**: リクエストのたびに接続を確立し直す必要があり、多数のマイクロサービスが存在する環境では通信コストが増大する
- **型安全性の欠如**: JSON スキーマは強制力が弱く、クライアントとサーバーの契約（インターフェース）が曖昧になりやすい
- **ストリーミングの困難**: 標準の HTTP/1.1 では、サーバーからクライアントへの継続的なデータ配信（ストリーミング）を実現するのが難しい

### マイクロサービスアーキテクチャの普及

同時期、Netflix・Uber・Airbnb などの大規模テック企業を中心に **マイクロサービスアーキテクチャ** が急速に普及しました。モノリシックなアプリケーションを多数の小さなサービスに分割することで、スケーラビリティや開発速度が向上しますが、その代償としてサービス間通信の複雑さが増大します。

- サービスの数が増えるにつれ、通信のレイテンシが積み重なる
- 異なる言語・フレームワークで実装されたサービス同士が通信する必要が生じる
- API の変更が他のサービスに影響しないよう、厳密なインターフェース管理が求められる

### HTTP/2 の登場と gRPC の誕生

2015 年に HTTP/2 が標準化されました。HTTP/2 は多重化（1 つの接続で複数のリクエストを並列処理）、ヘッダー圧縮、サーバープッシュなど、HTTP/1.1 の欠点を解消する多くの改善を含んでいます。

Google はこれを機に Stubby をオープンソース化・標準化する形で **gRPC を開発し、2016 年に公開**しました。社内で実績を積んだ設計思想をベースに、HTTP/2 と Protocol Buffers を組み合わせることで、誰もが利用できる高性能 RPC フレームワークが誕生しました。

```
[時系列]
2000年代初頭  Google 社内で Stubby を開発・運用
2010年代      REST/JSON が普及、マイクロサービスが台頭
2015年        HTTP/2 標準化（RFC 7540）
2016年        gRPC をオープンソースとして公開
2017年        CNCF インキュベーションプロジェクトに採択
2019年        CNCF 卒業プロジェクトに昇格
現在          Kubernetes, Envoy など主要 OSS に採用
```

### gRPC が解決した課題

| 課題 | 解決策 |
|------|--------|
| 通信の非効率性 | HTTP/2 による多重化・ヘッダー圧縮 |
| データの冗長性 | Protocol Buffers（バイナリ形式）による高効率シリアライズ |
| 型安全性の欠如 | `.proto` ファイルによる厳密なスキーマ定義 |
| 多言語対応の困難 | `.proto` からの自動コード生成（10 言語以上に対応）|
| ストリーミングの欠如 | 4 種類の通信パターン（双方向ストリーミングを含む）|
| API 管理の複雑さ | スキーマファーストな設計によるインターフェースの明確化 |

---

## アーキテクチャ概要

gRPC は大きく 3 つのコンポーネントで構成されます。

### 1. サービス定義（`.proto` ファイル）

すべては `.proto` ファイルから始まります。サービスのインターフェース（どんな関数があり、何を受け取り、何を返すか）をここで定義します。

```protobuf
syntax = "proto3";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

### 2. コード自動生成（`protoc`）

`protoc`（Protocol Buffer Compiler）が `.proto` ファイルから各言語向けのコードを自動生成します。

```bash
python -m grpc_tools.protoc \
    -I . \
    --python_out=. \
    --grpc_python_out=. \
    demo.proto
```

生成されるファイル：
- `demo_pb2.py` — メッセージクラス（シリアライズ/デシリアライズ）
- `demo_pb2_grpc.py` — Stub（クライアント側）と Servicer（サーバー側）の基底クラス

### 3. サーバー／クライアント実装

自動生成されたクラスを継承して、実際のビジネスロジックを実装します。

```python
# サーバー側
class GreeterServicer(demo_pb2_grpc.GreeterServicer):
    def SayHello(self, request, context):
        return demo_pb2.HelloReply(message=f"Hello, {request.name}!")

# クライアント側
channel = grpc.insecure_channel('localhost:50051')
stub = demo_pb2_grpc.GreeterStub(channel)
response = stub.SayHello(demo_pb2.HelloRequest(name="World"))
```

---

## Protocol Buffers

**Protocol Buffers（protobuf）** は、Google が開発した言語・プラットフォーム中立なデータシリアライズフォーマットです。gRPC のデータ転送形式として使われています。

### JSON との比較

```python
# 同じデータを表現した場合のサイズ比較（実測値の目安）

JSON:    {"name": "gRPCテスト"}   → 約 28 bytes（テキスト）
protobuf: \n\x0egRPC\xe3\x83\x86\xe3\x82\xb9\xe3\x83\x88  → 約 18 bytes（バイナリ）
```

| 特性 | JSON | Protocol Buffers |
|------|------|-----------------|
| 形式 | テキスト | バイナリ |
| 可読性 | 高い | 低い（デバッグには `protoc --decode` を使用）|
| サイズ | 大きい | 3〜10 倍小さい |
| パース速度 | 遅い | 5〜10 倍高速 |
| スキーマ | 任意（JSON Schema は別途必要）| 必須（`.proto` で定義）|
| 後方互換性 | 管理が煩雑 | フィールド番号で自動管理 |

### フィールド番号による後方互換性

protobuf の重要な特性の一つが、**フィールド番号** による後方互換性の保証です。フィールドを追加しても既存のクライアントに影響せず、安全に API を進化させられます。

```protobuf
message UserRequest {
  string name = 1;       // 既存フィールド（変更不可）
  int32  age  = 2;       // 既存フィールド（変更不可）
  string email = 3;      // 新規追加 → 古いクライアントは無視するだけ
}
```

---

## 4 種類の通信パターン

### 1. Unary RPC（単項 RPC）

最もシンプルなパターン。通常の関数呼び出しと同様に、1 つのリクエストに対して 1 つのレスポンスを返します。

```
Client ──[Request]──► Server
Client ◄──[Response]─ Server
```

```protobuf
rpc SayHello (HelloRequest) returns (HelloReply);
```

**用途**: ユーザー認証、データ取得、CRUD 操作など

---

### 2. Server Streaming RPC（サーバーストリーミング RPC）

クライアントが 1 つのリクエストを送ると、サーバーが複数のレスポンスをストリームで返します。

```
Client ──[Request]────────────────────────► Server
Client ◄──[Response 1]── [Response 2]── ... Server
```

```protobuf
rpc SayHelloMultipleTimes (HelloRequest) returns (stream HelloReply);
```

**用途**: ログのリアルタイム配信、進捗通知、大量データの段階的な転送

---

### 3. Client Streaming RPC（クライアントストリーミング RPC）

クライアントが複数のリクエストを送り続け、すべて受信したサーバーが 1 つのレスポンスを返します。

```
Client ──[Request 1]── [Request 2]── ... ──► Server
Client ◄────────────────────────[Response]── Server
```

```protobuf
rpc Sum (stream NumberRequest) returns (CalcReply);
```

**用途**: ファイルアップロード、センサーデータの集約、バッチ処理

---

### 4. Bidirectional Streaming RPC（双方向ストリーミング RPC）

クライアントとサーバーが独立したストリームで、互いに非同期でメッセージを送受信します。

```
Client ──[Req 1]──────[Req 2]──────[Req 3]──► Server
Client ◄──[Res 1]──[Res 2]──[Res 3]────────── Server
```

```protobuf
rpc RunningTotal (stream NumberRequest) returns (stream CalcReply);
```

**用途**: チャット、リアルタイムコラボレーション、双方向データ同期、ゲームのリアルタイム通信

---

## gRPC vs REST API

| 比較項目 | gRPC | REST API |
|---------|------|----------|
| **トランスポート** | HTTP/2 | HTTP/1.1（主に）|
| **データ形式** | Protocol Buffers（バイナリ）| JSON（テキスト）|
| **速度** | ⚡ 高速（バイナリ + 多重化）| 比較的遅い |
| **ペイロードサイズ** | 小さい | 大きい |
| **型安全性** | ✅ `.proto` で強制 | ❌ 任意（OpenAPI は別途）|
| **コード生成** | ✅ 自動生成 | ❌ 手動（ツール使用で部分的に可能）|
| **ストリーミング** | ✅ ネイティブサポート | ⚠️ SSE/WebSocket で別途対応 |
| **ブラウザ対応** | ⚠️ grpc-web / プロキシが必要 | ✅ ネイティブ |
| **デバッグ容易性** | ⚠️ バイナリのため難（grpcurl 等を使用）| ✅ 可読テキスト |
| **学習コスト** | やや高い | 低い |
| **主な用途** | マイクロサービス間通信、内部 API | 公開 Web API、ブラウザ向け API |

### どちらを選ぶべきか

```
gRPC が向いているケース:
  ✅ マイクロサービス間の内部通信
  ✅ 高頻度・低レイテンシが求められる通信
  ✅ 多言語環境でのサービス連携
  ✅ ストリーミングデータの処理
  ✅ 型安全性を重視する場合

REST が向いているケース:
  ✅ 外部公開 API（ブラウザから直接利用）
  ✅ シンプルな CRUD 操作
  ✅ チームの学習コストを抑えたい場合
  ✅ 既存の REST エコシステムを活用したい場合
```

---

## チュートリアルの内容

| ステップ | 内容 |
|---------|------|
| Step 1 | ライブラリのインストール（`grpcio`, `grpcio-tools`）|
| Step 2 | `.proto` ファイルの作成（サービス・メッセージ定義）|
| Step 3 | `protoc` によるコード自動生成 |
| Step 4 | サーバーの実装（4 パターンすべて）|
| Step 5 | クライアントからの呼び出し（各パターンを実行）|
| Step 6 | エラーハンドリング（gRPC ステータスコード）|
| Step 7 | メタデータ（認証トークンの送受信）|
| Step 8 | Protocol Buffers のシリアライズ比較（JSON との比較）|
| Step 9 | まとめ |

---

## クイックスタート

### Google Colab で実行（推奨）

1. 上部の **"Open in Colab"** バッジをクリック
2. セルを上から順番に実行（`Shift + Enter`）

### ローカル環境で実行

```bash
# リポジトリをクローン
git clone https://github.com/your-username/grpc-tutorial.git
cd grpc-tutorial

# 依存ライブラリをインストール
pip install grpcio grpcio-tools jupyter

# Jupyter Notebook を起動
jupyter notebook grpc_tutorial.ipynb
```

**動作確認済み環境：**
- Python 3.8 以上
- grpcio 1.50 以上
- Google Colab（推奨）

---

## ディレクトリ構成

```
grpc-tutorial/
├── README.md                # このファイル
├── grpc_tutorial.ipynb      # メインのチュートリアル（Jupyter Notebook）
└── grpc_demo/               # Notebook 実行時に自動生成されるディレクトリ
    ├── demo.proto           # サービス定義ファイル
    ├── demo_pb2.py          # 自動生成：メッセージクラス
    └── demo_pb2_grpc.py     # 自動生成：Stub / Servicer クラス
```

---

## 参考リンク

- [gRPC 公式ドキュメント](https://grpc.io/docs/)
- [Protocol Buffers 言語ガイド（proto3）](https://developers.google.com/protocol-buffers/docs/proto3)
- [gRPC Python チュートリアル](https://grpc.io/docs/languages/python/quickstart/)
- [HTTP/2 仕様（RFC 7540）](https://datatracker.ietf.org/doc/html/rfc7540)
- [CNCF gRPC プロジェクトページ](https://www.cncf.io/projects/grpc/)
- [grpcurl（コマンドラインデバッグツール）](https://github.com/fullstorydev/grpcurl)

---

## ライセンス

[MIT License](LICENSE)
