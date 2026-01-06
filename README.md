# dify-insight-workflow

このワークフローのゴールは、「散らばった最新情報を自動で拾い上げ、ビジネスに役立つ示唆（インサイト）に変えて通知すること」です。

---

## 1. 全体コンセプト

特定のキーワードや競合サイトから最新情報を収集し、単なる要約ではなく「自社にとっての意味合い」を付加して、指定したチャンネル（Slack/Teams等）へ配信する。

## 2. ワークフロー構成（ロジック）

1. **インプット・ソース (Data Sources)**
* **Google News RSS:** 特定キーワード（例：「生成AI 業界動向」）の最新記事。
* **特定のWebサイト:** 競合他社のプレスリリース掲載ページなど。


2. **データ抽出・フィルタリング (Processing)**
* 過去24時間以内の記事のみに絞り込み。
* HTTPリクエストを使用して、記事タイトルとURL、概要を取得。


3. **AIインサイト生成 (Intelligence)**
* **要約:** 記事の内容を3行で要約。
* **重要度判定:** ビジネスへの影響度を低・中・高で判定。
* **ネクストアクション提案:** 「このニュースを受けて、我々が検討すべきこと」をAIが提案。


4. **アウトプット (Output)**
* 整形されたMarkdownテキストを生成。
* Webhook経由でSlack/Teamsへ通知。



---

## 3. 入力パラメータ (Variables)

Difyの開始ノードで設定する項目です。

| 変数名 | 型 | 内容 |
| --- | --- | --- |
| `target_keywords` | String | 収集したいキーワード（例：生成AI, SaaS） |
| `insight_focus` | String | 分析の切り口（例：技術動向重視、競合の価格戦略重視） |
| `notification_channel` | String | 通知先のWebhook URL |

---

## 4. プロンプト設計（核となる部分）

LLMノードに設定するシステムプロンプトの構成案です。

> **役割:** あなたは戦略コンサルタント兼リサーチアシスタントです。
> **タスク:** 収集されたニュース群から、ユーザーが設定した「分析の切り口」に基づき、実務に役立つインサイトを抽出してください。
> **出力フォーマット:**
> 1. 【ニュース概要】
> 2. 【なぜ重要か】（ビジネス視点での分析）
> 3. 【推奨アクション】（自社がとるべき一歩）
> 
> 

---

## 5. 拡張性の検討（今後の練習項目）

* **多言語対応:** 海外の最新論文やTechニュースを翻訳して要約。
* **RAG連携:** 社内の既存戦略資料と照らし合わせ、「既知の情報か新規の情報か」を判別。
* **データベース保存:** Notion等に蓄積し、週報を自動作成。
* **ファクトチェック:** 情報の不確からしさを判定して確信度を出す。

---

## 6. Difyワークフロー実装手順

Difyのコミュニティ版（またはクラウド版）を利用して、このワークフローを構築する具体的な手順です。

### 1. アプリケーションの作成
1.  Difyにログインし、**「スタジオ」**へ移動します。
2.  **「アプリを作成」**をクリックし、「**ワークフロー**」を選択します。
3.  アプリ名（例: `Insight Workflow`）と説明を入力して作成します。

### 2. 「開始」ノードの設定
ワークフロー編集画面の最初の「**開始**」ノードをクリックし、`README.md`の「3. 入力パラメータ (Variables)」で定義した変数を追加します。

*   **「変数を追加」** をクリックして、以下の3つを追加してください。
    *   **変数名:** `target_keywords`, **型:** `String`
    *   **変数名:** `insight_focus`, **型:** `String`
    *   **変数名:** `notification_channel`, **型:** `String`

### 3. Codeノードで最新情報を収集
Difyの標準機能だけでは外部サイトの情報を柔軟に取得しづらいため、**Code（コードを実行）**ノードを使います。

1.  「開始」ノードの `+` をクリックし、**「Code」**ノードを追加します。
2.  Codeノードの入力として、開始ノードの `target_keywords` を受け取るように設定します。
    *   Codeノードの **「変数を追加」** をクリックします。
    *   変数名（例: `keywords`）を入力し、右側の `{{ }}` アイコンから **`#start.target_keywords`** を選択します。
3.  以下のPythonコードをCodeノードに貼り付けます。
    *   このコードは、Google NewsのRSSフィードから、指定されたキーワードの記事を検索し、タイトル、リンク、概要をリスト形式で出力します。

    ```python
    import feedparser
    import json
    from datetime import datetime, timedelta

    # DifyのCodeノードの入力からキーワードを取得
    keyword = keywords # Codeノードの入力変数名に合わせる

    # Google News RSSのURLを構築
    rss_url = f"https://news.google.com/rss/search?q={keyword}&hl=ja&gl=JP&ceid=JP:ja"

    # RSSフィードを解析
    feed = feedparser.parse(rss_url)

    # 24時間以内の記事をフィルタリング
    articles = []
    yesterday = datetime.now() - timedelta(days=1)

    for entry in feed.entries:
        # 公開日時を取得（タイムゾーン情報を除去）
        published_time = datetime(*entry.published_parsed[:6])
        
        if published_time >= yesterday:
            articles.append({
                "title": entry.title,
                "link": entry.link,
                "summary": entry.summary
            })

    # 結果をJSON形式の文字列として出力変数に設定
    # この 'outputs' はDifyのCodeノードで定義された出力オブジェクトです
    outputs = {"articles_json": json.dumps(articles, ensure_ascii=False)}
    ```
4.  Codeノードの出力変数を定義します。
    *   コードエリアの下にある**「出力を追加」**をクリックします。
    *   変数名 `articles_json`、型 `String` を設定します。

### 4. LLMノードでインサイトを生成
1.  Codeノードの `+` をクリックし、**「LLM」**ノードを追加します。
2.  **「プロンプト」**タブの**「システムプロンプト」**に、「4. プロンプト設計」の内容を貼り付けます。
3.  **「変数」** セクションで、`{{ }}` アイコンをクリックし、以下の2つを変数として挿入します。
    *   **`#start.insight_focus`**: 分析の切り口
    *   **`#code.articles_json`**: Codeノードが出力した記事リスト

### 5. HTTPリクエストノードで通知
1.  LLMノードの `+` をクリックし、**「HTTPリクエスト」**ノードを追加します。
2.  **URL**欄に、`{{ }}` アイコンから **`#start.notification_channel`** を選択します。
3.  **Method**を `POST` に、**Headers**の `Content-Type` を `application/json` にします。
4.  **Body**を `raw` に設定し、通知したい形式でJSONを作成します。`{{ }}` アイコンから **`#llm.text`** (LLMノードの出力) を挿入します。
    *   **Slackへの通知例:**
        ```json
        {
          "text": "{{#llm.text}}"
        }
        ```

### 6. 「終了」ノード
最後に、HTTPリクエストノードの出力を「**終了**」ノードに接続して、ワークフローを完成させます。
