---
layout: post
title: "【dbt入門】MacBookでdbt + DuckDBを動かして基本を理解する"
date: 2026-09-26 11:59:35 +0900
categories: [データ基盤, dbt]
tags: [dbt, duckdb, mac]
---

最近本屋で見かけた **dbt** という技術ワードに興味が湧いたのですが、ネットで調べてもイマイチピンとこなかったので、手持ちのMacBookで実際に動かして試してみました。

システムのサインイン履歴や、ネットワーク機器の各種ログなどをいい感じに集計し、一箇所に保存して分析する用途に使えればいいな、というモチベーションです。

Python使いなので、今回はパッケージマネージャーに `uv` を使って仮想環境を構築していきます。

---

## 1. 環境構築 (uvを使用)

まずは `uv` を使ってプロジェクトの初期化と、dbtをDuckDBで試すためのモジュールをインストールします。

```bash
# 仮想環境を作成
uv init dbt-test
cd dbt-test

# dbtをduckdbで試せるモジュールを追加
uv add dbt-duckdb

# dbtのプロジェクトを作成
# 実行すると対話プロンプトが出るので、データベースはduckdbを選択
uv run dbt init test
```

初期状態のディレクトリ構成は以下のようになります。

```text
dbt-test % tree
.
├── logs
│   └── dbt.log
├── main.py
├── pyproject.toml
├── README.md
├── test
│   ├── analyses
│   ├── dbt_project.yml
│   ├── dev.duckdb
│   ├── macros
│   ├── models
│   │   └── example
│   │       ├── my_first_dbt_model.sql
│   │       ├── my_second_dbt_model.sql
│   │       └── schema.yml
│   ├── README.md
│   ├── seeds
│   ├── snapshots
│   └── tests
└── uv.lock
```

`dev.duckdb` の中身を見てみると、当然ですがまだテーブルは0件です。

```text
test % duckdb dev.duckdb 
DuckDB v1.5.2 (Variegata)
Enter ".help" for usage hints.
dev D show tables;
┌─────────┐
│  name   │
│ varchar │
└─────────┘
  0 rows 
```

## 2. モデル (SQL) の作成

dbtでは、`models` ディレクトリ配下にSQLファイルを作成していくようです。
とりあえず初期のサンプルは削除してしまいます。

```bash
test % rm -rf models/example 
```

試しに `test/models` 配下に `customers.sql` を作成してみます。

```sql
-- models/customers.sql

with source_data as (
    select 1 as id, 'Alice' as name, 'Tokyo' as city
    union all
    select 2 as id, 'Bob' as name, 'Osaka' as city
    union all
    select 3 as id, 'Charlie' as name, 'Tokyo' as city
),

filtered_data as (
    select *
    from source_data
    where city = 'Tokyo'
)

select * from filtered_data
```

## 3. dbtの実行と結果確認

準備ができたので、`run` コマンドを実行してみます。

```bash
uv run dbt run
```

無事に実行され、`target` ディレクトリ配下などにコンパイルされたファイルが色々と生成されました。

さて、肝心の `dev.duckdb` の中身を再度確認してみます。

```text
test % duckdb dev.duckdb 
DuckDB v1.5.2 (Variegata)
Enter ".help" for usage hints.
dev D show tables;
┌───────────┐
│   name    │
│  varchar  │
├───────────┤
│ customers │
└───────────┘
dev D select * from customers;
┌───────┬─────────┬─────────┐
│  id   │  name   │  city   │
│ int32 │ varchar │ varchar │
├───────┼─────────┼─────────┤
│     1 │ Alice   │ Tokyo   │
│     3 │ Charlie │ Tokyo   │
└───────┴─────────┴─────────┘
```

おお！ `customers.sql` に書いたSELECT文の結果が、DWH（DuckDB）の中にしっかりとテーブルとして作成されています！

## 💡 今回の気づき

なるほど、**「SELECT文さえ定義しておけば、DWH（DuckDB）の中のテーブルやビューの作成・更新（DDL）はdbtがメンテしてくれる」** ということなんですね。

実際に手を動かしてみることで、少しだけdbtのノリが分かりました。