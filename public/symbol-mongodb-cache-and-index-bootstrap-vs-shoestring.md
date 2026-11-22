---
title: SymbolノードのMongoDB設定をbootstrapとShoestringで比較する
tags:
  - Symbol
  - MongoDB
  - Docker
  - docker-compose
  - shoestring
private: false
updated_at: '2026-11-22T21:40:19+09:00'
id: 58e8a6314e2bb10aeacc
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

Symbolノードを運用していて、MongoDBの起動設定の違いが気になりました。

従来の `symbol-bootstrap` では、MongoDBのWiredTigerキャッシュサイズが2GBに設定されています。一方、新しいノード構築ツールである `Shoestring` には、この指定がありません。

「MongoDBがメモリを大量に使って、最悪の場合は落ちるのでは？」と思ったので、実際の設定とMongoDBの仕様を確認しました。

その結果、キャッシュサイズのほかに、**MongoDBのインデックスをいつ作成するか**という運用上の違いも見つかりました。この記事では、その2点を整理します。

## bootstrapとShoestringのMongoDB設定

### symbol-bootstrap

bootstrapのMongoDB起動コマンドは次のとおりです。

```bash
mongod --dbpath=/dbdata --bind_ip=db --wiredTigerCacheSizeGB 2
```

末尾の `--wiredTigerCacheSizeGB 2` は、MongoDBのストレージエンジンであるWiredTigerの内部キャッシュを2GB（MongoDBの指定単位はGiB相当）に制限する設定です。この値は `presets/shared.yml` で定義されています。

```yaml:presets/shared.yml
mongoComposeRunParam: --wiredTigerCacheSizeGB 2
```

### Shoestring

:::note info
この記事のShoestringに関する説明は、MongoDBを含むフルAPIのDual構成を対象としています。ライトAPIの構成にはMongoDBがありません。
:::

ShoestringのDual構成テンプレート `docker-compose-dual.yaml` では、MongoDBの起動コマンドは次のとおりです。

```yaml:docker-compose-dual.yaml（抜粋）
db:
  image: '{{ mongo_image }}'
  user: '{{ user }}'
  command: mongod --dbpath=/dbdata
  stop_signal: SIGINT
```

WiredTigerのキャッシュサイズは指定されていないため、MongoDBのデフォルト値が使われます。

## MongoDBのキャッシュは際限なく増えるのか

キャッシュサイズを指定しなくても、キャッシュが無制限に増えるわけではありません。WiredTigerにはデフォルトの上限があり、MongoDB 7.0では次の式で決まります。

**上限 = max(256MiB, (RAM容量 − 1GiB) × 50%)**

| RAM容量 | デフォルトのキャッシュ上限 |
| ------- | -------------------------- |
| 4GiB    | 約1.5GiB                   |
| 8GiB    | 約3.5GiB                   |
| 16GiB   | 約7.5GiB                   |
| 32GiB   | 約15.5GiB                  |

ここでの「RAM容量」は、MongoDBが認識しているメモリ量です。コンテナにcgroupのメモリ制限が設定されている場合、MongoDBはその制限値を基準に計算します。確認したShoestringのテンプレートにはメモリ制限がないため、通常はホストのRAM容量が基準になります。

例えば16GiBのサーバーでは、上限は約7.5GiBです。bootstrapの2GiB固定と比べると、かなり大きくなります。ただし、これは上限です。常にその容量を使うわけではありません。

## キャッシュサイズ2GiBはMongoDB全体のメモリ制限ではない

誤解しやすいのですが、`--wiredTigerCacheSizeGB 2` はWiredTigerの内部キャッシュの上限を決める設定です。MongoDBプロセス全体のメモリ使用量を2GiB以下に抑えるものではありません。

MongoDBは、キャッシュ以外にも次のような用途でメモリを使います。

- クエリ処理や接続ごとの領域
- インデックスの作成
- 各種内部データ構造やバッファ

そのため、`mongod` プロセスの使用量はWiredTigerの設定値より大きくなります。

また、MongoDBはデータファイルの読み書きでLinuxのファイルシステムキャッシュ（ページキャッシュ）も間接的に利用します。ページキャッシュはプロセスのメモリとは別に計上され、必要に応じてカーネルが回収します。そのため、空きメモリが少なく見えても、すぐにメモリ不足になるとは限りません。

### Shoestringで問題になる可能性

確認したShoestringのDual構成テンプレートには、WiredTigerのキャッシュ上限もDockerのメモリ上限も指定されていません。

Dualノードでは、MongoDBのほかにCatapult Client、Broker、RESTなども動作します。メモリに余裕のないサーバーでは、MongoDBがほかのプロセスとメモリを奪い合い、最悪の場合はOOM KillerによってMongoDBやほかのプロセスが強制終了される可能性があります。

これは、Shoestringを使うと必ずOOMになるという意味ではありません。メモリに余裕がある環境では、キャッシュを大きく使えることが性能面の利点になります。重要なのは、**MongoDBのメモリ使用量がサーバー全体の構成に対して適切かどうか**です。

## 実際のキャッシュサイズを確認する

MongoDBのキャッシュ上限と現在の使用量は `mongosh` から確認できます。Shoestringが生成した `docker-compose.yaml` があるディレクトリで実行します。

```bash
docker compose exec db mongosh --quiet --eval '
const c = db.serverStatus().wiredTiger.cache;
printjson({
  maxGiB: c["maximum bytes configured"] / 1024**3,
  usedGiB: c["bytes currently in the cache"] / 1024**3
});
'
```

- `maxGiB`：WiredTigerのキャッシュ上限
- `usedGiB`：現在のキャッシュ使用量

コンテナ全体のメモリ使用量は `docker stats` で確認できます。

```bash
docker stats --no-stream
```

`docker stats` の値は、コンテナのメモリ使用量から一部のページキャッシュを除いたものです。WiredTigerのキャッシュ使用量とは一致しません。WiredTigerの値と `docker stats` の値の両方を見ることで、キャッシュとプロセス全体の使用状況を把握できます。

## Shoestringのキャッシュ上限を変更する

Shoestringでbootstrapと同じ2GiBの上限を設定したい場合は、`docker-compose.override.yaml` を使うのが手軽です。

Docker Composeは、`-f` を指定せずに実行すると、カレントディレクトリの `docker-compose.yaml` に加えて `docker-compose.override.yaml` を自動で読み込み、マージします。Shoestringのノードは `-f` なしの `docker compose up -d` で起動するので、Shoestringが生成したファイルを直接編集しなくても、overrideファイルを置くだけで設定を上書きできます。

Shoestringが生成した `docker-compose.yaml` と同じディレクトリに、次のファイルを作成します。

```yaml:docker-compose.override.yaml
services:
  db:
    command: mongod --dbpath=/dbdata --wiredTigerCacheSizeGB 2
```

`command` は元の設定とマージされず、丸ごと置き換えられます。そのため、`--dbpath=/dbdata` を含めたコマンド全体を書いてください。

反映されているかは、マージ後の設定で確認できます。

```bash
docker compose config db
```

出力された `db` サービスの `command` に `--wiredTigerCacheSizeGB` が含まれていれば成功です。

設定を適用するには、通常どおりコンテナを起動します。設定が変わった `db` コンテナは再作成されます。

```bash
docker compose up -d
```

適用後は、前述の `mongosh` のコマンドで `maxGiB` が `2` になっていることを確認します。

:::note warn
**注意点**

- `docker compose -f docker-compose.yaml ...` のように `-f` を明示すると、`docker-compose.override.yaml` は読み込まれません。独自の起動スクリプトなどで `-f` を使う場合は、overrideファイルも明示的に指定してください。
- `db` コンテナの再作成中は、MongoDBを使うRESTなどのサービスにも影響があります。稼働中のノードでは、停止手順や依存サービスを確認してから作業してください。
- Shoestringのアップグレードなどで `docker-compose.yaml` が再生成された場合は、overrideファイルが残っているか、`docker compose config db` の結果が意図どおりかを確認してください。
:::

:::note info
2GiBは、bootstrapと比較するための値です。すべてのサーバーにとって適切な上限ではありません。キャッシュを小さくしすぎると、ディスクI/Oが増えて性能が下がる場合があります。
:::

## もうひとつの違い：MongoDBのインデックス

SymbolのMongoDBでは、ブロック、トランザクション、アカウントなどの検索を効率化するため、多数のインデックスを作成します。インデックスが不足すると、クエリに時間がかかり、REST APIの応答性能が下がる可能性があります。

このインデックス作成スクリプトを実行する仕組みが、bootstrapとShoestringで異なります。

### bootstrap：DBの初期化時にのみ実行

bootstrapは、MongoDB公式Dockerイメージの初期化機構を利用しています。Compose設定では、インデックス作成用のスクリプトを次の場所にマウントしています。

```yaml
volumes:
  - "./mongo:/docker-entrypoint-initdb.d:ro"
```

MongoDB公式イメージは、`/docker-entrypoint-initdb.d` のスクリプトを**データディレクトリが空のとき（初回起動時）だけ**実行します。既存のデータを残したままコンテナを再起動しても、スクリプトは再実行されません。

そのため、次の点に注意が必要です。

**bootstrapの更新で新しいインデックスの定義が追加されても、既存のMongoDBには自動で反映されません。**

データベースを初期化して再同期すれば、更新後のスクリプトが実行されて新しいインデックスが作られます。ただし、**インデックスを追加するためだけに再同期する必要はありません。** 必要なインデックスを確認して、既存のデータベースに `createIndex()` を実行すれば、再同期せずに追加できます。

大きなコレクションにインデックスを作成すると、CPU、メモリ、ディスクI/Oを消費します。実行前に空き容量と負荷を確認してください。

### Shoestring：DB準備用コンテナから実行

ShoestringのDual構成には `initiate` というサービスがあります。

```yaml:docker-compose-dual.yaml（抜粋）
initiate:
  image: '{{ mongo_image }}'
  command: /bin/bash /startup/mongors.sh /real_data/startup/mongo-initialized
```

`mongors.sh` は、MongoDBの起動を待ってから次の処理を実行します。

```bash:mongors.sh（抜粋）
cd /mongo
mongosh db/catapult < mongoDbPrepare.js
```

この方式は、MongoDB公式イメージの初回初期化に依存していません。そのため、`initiate` コンテナが起動してスクリプトが正常に実行されれば、既存のMongoDBにもインデックスの作成処理が適用されます。

MongoDBの `createIndex()` は、まったく同じ定義のインデックスがすでにある場合は何もしません。そのため、スクリプトを繰り返し実行しても、既存のインデックスが作り直されることはありません。一方、名前またはキーが同じで定義が一致しないインデックスがある場合はエラーになり、自動では置き換えられません。

また、**Shoestringを更新しただけで、必ず最新のインデックスになるわけではありません。** `initiate` コンテナはスクリプトを実行したあと `sleep infinity` で動き続けます。`./mongo` はバインドマウントなので、中のスクリプトが更新されても `docker compose up -d` ではコンテナが再作成されず、スクリプトも再実行されません。スクリプトを再実行するには、`initiate` コンテナを再起動します。

```bash
docker compose restart initiate
```

## bootstrapで運用しているノードはインデックスを確認する

長期間bootstrapで運用しているノードでは、構築した時点のインデックスがそのまま残っている可能性があります。MongoDBやRESTのバージョンを更新していても、インデックスまで更新されているとは限りません。

例えば、symbol-bootstrapでは [#377](https://github.com/symbol/symbol-bootstrap/commit/3973491e9b2616dae1cf698c185109cb444ec1a1) の変更で次のインデックスが追加され、v1.1.10（2023年4月）でリリースされました。

- `transactions`：`{ 'transaction.mosaics.id': 1 }`
- `transactionStatements`：`{ 'statement.receipts.targetAddress': 1 }`、`{ 'statement.receipts.senderAddress': 1 }`

v1.1.10より前のbootstrapで構築したノードには、これらのインデックスがない可能性があります。

現在のインデックスは、Composeファイルがある `target/docker/` で次のように確認できます。bootstrapの `db` は `--bind_ip=db` で起動していて `127.0.0.1` では待ち受けていないため、`--host db` を指定します。

```bash
docker compose exec db mongosh --host db --quiet \
  --eval 'db.getSiblingDB("catapult").transactions.getIndexes()'

docker compose exec db mongosh --host db --quiet \
  --eval 'db.getSiblingDB("catapult").transactionStatements.getIndexes()'
```

この結果と、利用している構築ツールの最新版にあるインデックス定義を比較します。インデックス定義は `mongoDbPrepare.js` のほかに、プラグインごとの `mongo*DbPrepare.js` にも分かれているので、まとめて確認してください。不足しているものがあれば、データベースを初期化する前に、個別に追加できないか検討するとよいでしょう。

:::note warn
bootstrap環境で手動で適用する場合は、準備スクリプトをそのまま流すのではなく、内容を確認して必要な `createIndex()` だけを実行する方が安全です。利用中のバージョンとスクリプトの前提が合っているとは限らないためです。
:::

## 比較まとめ

| 項目                           | bootstrap                    | Shoestring                         |
| ------------------------------ | ---------------------------- | ---------------------------------- |
| WiredTigerキャッシュ           | 2GiBを指定                   | MongoDBのデフォルト値              |
| コンテナのメモリ上限           | 未設定（キャッシュのみ2GiB） | 確認したテンプレートでは未設定     |
| インデックス作成               | DBの初回初期化時             | `initiate` サービスの実行時        |
| 更新で追加されたインデックス   | 通常の再起動では反映されない | 準備スクリプトの再実行で追加される |
| インデックス追加のための再同期 | 手動で追加すれば不要         | 原則不要                           |

bootstrapは、WiredTigerのキャッシュ使用量を予測しやすい反面、インデックスの更新に注意が必要です。Shoestringは、既存のデータベースにもインデックス作成スクリプトを適用できる反面、メモリ使用量の上限を環境に合わせて確認した方がよいでしょう。

## おわりに

ノード構築ツールを使えばコマンド一つでノードを起動できるため、内部のMongoDB設定まで確認する機会はあまりありません。しかし、設定の小さな違いが、メモリ使用量やREST APIの応答性能に影響することがあります。

今回確認したのは、次の2点です。

1. ShoestringではWiredTigerのキャッシュサイズが指定されておらず、MongoDBのデフォルト値が使われる。必要なら `docker-compose.override.yaml` で上書きできる。
2. bootstrapではインデックスの作成がDBの初期化時に限られるため、新しい定義が追加されても既存のDBには自動で反映されない。

どちらも、ノードが正常に起動しているだけでは気づきにくい問題です。**動いていることと、現在の推奨設定が反映されていることは別**だと考えた方がよさそうです。ノードを更新するときは、バージョンだけでなく、MongoDBの設定やインデックスも確認しておきたいところです。

## 参考資料

- [symbol-bootstrap：MongoDB共通設定](https://github.com/symbol/symbol-bootstrap/blob/dev/presets/shared.yml)
- [symbol-bootstrap：MongoDB初期化スクリプト](https://github.com/symbol/symbol-bootstrap/blob/dev/config/docker/mongo/mongoDbPrepare.js)
- [Shoestring：Docker Composeテンプレート](https://github.com/symbol/product/blob/dev/tools/shoestring/shoestring/templates/docker-compose-dual.yaml)
- [Shoestring：MongoDB準備スクリプト](https://github.com/symbol/product/blob/dev/tools/shoestring/shoestring/startup/mongors.sh)
- [MongoDB公式：WiredTiger Storage Engine](https://www.mongodb.com/docs/v7.0/core/wiredtiger/)
- [MongoDB公式Dockerイメージ](https://hub.docker.com/_/mongo)
- [Docker公式：Merge Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)

:::note info
この記事は、2026年11月22日時点の公式リポジトリのdevブランチと、MongoDB 7.0の仕様を基にしています。実際の挙動は、利用するバージョンや構成によって異なる場合があります。
:::
