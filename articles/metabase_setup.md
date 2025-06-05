---
title: "Metabaseをサンプルデータ付で簡単に試す方法"
emoji: "📊"
type: "idea"
topics: ["metabase", "データ分析", "可視化", "docker"]
published: false
publication_name: "yaoko_tech_blog"
---

## はじめに

データドリブンな意思決定が求められる現代において、ビジネスインテリジェンス（BI）ツールの導入は企業にとって重要な課題となっています。
しかし、本格的なBI環境の構築には専門知識が必要で、導入のハードルが高いと感じている方も多いのではないでしょうか。

本記事では、オープンソースのBIツール「Metabase」とPostgreSQLを使用して、ECサイトのサンプルデータを分析できる環境をDocker Composeで簡単に構築する方法を紹介します。

## この記事で構築する環境の特徴

### 1. ワンコマンドで起動可能
Docker Composeを使用することで、複雑な環境構築作業を自動化。`docker compose up -d`コマンド一つで、以下の環境が立ち上がります：

- **Metabase**: 直感的なUIでデータ分析が可能なBIツール
- **PostgreSQL**: サンプルデータを格納するリレーショナルデータベース

### 2. 実践的なECサイトデータ
『10年戦えるデータ分析入門』で使用されているサンプルデータを利用しています。
顧客、商品、注文、アクセスログなど、実際のビジネスシーンを想定したデータセットでMetabaseを試すことができます。
データセットに関しては公式のページを参照してください：[10年戦えるデータ分析入門](https://i.loveruby.net/stdsql/)

## アーキテクチャ図

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Browser   │────▶│   Metabase   │────▶│  PostgreSQL  │
│             │     │  (Port 3000) │     │  (Port 5432) │
└─────────────┘     └──────────────┘     └──────────────┘
                           │                      │
                           └──────────────────────┘
                              Docker Network
                              (metanet1)
```


## 環境構築の手順

### 1. ファイルの準備
まず、以下のファイル構成でディレクトリを作成します：

```
project-root/
├── docker-compose.yml
├── init-db.sh
├── create_db.sh
└── seeds/
    ├── access_log.csv
    ├── customers.csv
    ├── items.csv
    ├── order_details.csv
    ├── orders.csv
    ├── shops.csv
    └── web_pages.csv
```

### 2. Docker Composeファイル
以下の`docker-compose.yml`ファイルを使用して、MetabaseとPostgreSQLの環境を構築します。

```yaml docker-compose.yml
services:
  metabase:
    image: metabase/metabase:latest
    container_name: metabase
    hostname: metabase
    volumes:
      - /dev/urandom:/dev/random:ro
    ports:
      - 3000:3000
    environment:
      MB_DB_TYPE: postgres
      MB_DB_DBNAME: metabaseappdb
      MB_DB_PORT: 5432
      MB_DB_USER: metabase
      MB_DB_PASS: mysecretpassword
      MB_DB_HOST: postgres
    networks:
      - metanet1
    healthcheck:
      test: curl --fail -I http://localhost:3000/api/health || exit 1
      interval: 15s
      timeout: 5s
      retries: 5
  postgres:
    image: postgres:latest
    container_name: postgres
    hostname: postgres
    environment:
      POSTGRES_USER: metabase
      POSTGRES_DB: metabaseappdb
      POSTGRES_PASSWORD: mysecretpassword
    volumes:
      - ./init-db.sh:/docker-entrypoint-initdb.d/init-db.sh
      - ./create_db.sh:/tmp/create_db.sh
      - ./seeds:/tmp/seeds
    networks:
      - metanet1
    ports:
      - 5432:5432
networks:
  metanet1:
    driver: bridge

```

### 3. 初期化スクリプト

PostgreSQLの初期化処理を自動化するため、以下の2つのスクリプトを作成します。

#### init-db.sh
このスクリプトは、PostgreSQLコンテナの起動時に自動実行されます。

```bash init-db.sh
#!/bin/bash
set -e

# 初回セットアップかどうかを確認（shopデータベースが存在するかチェック）
if ! psql -U metabase -lqt | cut -d \| -f 1 | grep -qw shop; then
    echo "First time setup detected. Running initialization script..."
    /tmp/create_db.sh
    echo "Initialization completed!"
else
    echo "Shop database already exists. Skipping initialization."
fi

```

#### create_db.sh
実際のデータベース作成とデータインポートを行うスクリプトです。

```bash create_db.sh
#!/bin/sh

# エラーが発生したら即座に終了
set -e

# PostgreSQLのパスワードを環境変数に設定
export PGPASSWORD=mysecretpassword

# psqlコマンドでshopデータベースを作成する
echo "Creating shop database..."
psql -U metabase -d postgres -c 'CREATE DATABASE shop;'

# データベース作成を確認
if [ $? -eq 0 ]; then
    echo "Database 'shop' created successfully"
else
    echo "Failed to create database 'shop'"
    exit 1
fi

# shopデータベースにテーブルを作成
echo "Creating tables..."
psql -U metabase -d shop <<EOF
-- customersテーブル
CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_age INTEGER,
    customer_birthday DATE,
    customer_gender CHAR(1),
    customer_location VARCHAR(50)
);

-- shopsテーブル
CREATE TABLE shops (
    shop_id INTEGER PRIMARY KEY,
    shop_name VARCHAR(100)
);

-- itemsテーブル
CREATE TABLE items (
    item_id INTEGER PRIMARY KEY,
    shop_id INTEGER,
    item_name VARCHAR(100),
    item_price INTEGER
);

-- ordersテーブル
CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    order_time TIMESTAMP,
    shop_id INTEGER,
    customer_id INTEGER,
    order_amount INTEGER
);

-- order_detailsテーブル
CREATE TABLE order_details (
    order_id INTEGER,
    item_id INTEGER,
    item_qty INTEGER,
    item_price INTEGER
);

-- web_pagesテーブル
CREATE TABLE web_pages (
    request_path VARCHAR(200) PRIMARY KEY,
    page_category VARCHAR(50),
    page_name VARCHAR(100)
);

-- access_logテーブル
CREATE TABLE access_log (
    request_time TIMESTAMP,
    request_method VARCHAR(10),
    request_path VARCHAR(200),
    customer_id INTEGER,
    search_hit INTEGER,
    referer VARCHAR(200)
);
EOF

# CSVファイルをインポート
echo "Importing CSV data..."
psql -U metabase -d shop -c "\copy customers FROM '/tmp/seeds/customers.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy shops FROM '/tmp/seeds/shops.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy items FROM '/tmp/seeds/items.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy orders FROM '/tmp/seeds/orders.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy order_details FROM '/tmp/seeds/order_details.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy web_pages FROM '/tmp/seeds/web_pages.csv' WITH CSV HEADER"
psql -U metabase -d shop -c "\copy access_log FROM '/tmp/seeds/access_log.csv' WITH CSV HEADER"

echo "Data import completed!"

echo "Database setup completed!"
```

### 4. 環境の起動

プロジェクトルートディレクトリで以下のコマンドを実行します：

```bash
docker compose up -d
```

### 5. 動作確認

環境が正常に起動したら、以下のURLにアクセスできます：

- **Metabase**: http://localhost:3000
- **PostgreSQL**: localhost:5432（必要に応じて）

### 6. Metabaseの初期設定

初回アクセス時は、Metabaseの初期設定画面が表示されます。以下の手順で設定を行います：

1. **アカウント作成**: 管理者アカウントを作成
2. **データベース接続**: PostgreSQLの`shop`データベースに接続
   - データベースタイプ: PostgreSQL
   - ホスト: `postgres`
   - ポート: `5432`
   - データベース名: `shop`
   - ユーザー名: `metabase`
   - パスワード: `mysecretpassword`

これで、ECサイトのサンプルデータを使用したダッシュボード作成が可能になります。

## まとめ

Docker Composeを使用することで、Metabaseとサンプルデータ付きのPostgreSQLを簡単に構築できました。この環境を使って、実際のビジネスデータを想定したダッシュボード作成やデータ分析を試すことができます。

本格的なBI環境の導入を検討している方は、まずこの環境でMetabaseの機能を体験してみることをおすすめします。