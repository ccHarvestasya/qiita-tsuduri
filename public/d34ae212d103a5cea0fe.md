---
title: Docker Composeのdepends_onでDB接続エラーが消えない理由
tags:
  - docker-compose
  - Docker
  - PostgreSQL
  - devops
  - トラブルシューティング
private: false
updated_at: '2026-11-12T07:43:26+09:00'
id: d34ae212d103a5cea0fe
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
Docker ComposeでDBとアプリケーションを起動するとき、サービス同士の依存関係を指定する `depends_on` を設定しているのにDB接続エラーになることがあります。DBを先に起動する設定にしたはずなのに、少し時間を置いてアプリケーションを起動し直すと、今度は接続できる。

`depends_on` の設定が無視されているわけではありません。依存先のサービス名を並べるだけの設定で待つのは、あくまで**DBコンテナの起動**です。DBが接続を受け付ける状態になるまで待つわけではありません。

今回はDBにPostgreSQLを使い、起動順の指定と、接続を受け付けるまで待つ設定の違いを見ていきます。

## depends_onにサービス名を指定するだけでは起動順しか待たない

よく使うのは、次の書き方です。

```yaml
depends_on:
  - db
```

これでアプリケーションより先に `db` コンテナが起動します。この書き方で待つ条件は `service_started` です。次のように明示しても、起動を待つ条件は変わりません。

```yaml
depends_on:
  db:
    condition: service_started
```

`condition` は依存先がどの状態になるまで待つかを指定する項目で、`service_started` は「コンテナが起動したら先へ進む」という条件です。

どちらの書き方も、`db` にヘルスチェックが設定されているかどうかに関係なく、コンテナの起動までしか待ちません。ヘルスチェックの成功を待つには、後述する `condition: service_healthy` を指定します。[Composeの仕様](https://docs.docker.com/reference/compose-file/services/#depends_on)でも、サービス名だけの指定と `service_started` は同じ待機条件として定義されています。

ところが、PostgreSQLはコンテナが起動しても、すぐに接続できるとは限りません。初回起動時はデータを保存するディレクトリの初期化や初期化スクリプトの実行に時間がかかり、再起動時にも復旧処理が必要になる場合があります。その途中でアプリケーションが接続しようとすると、接続エラーになります。

コンテナの起動順は合っていても、DBの準備が終わっていないわけです。

[Dockerの公式ドキュメント](https://docs.docker.com/compose/how-tos/startup-order/)でも、コンテナの起動とサービスの準備完了は区別されています。

対策には `healthcheck` と `condition: service_healthy` を使います。`healthcheck` は、コンテナ内で確認用のコマンドを定期的に実行し、その結果から状態を判定する設定です。`service_healthy` を指定すると、そのチェックに成功して `healthy`（正常）と判定されるまで、依存するサービスの起動を待ちます。今回はDBの接続受付を確認するコマンドを使い、接続できる状態になってからアプリケーションを起動します。

なお、**`healthcheck` を追加するだけでは不十分です。** `depends_on` にサービス名を並べるだけでは、依然として起動順しか待ちません。`healthcheck` と `condition: service_healthy` の両方を設定する必要があります。

## 接続確認用のコンテナで試す

接続確認のためだけにアプリケーションを作るのも大げさなので、今回は `SELECT 1;` を実行して終了する `probe` コンテナを使います。DBに接続できるかを試すだけの簡単な構成です。

使用するのはPostgreSQL 16です。Docker EngineとComposeプラグインがインストールされ、`docker compose` が実行できることを前提とします。`docker compose up` や `down` を使ったことがあれば、ヘルスチェックの知識はなくても読み進められます。ここでは1台のマシンでComposeを使います。複数のマシンにコンテナを分散して動かすDocker Swarmは対象外です。

2026年11月12日に、Ubuntu 26.04.1 LTS（WSL2）、Docker Engine 29.9.1、Docker Compose 5.6.0、PostgreSQL 16.15で動作を確認しています。

新しい作業用ディレクトリを用意し、以下を `compose.yaml` として保存してください。

実験では `depends-on-demo` というプロジェクト名を使用します。DBのデータは名前付きボリュームに保存されるので、同じ名前の既存プロジェクトがないことを確認してください。パスワードは実験専用で、本番用には使わないでください。DBのポートはホストには公開していません。

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: local-demo-only
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", 'pg_isready -h 127.0.0.1 -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"']
      interval: 2s
      timeout: 3s
      retries: 15
      start_period: 10s

  probe:
    image: postgres:16
    environment:
      PGHOST: db
      PGPORT: "5432"
      PGUSER: demo
      PGPASSWORD: local-demo-only
      PGDATABASE: demo
      PGCONNECT_TIMEOUT: "3"
    depends_on:
      db:
        condition: service_healthy
    command: ["psql", "-w", "-v", "ON_ERROR_STOP=1", "-c", "SELECT 1;"]

volumes:
  db-data:
```

`db` には `healthcheck` を設定し、`probe` 側には `condition: service_healthy` を指定しています。[Composeの仕様](https://docs.docker.com/reference/compose-file/services/#depends_on)に従い、次の順序で進みます。

1. Composeが `db` コンテナを起動する。
2. `db` 内でヘルスチェックを繰り返し、接続を受け付けるようになったら `healthy` と判定する。
3. Composeが `probe` コンテナを起動する。
4. `probe` がDBに接続して `SELECT 1;` を実行し、終了する。

接続やSQLの実行失敗を見落とさないために、`probe` の `psql` では、パスワードの入力待ちをしない `-w` と、SQLエラー時に処理を中断する `-v ON_ERROR_STOP=1` を付けています。`-c` の後ろには、実行するSQLを指定します。

### healthcheckのtestで接続受付を確認する

設定例の `db` → `healthcheck` → `test` に書いた `pg_isready` が、PostgreSQLの接続受付を確認するコマンドです。接続を受け付けていれば終了コード0を返し、ヘルスチェックが成功します。

`-h 127.0.0.1` を付けているのは、DBコンテナ内からTCPで接続を確認するためです。`-U` は接続ユーザー、`-d` はデータベース名の指定です。

公式PostgreSQLイメージは初期化中に一時的なサーバーを起動します。このサーバーはUnixソケット（同じコンテナ内で使う通信口）での接続を受け付けます。この経路で `pg_isready` を実行すると、一時サーバーへの接続に成功し、ヘルスチェックも成功します。その結果、初期化が終わっていないのに `healthy` と判定され、`probe` が起動してしまう可能性があります。

今回はTCP接続を指定して、この一時サーバーを準備完了と判定しないようにしています。[公式イメージの起動スクリプト](https://github.com/docker-library/postgres/blob/master/docker-entrypoint.sh)に合わせた指定であり、すべてのDBイメージに共通する条件ではありません。別のコンテナで使うときも、そのサービスの「準備完了」を何で確認できるかを考えて、チェック用のコマンドや接続方法を選ぶ必要があります。

## 起動する

設定を保存したら、まずComposeの構成をチェックします。その後、起動してコンテナの状態とログを確認します。

```bash
docker compose -p depends-on-demo config -q
docker compose -p depends-on-demo up -d
docker compose -p depends-on-demo ps -a
docker compose -p depends-on-demo logs --timestamps db probe
```

`docker compose -p depends-on-demo ps -a` の状態表示で、`db` が `healthy`、`probe` が `Exited (0)`（終了コード0で正常終了）になっていることを確認します。続く `logs --timestamps db probe` の出力では、`probe` のログに `SELECT 1;` の結果として `1` が表示されることを確認してください。

`--timestamps` は各行に時刻を付けますが、複数コンテナのログを時刻順に並べ替える指定ではありません。`probe` のログが `db` より先に表示されることもあるため、処理の前後は表示位置ではなく、各行のタイムスタンプで確認してください。

`up -d` は `probe` の処理終了までは待ちません。`probe` がまだ実行中なら、少し待って同じ `ps -a` とログ確認をやり直してください。

`probe` は一度だけSQLを実行するので、終了していても問題ありません。

### コマンドのオプション

先ほどのコマンドで使ったオプションをまとめておきます。慣れている方は読み飛ばしてください。

| 指定 | 役割 |
| --- | --- |
| `-p depends-on-demo` | Composeのプロジェクト名を指定する。この `-p` はポート番号の指定ではない |
| `config -q` | 設定を検証する。正常時は結果を表示しない |
| `up -d` | コンテナをバックグラウンドで起動する |
| `ps -a` | 終了済みのコンテナも表示する |
| `logs --timestamps` | ログの各行に時刻を表示する |

### 起動順だけを待つ設定と比較する場合

比較用の設定全体は次のとおりです。`db` の `healthcheck` は残し、`probe` の `depends_on` だけをサービス名の指定に変えています。

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: local-demo-only
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", 'pg_isready -h 127.0.0.1 -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"']
      interval: 2s
      timeout: 3s
      retries: 15
      start_period: 10s

  probe:
    image: postgres:16
    environment:
      PGHOST: db
      PGPORT: "5432"
      PGUSER: demo
      PGPASSWORD: local-demo-only
      PGDATABASE: demo
      PGCONNECT_TIMEOUT: "3"
    depends_on:
      - db
    command: ["psql", "-w", "-v", "ON_ERROR_STOP=1", "-c", "SELECT 1;"]

volumes:
  db-data:
```

比較用に別の作業ディレクトリを用意し、この設定を `compose.yaml` として保存してください。そのディレクトリで、別のプロジェクト名を指定して起動します。新しいボリュームで初期化するため、先ほどの実験のデータは使いません。

```bash
docker compose -p depends-on-short-demo up -d
docker compose -p depends-on-short-demo ps -a
docker compose -p depends-on-short-demo logs --timestamps db probe
```

この設定ではヘルスチェックは実行されますが、その成功を待たずに `probe` が起動します。

ただ、起動順だけを待つ設定でも必ず失敗するわけではありません。PostgreSQLの初期化が速ければ、`probe` が接続するころには準備が終わっていることもあります。初期化済みのボリュームを使った場合も同様です。

成功したとしても、**DBの準備完了を待ってくれたわけではない**点には注意してください。

## healthyになれば接続は成功する？

当たり前ですが、DBが接続を受け付けていても、ユーザー名やパスワードが間違っていれば接続できません。`pg_isready` が確認するのは接続を受け付ける状態かどうかで、**アプリケーションの認証情報が正しいかまでは確認しません。**

[PostgreSQLの公式ドキュメント](https://www.postgresql.org/docs/16/app-pg-isready.html)にも、サーバーの状態確認に正しいユーザー名やパスワードは必須ではないと書かれています。

そのため、今回の例では `pg_isready` とは別に、`probe` から `SELECT 1;` を実行しています。こちらではサービス名の解決、ネットワーク経由の接続、認証、SQLの実行を確認できます。使用したオプションは[psqlの公式ドキュメント](https://www.postgresql.org/docs/16/app-psql.html)を参照してください。

もちろん、`SELECT 1;` が成功しても、アプリケーションが必要とするテーブルや操作権限がそろっているとは限りません。マイグレーションが必要なら、その完了も別に確認します。Composeには、別サービスの正常終了を待つ `service_completed_successfully` という条件もあります。

つまり、DBへの接続ができることと、アプリケーションが必要な処理を実行できることは別です。

### エラーが出たら

すべての接続エラーが起動タイミングの問題とは限りません。エラーメッセージに合わせて確認する場所を変えます。

| 症状 | 確認するところ |
| --- | --- |
| 接続を拒否される | DBの起動状態、待受アドレス、接続先ポート |
| ホスト名が解決できない | サービス名、コンテナが参加しているネットワーク |
| パスワード認証に失敗する | 認証情報、DB側の認証設定 |
| テーブルが存在しない | 接続先DB、スキーマ、マイグレーション |

これはあくまで調査の取っかかりです。症状と原因が必ず一対一になるわけではありません。

接続先の指定も確認しておきましょう。今回の `probe` からDBへ接続するときは `db:5432` です。`localhost` を指定すると、DBではなく `probe` 自身に接続しようとします。

同じComposeネットワークにあるコンテナ同士は、サービス名とコンテナ側のポートを指定します。[Dockerのネットワークの説明](https://docs.docker.com/compose/how-tos/networking/)にもあるとおり、ホスト側にポートを公開する必要はありません。接続先が間違っているなら、ポートを公開しても解決しません。

## 起動後の接続エラーは別の対策が必要

`service_healthy` で待てるのは起動時の準備完了です。DBの起動後に接続が切れた場合まで、Composeが面倒を見てくれるわけではありません。

例えばDBが再起動すると、アプリケーションが保持している接続は使えなくなることがあります。この場合は、アプリケーション側で再接続できるようにしておく必要があります。

ドライバーや接続プールの動作を確認し、一時的な接続失敗なら間隔と回数・期限を決めて再試行します。パスワード間違いのような継続的なエラーを、いつまでも待ち続ける構成にはしません。

なお、**再接続とSQLの再実行は別の話です。** 書き込み直後に接続が切れると、すでにコミットされたのか分からない場合があります。同じ書き込みを再実行して重複しないかも考える必要があります。

Compose 2.17.0以降では、アプリケーション側の `depends_on` の `db` の下に `restart: true` を指定すると、Composeの操作でDBを更新・再起動した際に、依存するアプリケーションも再起動できます。例えば、`docker compose restart db` でDBを再起動した場合です。コンテナがクラッシュして自動再起動されたときは対象外で、アプリケーションの再接続処理の代わりにもなりません。

起動時のエラーを避けるために `sleep 10` を入れる方法もあります。これで動く環境はあるでしょうが、DBの準備に10秒以上かかればまた失敗します。待つべきなのは時間ではなく、DBが接続を受け付ける状態になったことです。

## 後片付け

実験を終えたら、DBのデータを残すかどうかで、次のどちらか一方を実行します。設定を保存した作業ディレクトリで操作してください。

データを残す場合は、コンテナとネットワークだけを削除します。

```bash
docker compose -p depends-on-demo down
```

実験用DBのデータも不要な場合は、上の `down` の代わりに、次の `down -v` を実行します。コンテナ・ネットワークと一緒に、DBのボリュームも削除します。

**`down -v` はボリューム内のデータも削除します。** 必ず実験用のプロジェクト名と作業ディレクトリを確認してから実行してください。

```bash
docker compose -p depends-on-demo down -v
```

比較用の構成も試した場合は、比較用の作業ディレクトリで、上の後片付けコマンドのプロジェクト名を `depends-on-short-demo` に置き換えて実行してください。

`depends_on` を指定してもDB接続エラーが消えないときは、まず起動順だけを待っているのか、`healthy` になるまで待っているのかを確認します。認証やスキーマのエラー、起動後の接続断は、それぞれ別に対処する必要があります。
