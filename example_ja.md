---
layout: manual_ja
title: Setting Example / genn.ai
---

# genn.ai 設定例

> ここでは、genn.aiを構成するGungnirServer, TupleStoreServerなどの設定例について記載します。

* [Standaloneでgennaiを起動](#standalone)
* [MetaStoreをMongoDBに設定](#mongodbmetastore)
* [TupleStoreServerをKafkaに設定](#kafkaemitter)
* [DistributedのStormを利用](#distributedstorm)
* [GungnirServerを複数ホストで冗長化](#distributedgs1)
* [GungnirServerを単一ホストで冗長化](#distributedgs2)
* [TupleStoreServerを複数ホストで冗長化](#distributedts1)
* [TupleStoreServerを単一ホストで冗長化](#distributedts2)
* [TupleStoreServerのPING確認](#pingts)

## Standaloneでgennaiを起動 <a name="standalone"></a>

GungnirServerのみで動作確認をすることが可能なモードです。

[gungnir/README.md](https://github.com/TryGennai/gennai/tree/master/gungnir)の「Deploying and starting the standalone server」に記載されています。

使用している設定項目は下記です。

* [persistent.emitter](/config_ja.html#s.persistent.emitter)
* [metastore](/config_ja.html#s.metastore)

これらの設定項目以外はデフォルト設定値が適用されます。

> Example:
>
    persistent.emitter: org.gennai.gungnir.tuple.persistent.InMemoryEmitter
    metastore: org.gennai.gungnir.metastore.InMemoryMetaStore

Standaloneモードでは下記の制限事項があります。

1. Apache Kafkaを起動する必要はありませんが、Tupleは永続化されません。
2. 投入するTopologyで使用可能な **Spout Processor** は **Memory Spout Processor** のみです。


## MetaStoreをMongoDBに設定 <a name='mongodbmetastore'></a>

デフォルト設定ですが、Standaloneモードにおいても設定することが可能です。

事前にMongoDBを設定・起動しておく必要があります。

* [metastore](/config_ja.html#s.metastore)
* [metastore.mongodb.servers](/config_ja.html#s.metastore.mongodb.servers)

MongoDBを別ホストで起動している場合は **metastore.mongodbservers** も必ず指定してください。 **metastore.mongodb.servers** には、レプリケーション設定をした複数のMongoDBサーバを指定することが可能です。レプリケーション設定がされているMongoDBの場合にのみ、複数のMongoDBサーバを指定してください。レプリケーション設定がされていないMongoDBサーバを複数指定した場合、複数台にメタ情報を分散してしまう可能性があるため、メタ情報の一貫性が保証できなくなります。

> Example:
>
    ### Metastore
    metastore: org.gennai.gungnir.metastore.MongoDbMetaStore
    metastore.mongodb.servers:
      - "localhost:27017"

> Example: レプリケーション設定をしているMongoDBにメタ情報を保存
>
    ### Metastore
    metastore: org.gennai.gungnir.metastore.MongoDbMetaStore
    metastore.mongodb.servers:
      - "mongodb1:27017"
      - "mongodb2:27017"
      - "mongodb3:27017"


## TupleStoreServerをKafkaに設定 <a name='kafkaemitter'></a>

TupleStoreServerが受領したTupleをApache Kafkaに保存します(デフォルト設定)。保存されたTupleは、Topologyで吸い出すことができます。

* [persistent.emitter](/config_ja.html#s.persistent.emitter)
* [kafka.brokers](/config_ja.html#s.kafka.brokers)
* [kafka.zookeeper.servers](/config_ja.html#s.kafka.zookeeper.servers)

Apache Kafkaを別ホストで起動している場合は、 **kafka.brokers** も必ず指定してください。Apache Kafkaクラスタを構成するBrokerサーバを全て指定してください。また、Apache Kafkaが使用するZooKeeperアンサンブルも指定してください。

> Example:
>
    ### Tuple store server
    persistent.emitter: org.gennai.gungnir.tuple.persistent.KafkaPersistentEmitter
    ### Tuple store
    kafka.brokers:
      - "localhost:9092"
    kafka.zookeeper.servers:
      - "localhost:2181"

> Example: Apache Kafkaクラスタの設定
>
    ### Tuple store server
    persistent.emitter: org.gennai.gungnir.tuple.persistent.KafkaPersistentEmitter   
    ### Tuple store
    kafka.brokers:
      - "kafka1:9092"
      - "kafka2:9092"
      - "kafka3:9092"
    kafka.zookeeper.servers:
      - "kafka1:2181"
      - "kafka2:2181"
      - "kafka3:2181"


## DistributedのStormを利用 <a name="distributedstorm"></a>

分散モードで稼働しているStormクラスタを利用します。デフォルト設定では、Stormはローカルモードで稼働します。

* [storm.cluster.mode](/config_ja.html#s.storm.cluster.mode)
* [storm.nimbus.host](/config_ja.html#s.storm.nimbus.host)

Stormクラスタを別ホストで稼働させている場合、 **storm.cluster.mode** を **distributed** に設定すると共に、 **storm.nimbus.host** を正しく設定する必要があります。

> Example:
>
    ### Storm cluster
    storm.cluster.mode: "distributed"
    storm.nimbus.host: "localhost"


## GungnirServerを複数ホストで冗長化 <a name="distributedgs1"></a>

GungnirServerを複数のホストで稼働させることで、冗長化することができます。

稼働しているGungnirServerの情報を保存するZooKeeperが必須となります。クライアントツールは、ZooKeeperから稼働中のGungnirServerの情報を随時取得します。クライアントツールは、GungnirServerの構成に変更があった場合、ZooKeeperより接続情報を取得し、常に接続可能なGungnirServerにアクセスするようになります。

* [cluster.mode](/config_ja.html#s.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#s.cluster.zookeeper.servers)

上記設定項目は全てのホストで同一の設定をしてください。ZooKeeperアンサンブルを構成せず、1台のZooKeeperサーバにて稼働させる場合には、"localhost:2181"とそれぞれ設定してしまうと、異なるZooKeeperサーバを参照することになってしまいますのでご注意ください。

> Example: gungnir-server/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

それぞれのホストで下記コマンドを実行し、GungnirServerを起動してください。

> Example: GungnirServerの起動
>
    $ cd ${GUNGNIR_SERVER_INSTALL_DIR}
    $ ./bin/gungnir-server.sh start

**cluster.mode** を **distributed** に設定した場合、TupleStoreServerの起動も別途必要です。

また、GungnirServerを冗長化している場合、接続するクライアントの設定も分散モードを利用してください。

* [cluster.mode](/config_ja.html#c.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#c.cluster.zookeeper.servers)

**cluster.zookeeper.servers** に設定するZooKeeperは、GungnirServerに設定したものと同じ設定をしてください。

> Example: gungnir-client/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

この際、 クライアントの下記設定項目は使用されません。

* [gungnir.server.host](/config_ja.html#c.gungnir.server.host)
* [gungnir.server.port](/config_ja.html#c.gungnir.server.port)
* [tuple.store.server.host](/config_ja.html#c.tuple.store.server.host)
* [tuple.store.server.port](/config_ja.html#c.tuple.store.server.port)


## GungnirServerを単一ホストで冗長化 <a name="distributedgs2"></a>

単一のホストで複数のGungnirServerプロセスを起動することで、冗長化することができます。起動するプロセス数と同数の設定ファイルが必要となります。

稼働しているGungnirServerの情報を保存するZooKeeperが必須となります。クライアントツールは、ZooKeeperから稼働中のGungnirServerの情報を随時取得します。クライアントツールは、GungnirServerの構成に変更があった場合、ZooKeeperより接続情報を取得し、常に接続可能なGungnirServerにアクセスするようになります。

* [gungnir.server.port](/config_ja.html#s.gungnir.server.port)
* [gungnir.server.pid.file](/config_ja.html#s.gungnir.server.pid.file)
* [cluster.mode](/config_ja.html#s.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#s.cluster.zookeeper.servers)

上記設定項目のうち、 **cluster.mode**, **cluster.zookeeper.servers** は全ての設定ファイルで同一の設定をしてください。

> Example: gungnir-server/conf/gungnir1.yaml
>
    ### Gungnir server
    gungnir.server.port: 7101
    gungnir.server.pid.file: gungnir-server1.pid
     :
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "localhost:2181"
> Example: gungnir-server/conf/gungnir2.yaml
>
    ### Gungnir server
    gungnir.server.port: 7101
    gungnir.server.pid.file: gungnir-server2.pid
     :
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "localhost:2181"


下記コマンドを実行し、GungnirServerを複数起動してください。

> Example: GungnirServerの起動
>
    $ cd ${GUNGNIR_SERVER_INSTALL_DIR}
    $ ./bin/gungnir-server.sh start ./conf/gungnir1.yaml
    $ ./bin/gungnir-server.sh start ./conf/gungnir2.yaml

**cluster.mode** を **distributed** に設定した場合、TupleStoreServerの起動も別途必要です。

また、GungnirServerを冗長化している場合、接続するクライアントの設定も分散モードを利用してください。

* [cluster.mode](/config_ja.html#c.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#c.cluster.zookeeper.servers)

**cluster.zookeeper.servers** に設定するZooKeeperは、GungnirServerに設定したものと同じ設定をしてください。

> Example: gungnir-client/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

この際、 クライアントの下記設定項目は使用されません。

* [gungnir.server.host](/config_ja.html#c.gungnir.server.host)
* [gungnir.server.port](/config_ja.html#c.gungnir.server.port)
* [tuple.store.server.host](/config_ja.html#c.tuple.store.server.host)
* [tuple.store.server.port](/config_ja.html#c.tuple.store.server.port)


## TupleStoreServerを複数ホストで冗長化 <a name="distributedts1"></a>

TupleStoreServerを複数のホストで稼働させることで、冗長化することができます。

稼働しているTupleStoreServerの情報を保存するZooKeeperが必須となります。クライアントツールは、ZooKeeperから稼働中のTupleStoreServerの情報を随時取得します。クライアントツールは、TupleStoreServerの構成に変更があった場合、ZooKeeperより接続情報を取得し、常に接続可能なTupleStoreServerにアクセスするようになります。

* [cluster.mode](/config_ja.html#s.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#s.cluster.zookeeper.servers)

上記設定項目は全てのホストで同一の設定をしてください。ZooKeeperアンサンブルを構成せず、1台のZooKeeperサーバにて稼働させる場合には、"localhost:2181"とそれぞれ設定をしてしまうと、異なるZooKeeperサーバを参照することになってしまいますのでご注意ください。

> Example: gungnir-server/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

それぞれのホストで下記コマンドを実行し、TupleStoreServerを起動してください。

> Example: TupleStoreServerの起動
>
    $ cd ${GUNGNIR_SERVER_INSTALL_DIR}
    $ ./bin/tuple-store-server.sh start

**cluster.mode** を **distributed** に設定した場合、GungnirServerの起動も別途必要です。

また、TupleStoreServerを冗長化している場合、接続するクライアントの設定も分散モードを利用してください。

* [cluster.mode](/config_ja.html#c.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#c.cluster.zookeeper.servers)

**cluster.zookeeper.servers** に設定するZooKeeperは、GungnirServer/TupleStoreServerに設定したものと同じ設定をしてください。

> Example: gungnir-client/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

この際、 クライアントの下記設定項目は使用されません。

* [gungnir.server.host](/config_ja.html#c.gungnir.server.host)
* [gungnir.server.port](/config_ja.html#c.gungnir.server.port)
* [tuple.store.server.host](/config_ja.html#c.tuple.store.server.host)
* [tuple.store.server.port](/config_ja.html#c.tuple.store.server.port)


## TupleStoreServerを単一ホストで冗長化 <a name="distributedts2"></a>

単一のホストで複数のTupleStoreServerプロセスを起動することで、冗長化することができます。起動するプロセス数と同数の設定ファイルが必要となります。

稼働しているTupleStoreServerの情報を保存するZooKeeperが必須となります。クライアントツールは、ZooKeeperから稼働中のTupleStoreServerの情報を随時取得します。クライアントツールは、TupleStoreServerの構成に変更があった場合、ZooKeeperより接続情報を取得し、常に接続可能なTupleStoreServerにアクセスするようになります。

* [tuple.store.server.port](/config_ja.html#s.tuple.store.server.port)
* [tuple.store.server.pid.file](/config_ja.html#s.tuple.store.server.pid.file)
* [cluster.mode](/config_ja.html#s.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#s.cluster.zookeeper.servers)

上記設定項目のうち、 **cluster.mode**, **cluster.zookeeper.servers** は全ての設定ファイルで同一の設定をしてください。

> Example: gungnir-server/conf/gungnir1.yaml
>
    ### Tuple store server
    tuple.store.server.port: 7201
    tuple.store.server.pid.file: tuple-store-server1.pid
     :
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "localhost:2181"
> Example: gungnir-server/conf/gungnir2.yaml
>
    ### Tuple store server
    tuple.store.server.port: 7202
    tuple.store.server.pid.file: tuple-store-server2.pid
     :
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "localhost:2181"


下記コマンドを実行し、TupleStoreServerを複数起動してください。

> Example: TupleStoreServerの起動
>
    $ cd ${GUNGNIR_SERVER_INSTALL_DIR}
    $ ./bin/tuple-store-server.sh start ./conf/gungnir1.yaml
    $ ./bin/tuple-store-server.sh start ./conf/gungnir2.yaml

また、TupleStoreServerを冗長化している場合、接続するクライアントの設定も分散モードを利用してください。

* [cluster.mode](/config_ja.html#c.cluster.mode)
* [cluster.zookeeper.servers](/config_ja.html#c.cluster.zookeeper.servers)

**cluster.zookeeper.servers** に設定するZooKeeperは、GungnirServerに設定したものと同じ設定をしてください。

> Example: gungnir-client/conf/gungnir.yaml
>
    ### Cluster
    cluster.mode: "distributed"
    cluster.zookeeper.servers:
      - "zookeeper1:2181"
      - "zookeeper2:2181"
      - "zookeeper3:2181"

この際、 クライアントの下記設定項目は使用されません。

* [gungnir.server.host](/config_ja.html#c.gungnir.server.host)
* [gungnir.server.port](/config_ja.html#c.gungnir.server.port)
* [tuple.store.server.host](/config_ja.html#c.tuple.store.server.host)
* [tuple.store.server.port](/config_ja.html#c.tuple.store.server.port)


## TupleStoreServerのPING確認 <a name="pingts"></a>

TupleStoreServerの疎通確認を行うことができます。

TupleStoreServerの下記PATHへアクセスをすると、GungnirServer/TupleStoreServerのバージョン情報を取得することができます。

    /gungnir/v1.0

> Example: curl
>
    $ curl http://localhost:7200/gungnir/v1.0
    {version: "0.0.1"}
