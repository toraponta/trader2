# MCP (Model Context Protocol) サーバー要件仕様

本ドキュメントでは、AI投資エージェントシミュレーターが自律的に市場情報の収集やデータ操作を行うために必要な、MCPサーバーの構成と提供ツール（Functions）を定義します。

## 1. 基本方針

* **責務の分離**: 株価取得、外部Web検索、DB操作など、役割ごとに独立したMCPサーバーとして定義します。
* **ローカルと外部APIの使い分け**: データベースや株価履歴などの確実なファクトデータはローカル自作のMCPサーバーから提供し、サプライチェーンの推論や最新ニュースは外部の検索特化型API（Tavily等）に委譲します。

---

## 2. 必要なMCPサーバーとツール群

### 2.1 Tavily Research MCP (外部連携 / 自作ラッパー)
**目的**: 最新の市場ニュース、テーマごとの銘柄発掘、動的オントロジー（サプライチェーン等）のリアルタイム推論。
**実装方式**: AIエージェント向け検索APIである **Tavily API**（無料枠: 月1000回）をラップしたMCPサーバー。

| ツール名 (Tool) | 引数 (Arguments) | 説明 |
| :--- | :--- | :--- |
| `ask_market_ontology` | `query` (str) | 「トヨタ自動車の主要なサプライヤーは？」といったクエリに対し、Web検索を通じて企業間の関係性や構造を推論し、銘柄リストを返します。 |
| `search_financial_news` | `ticker` (str) | 指定銘柄に関する最新の経済ニュース、適時開示情報のサマリーを検索して返します。 |
| `search_stocks_by_theme` | `theme` (str) | 「円安メリット」「AI関連」などのマクロテーマやトレンドに基づき、有望な関連銘柄群をWebから抽出します。 |
| `get_earnings_schedule` | `ticker` (str) | 指定銘柄の次回の決算発表予定日をWeb検索から抽出して返します。AIが決算またぎのリスクを回避する判断を行うために使用します。 |

### 2.2 Market Data MCP (ローカル自作)
**目的**: 確実な時系列株価データ、テクニカル指標、マクロ市場指数の提供。
**実装方式**: プロジェクト内に内包されるPython製のローカルMCPサーバー（`yfinance` などのライブラリを利用）。

| ツール名 (Tool) | 引数 (Arguments) | 説明 |
| :--- | :--- | :--- |
| `get_historical_prices` | `ticker` (str), `days` (int) | 指定銘柄の過去N日間の四本値（始値・高値・安値・終値）および出来高を取得します。 |
| `get_market_sentiment` | なし | 日経平均、TOPIX、マザーズ指数などの主要インデックスの直近の推移と、全体的な市場の強弱感を取得します。 |
| `get_company_profile` | `ticker` (str) | 指定銘柄の事業内容、時価総額、PER、PBRなどの基本的な財務・企業情報を取得します。 |

### 2.3 Portfolio & Memory MCP (ローカル自作)
**目的**: AIが自身の実績（過去の取引や反省）を振り返る機能と、自律的に監視対象を更新する機能。
**実装方式**: バックエンドのSQLiteデータベース(`trader.db`)と直接通信するローカルMCPサーバー。

| ツール名 (Tool) | 引数 (Arguments) | 説明 |
| :--- | :--- | :--- |
| `get_past_trade_history` | `ticker` (str) | その銘柄に対する「過去の取引結果（勝敗・損益）」と、当時の「推論ログ・反省内容（`reflection_text`）」を引き出します。同じ失敗を繰り返さないための長期記憶として機能します。 |
| `manage_watchlist` | `action` (add/remove), `ticker` (str), `reason` (str), `tags` (list) | AIが自らの判断でウォッチリストに銘柄を追加（または削除）します。追加理由は `ai_discovery_logs` に記録されます。 |

### 2.4 Document Analyzer MCP (ローカル自作)
**目的**: 決算短信や有価証券報告書などの複雑なPDFドキュメントの解析。
**実装方式**: PDF解析ライブラリ（`pdfplumber` 等）と、抽出処理用の軽量LLM（Gemini Flash等）を組み合わせたローカルMCPサーバー。

| ツール名 (Tool) | 引数 (Arguments) | 説明 |
| :--- | :--- | :--- |
| `analyze_company_pdf` | `url_or_path` (str), `query` (str) | 指定されたPDF文書を読み込み、クエリに対する回答（例：「今期の純利益見通しの上方修正はあるか？」）をピンポイントで抽出して返します。 |

### 2.5 J-Quants API MCP (JPX公式連携 / 推奨オプション)
**目的**: 決算発表予定日および決算短信の公式な財務数値（売上・利益・業績予想など）の確実な取得、および高度な需給分析。
**実装方式**: 日本取引所グループ（JPX）が提供する公式API「**J-Quants API**」を叩くローカルMCPサーバー。PDF解析の不確実性を排除し、正確なファクトデータをAIに提供します。

| ツール名 (Tool) | 引数 (Arguments) | 説明 |
| :--- | :--- | :--- |
| `get_jquants_earnings_schedule` | `ticker` (str) | 【Freeプラン対応】JPX公式の決算発表予定日データを取得します。 |
| `get_jquants_financial_statements` | `ticker` (str) | 【Freeプラン対応】決算短信ベースの正式な財務数値（売上、利益、次期業績予想・配当予想）をJSONで取得します。 |
| `get_jquants_investor_flows` | なし | 【Lightプラン以上: 月額1,650円】「海外投資家」「個人」などの投資部門別売買状況（週次）を取得し、市場全体への資金流入トレンドを把握します。 |
| `get_jquants_margin_balance` | `ticker` (str) | 【Standardプラン以上: 月額3,300円】個別銘柄の「信用買い残・売り残」を取得し、将来の売り圧力（しこり玉）や需給の過熱感を分析します。 |
| `get_jquants_short_interest` | `ticker` (str) | 【Standardプラン以上: 月額3,300円】機関投資家の空売り残高データを取得し、踏み上げ（ショートスクイーズ）狙いなどの需給トレードに活用します。 |
| `get_jquants_full_financials` | `ticker` (str) | 【Premiumプラン限定: 月額16,500円】貸借対照表(BS)、損益計算書(PL)、キャッシュフロー計算書(CF)の詳細な全項目データを取得し、精緻なファンダメンタルズ分析を行います。 |

---

## 3. 設定と統合 (DBスキーマとの連携)

これらのMCPサーバーは、`mcp_servers` テーブルに登録され、バックエンドの実行時に動的に読み込まれます。

*   **Tavily API キーの管理**: 
    Tavily MCPの起動に必要なAPIキーは、データベースの `mcp_servers.env_vars` カラムにJSON形式（例: `{"TAVILY_API_KEY": "tvly-xxx"}`）で保存し、Web画面からセキュアに設定できるようにします。
*   **ON/OFFの切り替え**: 
    各MCPサーバーは `is_active` フラグを持つため、「今日はニュース検索（Tavily）はオフにして、純粋に株価チャート（Market Data）だけでテクニカルトレードをさせる」といったエージェントごとの実験がUI上から容易に行えます。
