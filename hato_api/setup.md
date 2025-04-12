# Hato バックエンド運用環境構築

Hato を運営する上で不可欠な、バックエンドのサービス (Hato API) を運用する環境の構築ガイドです。

## 必要なもの

- OS: Windows or Linux (Linux 系が望ましい、macOS での動作は未確認)
- Node.js (v18 以降)
- Docker (v3 以降)
- Git
- GitHub アカウント

## 構築手順

1. Docker のインストール

Docker 公式サイトのインストール手順に従って Docker をインストールします。

> [!WARNING]
> [Linux に Docker をインストールする](https://docs.docker.com/desktop/setup/install/linux/) 場合、ディストリビューションごとにインストール方法が異なるので注意してください。

1. Git のインストール
1. Node.js のインストール
1. yarn の有効化

[フロントエンド開発環境構築ガイド](../hato_front/setup.md) を参照

1. リポジトリのクローン

Git を使い、GitHub 上の Hato API リポジトリをローカルにクローンします。

> [!WARNING]
> Hato API リポジトリはプライベートリポジトリのため、クローンするにはクローン元のアカウントが [Hato-org](https://github.com/hato-org) に所属している必要があります。

```bash
git clone https://github.com/hato-org/hatoapi.git
```

クローンが完了すると、現在のワーキングディレクトリに `hatoapi` という名前のディレクトリが作成されます。`cd hatoapi` で、Hato API のディレクトリに入ります。

1. 環境変数の設定

リポジトリのルートディレクトリには、Hato API を運用する上で必要となる機密性の高い情報 (外部 API キーなど) が保存する `.env` ファイルの雛形である `.env.placeholder` ファイルがあります。このファイルを同じディレクトリに `.env.production` としてコピーします。

その後、ファイル内の変数に値を設定します。**この値については、引き継ぎ等の前任者に直接確認する形で設定し、<ins>インターネット上や外部へ流出することが絶対にないようにしてください。</ins>**

1. Hato API 関連ファイルのコピー

Hato サービス上で扱われるデータは、主に MongoDB 上と 実ファイル (JSON ファイル) の主に2つの手段を用いて保存されています。MongoDB 上のデータの復元は Docker コンテナの起動後に行うため、実ファイルで管理されているデータを先にコピーして復元します。

これらのファイル群は環境変数と同じく、引き継ぎ等の前任者から何らかの手段で受け取ってください。その後、Hato API のルートディレクトリに `data` という名前で配置してください。

```text
hatoapi/ (Root dir)
│
├─ data/
│  ├─ auth/
│  ├─ classmatch/
│  └─ ...
│
├─ ...
├─ .env
├─ compose.yml
├─ turbo.json
└─ ...
```

上のようなディレクトリ構造になっていれば大丈夫です。

1. Docker イメージのビルド

Hato API は複数のサーバー・データベース等が並行して動作しており、それぞれが Docker コンテナ上で動くよう設計されています。

| コンポーネント名 | Docker イメージ名 | 役割 |
| - | - | - |
| API サーバー | `hatoapi-prod-api-1` | Hato のフロントエンド (Web ページ) から送信される HTTP リクエストを受け取り、データを読み書きしレスポンスを返す。 |
| クローラー | `hatoapi-prod-crawler-1` | Hato の乗り換え案内・年間行事予定で必要となるデータを Web や外部 API から定期的に取得する。 |
| プッシュ通知サーバー | `hatoapi-prod-push-server-1` | Hato で提供されているプッシュ通知 (Web Push) のためのサーバー。各イベントに応じてクライアントにプッシュ通知リクエストを送信する。 |
| 永続的データベース (MongoDB) | `hatoapi-prod-mongo-1` | Hato 上で利用される様々なデータを保存するデータベース。ユーザーデータ、時間割、年間行事予定などの永続的に使用されるデータが入っている。 |
| 一時的データベース (Redis) | `hatoapi-prod-redis-1` | Hato 上で利用されるデータのうち、短期間で書き換わるデータや負荷対策のためのキャッシュに利用されるデータベース。定期的にクローラーによって更新される乗換案内や、最近クライアントからアクセスされた API エンドポイントのレスポンスを保存している。 |

これら複数のコンポーネントの動作環境を一つずつ揃え、別々に管理するのはかなり大変なため、Hato では Turborepo / Docker Compose を用いてまとめて起動・監視・終了ができるようになっています。

ルートディレクトリでターミナルを起動し、Docker が起動していることを確認してから、以下のコマンドを実行します。

```bash
docker compose -f compose.production.yml --env-file .env.production -p hatoapi-prod build
```

> [!NOTE]
> 上のコマンドのオプションについて解説します。
>
> - `-f compose.production.yml`
>   - Docker Compose の構成について書かれているファイルを指定しています。
>   - `compose.production.yml` は本番環境用に用意されているファイルで、`-f` オプションなしで実行した場合は開発環境用の `compose.yml` が選択されます。
> - `--env-file .env.production`
>   - Docker Compose 内の Docker イメージで使用される環境変数ファイルを指定しています。
>   - `.env.production` は本番環境用のもので、`--env-file` オプションなしの場合は開発環境用の `.env` が選択されます。
> - `-p hatoapi-prod`
>   - Docker Compose でコンテナを起動した際のコンテナ群の名前を指定しています。
>   - 開発環境と本番環境のコンテナ群が一眼で見分けられるよう `-prod` をつけていますが、どんな名前でも大丈夫です。

1. Docker Compose コンテナの起動

先ほどビルドした Docker イメージを用いて Docker Compose で各コンテナを起動します。

```bash
docker compose -f compose.production.yml --env-file .env.production -p hatoapi-prod up -d
```

しばらくした後、ターミナルに表示される各コンテナのステータスが "Started" になれば成功です。

```bash
❯ docker compose -f compose.production.yml --env-file .env.production -p hatoapi-prod up -d
[+] Running 6/6
 ✔ Network hatoapi-prod_default          Created                                                                  0.0s 
 ✔ Container hatoapi-prod-mongo-1        Started                                                                  6.5s 
 ✔ Container hatoapi-prod-redis-1        Started                                                                  6.3s 
 ✔ Container hatoapi-prod-api-1          Started                                                                 11.4s 
 ✔ Container hatoapi-prod-push-server-1  Started                                                                 12.0s 
 ✔ Container hatoapi-prod-crawler-1      Started                                                                 10.0s 
```

1. データベースの復元

Hato API ではデータベースに MongoDB を使用しており、`mongodump` と `mongorestore` ユーティリティを使用してバックアップ・復元を行うことができます。先ほど起動した `hatoapi-prod-mongo-1` コンテナには今は何もデータが入っていない状態なので、`mongorestore` を使用してデータを復元します。  
復元に使用するダンプデータは、引き継ぎ等の前任者から何らかの手段で受け取ってください。

以下のコマンドを実行し、ダンプデータからコンテナ内のデータベースにデータを復元します。

> [!NOTE]
> データベースのユーザー名、パスワードは `.env.production` 内で設定している値と同じものです。

```bash
mongorestore --host localhost:27017 --username <データベースのユーザー名> --password <データベースのパスワード> --authenticationDatabase admin --gzip --archive=mongo.dump --db hatoapi
```

1. アクセス確認

ここまで終了したら、新しい Hato API が稼働しているサーバーアドレスを設定した Hato フロントエンドから Hato を開き、動作に問題ないことを確認します。
