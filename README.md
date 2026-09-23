# Personal Information Collector

技術記事や研究・セキュリティ・インフラ関連情報を自動収集し、後段のAI処理によって「読むべき情報を選びやすくする」ための個人向け情報収集基盤。

## 目的

インターネット上には大量の技術情報が存在するが、

* 情報源を巡回する
* 新着記事を探す
* 記事を開く
* 内容を少し読んで読む価値を判断する

という作業そのものに時間がかかる。

本システムでは、まず情報収集を自動化し、その後JEVやLLMを利用して記事の分類・関連度判定・要約を行うことで、原文を読むまでの負担を減らす。

最終的な目的は「AIが代わりに記事を読むこと」ではなく、**人間が読むべき情報を発見しやすい状態を作ること**である。

## 想定する処理

```text
RSS / Atom / Web / GitHub
           │
           ▼
    Python Collector
           │
           ▼
        SQLite
           │
           ▼
          JEV
  分類・関連度・選別
           │
           ▼
        Hermes
  要約・前提知識・読みどころ
           │
           ▼
 Karakeep / Obsidian
           │
           ▼
   Discord / Digest
```

ただし、第一段階では次の範囲だけを実装する。

```text
RSS / Atom
    │
    ▼
新着記事取得
    │
    ▼
重複排除
    │
    ▼
本文取得
    │
    ▼
 SQLite
```

## 第一段階の完成条件

次の条件を満たした時点でCollector部分を完成とする。

1. 複数のRSS/Atom Feedを設定できる
2. 各Feedから記事情報を取得できる
3. 同じ記事を何度実行しても重複保存しない
4. 記事本文をWebページから取得できる
5. SQLiteへ記事情報を保存できる
6. 一部のFeedや記事取得に失敗しても処理全体が停止しない
7. 実行結果として新規取得件数・重複件数・失敗件数を確認できる

## 将来的な拡張

### JEV

限定された判断を担当する。

* category
* relevance
* worth_reading
* priority

### Hermes / LLM

文章理解・生成が必要な処理を担当する。

* 要約
* 前提知識
* 読むべきポイント
* 記事の位置付け
* Reading Card生成

### 保存・提示

* Karakeep
* Obsidian
* Discord
* Daily Digest

## 技術構成

第一段階では以下を基本構成とする。

* Python
* SQLite
* RSS / Atom
* HTTP client
* HTML本文抽出ライブラリ

n8nやNode-REDはCollector内部には入れず、将来的に定期実行や外部サービスとの接続が必要になった場合に利用する。

## ディレクトリ構成

```text
personal-information-collector/
├── README.md
├── pyproject.toml
├── config/
│   └── feeds.yaml
├── src/
│   └── info_collector/
│       ├── __init__.py
│       ├── main.py
│       ├── models.py
│       ├── database.py
│       ├── feed.py
│       └── content.py
├── tests/
│   ├── test_database.py
│   ├── test_feed.py
│   └── test_content.py
└── data/
    └── .gitkeep
```

## Feed設定

情報源はコードへ直接記述せず、設定ファイルとして管理する。

```yaml
feeds:
  - name: example
    url: https://example.com/feed.xml
    category: software-engineering

  - name: example-security
    url: https://example.com/security/feed.xml
    category: security
```

ここで指定するcategoryは暫定的な「情報源のカテゴリ」であり、将来的なJEVによる記事単位の分類とは分離する。

## データモデル

第一段階では記事を次の情報として保存する。

```text
Article
├── id
├── url
├── canonical_url
├── title
├── source
├── source_category
├── published_at
├── fetched_at
├── content
├── content_fetched_at
├── status
└── content_fetch_status
```

`canonical_url`はURLの正規化後の値で、重複判定に利用する。

`content_fetch_status`は本文取得状態を表し、たとえば以下を使用する。

```text
pending
success
failed
```

## CLI

第一段階では、少なくとも次のコマンドで収集処理を実行できるようにする。

```bash
python -m info_collector
```

実行結果の例:

```text
Feeds checked:       12
Entries discovered:  84
New articles:        7
Duplicates skipped:  74
Failed entries:      3
```

## 設計方針

### CollectorはAIに依存させない

情報収集基盤はJEV、Hermes、その他LLMが停止していても動作できる状態にする。

### 生データを保持する

AI処理結果だけではなく、元記事のURL・メタデータ・本文を保持する。

これにより、将来的に異なるモデルや判定方法を使って再処理できる。

### AI処理と収集処理を分離する

Collectorは、

```text
情報を取得する
```

ことだけを担当する。

JEVやHermesは、

```text
取得済み情報を評価・理解する
```

ことを担当する。

この境界を維持する。
