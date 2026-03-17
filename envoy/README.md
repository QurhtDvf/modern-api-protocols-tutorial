# Envoy Proxy Demo on Google Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/YOUR_REPO/blob/main/envoy_demo.ipynb)
[![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Envoy](https://img.shields.io/badge/Envoy-v1.30.1-FF6C37?logo=envoy-proxy&logoColor=white)](https://www.envoyproxy.io/)
[![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)](https://jupyter.org/)
[![CNCF](https://img.shields.io/badge/CNCF-Graduated-231F20?logo=cncf&logoColor=white)](https://www.cncf.io/projects/envoy/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Google Colab 上で **Envoy Proxy** の動作を体験できる Jupyter Notebook です。
Docker 不要で、Envoy バイナリを直接インストールして以下をデモします。

- HTTP プロキシとしての基本動作
- ラウンドロビンによるロードバランシング
- Admin API によるリアルタイム統計確認
- バックエンド障害時のフェイルオーバー

---

## 目次

1. [デモの内容](#デモの内容)
2. [クイックスタート](#クイックスタート)
3. [Envoy の歴史](#envoy-の歴史)
4. [Envoy の主要機能](#envoy-の主要機能)
5. [アーキテクチャ](#アーキテクチャ)
6. [参考リンク](#参考リンク)

---

## デモの内容

```
Client (Python)
      │
      ▼
┌─────────────┐
│ Envoy :10000│  ← フロントプロキシ
└──────┬──────┘
       │  ラウンドロビン
  ┌────┴────┐
  ▼         ▼
:8081    :8082   ← Python バックエンドサーバー
```

| ステップ | 内容 |
|---|---|
| 1. インストール | GitHub Releases からバイナリを直接ダウンロード |
| 2. バックエンド起動 | Python HTTP サーバーを 2 台起動 (Backend-A / B) |
| 3. 設定作成 | YAML でリスナー・クラスター・Admin API を定義 |
| 4. Envoy 起動 | `subprocess` でバックグラウンド実行 |
| 5. LB デモ | 10 リクエストがラウンドロビンで均等分散されることを確認 |
| 6. Admin API | `/stats` でメトリクス・アクセスログを確認 |
| 7. フェイルオーバー | Backend-A 停止後も Backend-B が継続応答することを確認 |
| 8. クリーンアップ | 全プロセスを停止 |

---

## クイックスタート

### Google Colab で実行する場合

1. 上部の **"Open in Colab"** バッジをクリック、または `envoy_demo.ipynb` を [Google Colab](https://colab.research.google.com) にアップロード
2. **「ランタイム」→「すべてのセルを実行」** を選択
3. セル 1 の実行に 1〜2 分かかります（バイナリのダウンロード）

### ローカルで実行する場合

```bash
# リポジトリをクローン
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO

# Jupyter を起動
pip install notebook
jupyter notebook envoy_demo.ipynb
```

> **注意**: ローカル実行時は Linux (x86_64) 環境が必要です。macOS では Envoy バイナリの URL を適宜変更してください。

### Envoy バイナリのバージョンを変更したい場合

Notebook の最初のセルで `ENVOY_VERSION` を書き換えてください。

```python
ENVOY_VERSION = "1.30.1"  # ← ここを変更
```

利用可能なバージョンは [Envoy Releases](https://github.com/envoyproxy/envoy/releases) から確認できます。

---

## Envoy の歴史

### 誕生の背景 — Lyft のマイクロサービス移行 (2015)

Envoy は、ライドシェアリングサービスの **Lyft** が直面した現実的な問題を解決するために生まれました。

2015 年初頭、Lyft は PHP によるモノリシックアプリケーションから マイクロサービスアーキテクチャへの移行を進めており、すでに 30 以上のサービスが存在していました。しかし、移行を進めるにつれて深刻な問題が浮かび上がりました。**ネットワークの信頼性と可観測性の欠如**です。

サービス間で問題が起きても「どこで何が起きているのか」がわからない。Lyft のエンジニアたちはサービス呼び出しを恐れるようになっていました。NGINX や HAProxy といった既存のプロキシは、L4/L7 ルーティングの柔軟性と言語横断的な観測性の両面で、Lyft のニーズを満たしていませんでした。

そこで Matt Klein（後の Envoy の生みの親）が 2015 年 5 月に Lyft に入社し、「Lyft proxy」という名前で新しいネットワーキングシステムの開発を提案しました。実装言語については Python も候補に上がりましたが、パフォーマンス上の理由から **C++** が採用されました。

> "The network should be transparent to applications. When network and application problems do occur, it should be easy to determine the source of the problem."
> — Matt Klein, Envoy の設計目標

プロジェクト名は、devops の初期設定を進める中でチームメンバーの Ryan Lane が提案した「**Envoy**（使節・護衛）」という名前に決まりました。

### 初期デプロイから全社展開へ (2015〜2016)

開発開始から約 3〜4 ヶ月で MVP が完成し、2015 年 9 月初旬に Lyft のエッジプロキシとして最初にデプロイされました。最初はモノリスとエッジの間の観測性向上を目的としていましたが、その価値はすぐに証明されました。

その後チームはアウトライヤー検知、ヘルスチェック、リトライ、サーキットブレーカーといった信頼性機能を次々と追加。2016 年初頭には、Envoy は Lyft 全体のネットワーク通信（エッジ、サービス間通信、データベース、外部パートナーへのアクセスまで）を担うようになり、負荷起因の大規模インシデントは激減しました。

### オープンソース化 (2016年9月)

2016 年夏、チームはオープンソース化の議論を始めました。「Envoy は Lyft のコアビジネスではない。同じ課題を抱えるすべての組織に還元しよう」という判断でした。同年 9 月 14 日、Envoy は正式に OSS として公開されます。

公開直後から業界の反応は想像をはるかに超えるものでした。Airbnb、eBay、Google、Pinterest、Salesforce など大企業が次々と採用を表明し、コントリビューターも急増しました。

### CNCF への参加と卒業 (2017〜2018)

- **2017 年 9 月**: Envoy は CNCF（Cloud Native Computing Foundation）の インキュベーティングプロジェクトとして参加（Kubernetes に続く 11 番目のプロジェクト）
- **2017 年 5 月**: Google・IBM・Lyft が共同で **Istio** をオープンソース化。Envoy をデータプレーンプロキシとして採用
- **2018 年 11 月**: Kubernetes に次ぐ CNCF の **2 番目の卒業プロジェクト**（Graduated Project）として認定。成熟度・コントリビューターの幅・採用規模が評価された

### Envoy Gateway の登場 (2022〜現在)

サービスメッシュ用途での普及が進む一方、API ゲートウェイとしての活用にはハードルの高さが指摘されていました。2022 年、この課題に応える形で **Envoy Gateway** プロジェクトが発表されました。既存の CNCF API ゲートウェイプロジェクト（Contour・Emissary）を統合し、Kubernetes Gateway API をベースにした簡易なデプロイモデルを提供します。

今日、Envoy は AWS、Google Cloud、Microsoft Azure を含むすべての主要パブリッククラウドプロバイダーに採用されており、数百万リクエスト/秒を処理するプロダクション環境で稼働しています。

---

## Envoy の主要機能

### 1. アーキテクチャ — アウトオブプロセス設計

Envoy の最も重要な設計思想のひとつが「**アウトオブプロセス（Out-of-Process）**」アーキテクチャです。

アプリケーションと別プロセスとして隣（サイドカー）で動作するため、アプリケーションの実装言語（Go、Java、Python、Node.js など）に依存しません。サービスは「ローカルの Envoy と話す」だけでよく、Envoy がネットワークの複雑さをすべて引き受けます。これにより、ポリグロット（多言語）な環境でも一貫したネットワーク機能を提供できます。

### 2. プロトコルサポート

Envoy は幅広いプロトコルに対応しています。

- **HTTP/1.1 および HTTP/2**: 双方向に完全対応。HTTP/1.1 ↔ HTTP/2 の透過的なブリッジ機能も持ちます
- **gRPC**: ファーストクラスサポート。HTTP/1.1 ↔ gRPC のブリッジも可能
- **TCP / UDP**: L3/L4 レベルの汎用プロキシとして機能
- **MongoDB、Redis、Thrift**: ワイヤーレベルのプロトコル解析と観測性

### 3. ロードバランシング

Envoy はリッチなロードバランシングアルゴリズムを内蔵しています。

| アルゴリズム | 説明 |
|---|---|
| `ROUND_ROBIN` | 順番に均等分散（本デモで使用） |
| `LEAST_REQUEST` | 最もリクエスト数が少ないホストへ |
| `RANDOM` | ランダムに選択 |
| `RING_HASH` | 一貫性ハッシュ（セッションアフィニティ） |
| `MAGLEV` | Google 開発の高性能一貫性ハッシュ |

加えて、**自動リトライ**、**サーキットブレーカー**、**アウトライヤー検知**（異常なホストの自動排除）、**ゾーンアウェアロードバランシング**（レイテンシ最小化）などの高度な機能も備えています。

### 4. 可観測性（Observability）

Envoy の設計目標の中心にあるのが可観測性です。Envoy 自体が豊富なテレメトリを自動的に生成します。

**メトリクス**: Prometheus 互換の統計情報を Admin API (`/stats`) または `/stats/prometheus` エンドポイントで取得可能。リクエスト数、レイテンシのパーセンタイル、接続数、エラー率などが含まれます。

**分散トレーシング**: Zipkin、Jaeger、AWS X-Ray、Datadog などのトレーシングシステムとネイティブに統合できます。サービス間をまたがるリクエストの流れをエンドツーエンドで可視化できます。

**アクセスログ**: 柔軟なフォーマットで各リクエストの詳細をロギング。stdout、ファイル、gRPC などへの出力に対応しています。

### 5. 動的設定 — xDS API

Envoy の最大の技術的特徴のひとつが **xDS（Extensible Discovery Service）API** です。

従来のプロキシがファイルの再読み込みやプロセス再起動によって設定変更を適用するのに対して、Envoy は gRPC ストリーミングを通じてコントロールプレーンからリアルタイムに設定を受け取ります。これにより**ダウンタイムゼロでの設定変更**が可能になります。

xDS は以下のサービス群から構成されています。

| API 名 | 略称 | 役割 |
|---|---|---|
| Listener Discovery Service | LDS | リスナー設定の動的更新 |
| Route Discovery Service | RDS | ルーティングルールの動的更新 |
| Cluster Discovery Service | CDS | クラスター（バックエンドグループ）の動的更新 |
| Endpoint Discovery Service | EDS | 各クラスター内のエンドポイント（ホスト）の動的更新 |
| Aggregate Discovery Service | ADS | 上記すべてを単一 gRPC ストリームで集約 |

Istio の istiod はこの xDS API を実装したコントロールプレーンであり、Kubernetes の変更を検知して全サイドカー Envoy に設定を動的配布しています。xDS は今や CNCF API ワーキンググループによって、L4/L7 データプレーン設定の事実上の標準として推進されています。

### 6. セキュリティ

- **mTLS（相互 TLS）**: サービス間通信の暗号化と相互認証。サービスメッシュでのゼロトラストネットワーク実現の基盤
- **外部認証・認可**: ext_authz フィルターにより、OPA（Open Policy Agent）など外部サービスに認可判断を委譲可能
- **レート制限**: ローカルレート制限（`local_ratelimit` フィルター）および外部レート制限サービスとの統合

### 7. トラフィック管理

- **ヘルスチェック**: アクティブ（Envoy からポーリング）とパッシブ（レスポンスを監視）の両方式に対応。異常なホストを自動的にクラスターから除外
- **カナリーリリース / A/B テスト**: 重み付けルーティングにより、新バージョンへのトラフィックを段階的に切り替え可能
- **リクエストシャドウイング（ミラーリング）**: 本番トラフィックのコピーをテスト環境に送信して、影響なしに新バージョンを検証
- **フォールト・インジェクション**: 意図的に遅延やエラーを注入し、カオスエンジニアリングを実施可能
- **タイムアウト / リトライ**: きめ細かいタイムアウト設定と、再試行ポリシー（回数、条件、バックオフ）の制御

### 8. フィルターチェーンアーキテクチャ

Envoy の拡張性の核心は**フィルターチェーン**です。受信・送信リクエストは一連のフィルターを通過し、各フィルターが認証、レート制限、プロトコル変換、ロギングなどの処理を担います。

標準的なフィルターに加えて、**WebAssembly（Wasm）** を使ったカスタムフィルターの組み込みもサポートしており、高い拡張性を持ちます。

### 9. Admin API

`:9901` で公開される管理 API（本デモでも使用）は、Envoy の内部状態をリアルタイムで確認・操作するためのインターフェースです。

| エンドポイント | 説明 |
|---|---|
| `/stats` | 全統計情報（Prometheus 形式も可） |
| `/clusters` | クラスターとエンドポイントの状態 |
| `/config_dump` | 現在の設定全体をダンプ |
| `/listeners` | アクティブなリスナー一覧 |
| `/ready` | Envoy の起動完了確認（ヘルスチェック用） |
| `/drain_listeners` | グレースフルシャットダウン開始 |

---

## アーキテクチャ

### サイドカーパターン（サービスメッシュ）

```
┌─────────────────── Pod A ───────────────────┐
│  ┌───────────────┐     ┌──────────────────┐ │
│  │  Application  │────▶│  Envoy (sidecar) │ │
│  └───────────────┘     └────────┬─────────┘ │
└───────────────────────────────  │ ───────────┘
                                  │ mTLS
┌─────────────────── Pod B ───────┼───────────┐
│  ┌───────────────┐     ┌────────▼─────────┐ │
│  │  Application  │◀────│  Envoy (sidecar) │ │
│  └───────────────┘     └──────────────────┘ │
└─────────────────────────────────────────────┘
         ▲ xDS (gRPC)  ▲ xDS (gRPC)
         └──────────────┘
           コントロールプレーン
           (Istio istiod など)
```

### エッジプロキシ / API ゲートウェイ

```
Internet
    │
    ▼
┌──────────────────────────────┐
│  Envoy (Edge / Ingress)      │
│  - TLS 終端                  │
│  - 認証・認可                │
│  - レート制限                │
│  - ルーティング              │
└───────────┬──────────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
Service A       Service B
(sidecar)       (sidecar)
```

---

## 参考リンク

- [Envoy 公式ドキュメント](https://www.envoyproxy.io/docs)
- [Envoy GitHub リポジトリ](https://github.com/envoyproxy/envoy)
- [Envoy Releases（バイナリ一覧）](https://github.com/envoyproxy/envoy/releases)
- [5 years of Envoy OSS — Matt Klein](https://mattklein123.dev/2021/09/14/5-years-envoy-oss/) — Envoy の歴史を語った作者本人のブログ
- [CNCF Envoy Graduation アナウンス](https://www.cncf.io/announcements/2018/11/28/cncf-announces-envoy-graduation/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [xDS Protocol ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

---

## ライセンス

このデモノートブックは MIT ライセンスのもとで公開しています。
Envoy 本体は [Apache License 2.0](https://github.com/envoyproxy/envoy/blob/main/LICENSE) です。
