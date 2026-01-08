---
title: "はじめに"
---


## はじめに

このチュートリアルはすべてDocker環境で行うためローカル環境を汚染しません。
最初のベースとなるファイルのみレポジトリに配置されているので、それらをどんどんとdbtを操作していくのがこのチュートリアルの基本的な流れとなっています。

また、dbtに用いるOLAPDBとしてDuckDBを使用しています。
今回はDuckDBの中身まで言及はしませんが、SQLiteのOLAPDB版とざっくり捉えていただいて構いません。
SQLiteのようにサーバを立ち上げなくてもファイルベースで管理できるとても優れたDBです。
wasmで実装されたWebGUIがあるなど、とてもイケているDBなので興味があったら深ぼっていただいても良いかもしれません。私はSnowflakeとDuckDBのハイブリット構成でコスト最適化ができないか日々考えています。

まずは下記のレポジトリよりDockerfileおよびSeedファイルをCloneしてきましょう。

```bash
git clone https://github.com/yo4raw/dbt-tutorial
cd dbt-tutorial
```

Dockerfileとcompose.ymlはよくありがちな構成なので特に解説はしませんが、ご確認いただきたいのはseedsの中にある4ファイルです。

今回seedデータとして準備した3ファイルと、それらの内容を記載したschema.ymlの4つで構成されています。

```mermaid
erDiagram
    CUSTOMERS {
        int customer_id PK
        string first_name
        string last_name
        string email
        string phone
        string address
        string city
        string postal_code
        int age
        string gender
        string membership_level
        date registration_date
    }

    PRODUCTS {
        int product_id PK
        string product_name
        string category
        string subcategory
        decimal price
        decimal cost
        string brand
        string supplier
        string barcode
        date created_date
    }

    SALES_TRANSACTIONS {
        int transaction_id
        int customer_id FK
        int product_id FK
        int quantity
        decimal unit_price
        decimal total_amount
        datetime transaction_date
        string payment_method
        string store_id
        string cashier_id
    }

    CUSTOMERS ||--o{ SALES_TRANSACTIONS : "purchases"
    PRODUCTS ||--o{ SALES_TRANSACTIONS : "sold_in"
```

この構造をまずは把握していただいたうえで早速チュートリアルに入っていきましょう！

下記のコマンドでDockerを立ち上げてsshでDockerの中に入りましょう。

```bash
docker compose up -d
docker compose exec dbt-tutorial ssh
```
