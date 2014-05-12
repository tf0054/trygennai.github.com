---
layout: manual_ja
title: genn.ai
---

## LanguageManual CLI

### Gungnir Command Line Options

    $ gungnir options

    usage: gungnir
     -h,--help       Print help information
     -p <password>   Password to use when connecting to gungnir server
     -u <username>   Username to use when connecting to gungnir server

---

### Gungnir Interactive Shell Commands

Use ";" (semicolon) to terminate commands.

#### quit | exit

Gungnir CLIを終了します。

    gungnir> quit;

---

#### EXPLAIN

Topologyの実行計画を表示します。

    gungnir> EXPLAIN;

> Example:

    gungnir> CREATE TUPLE userAction (field1 INT, field2 STRING);
    OK
    gungnir> FROM userAction AS ua1 USING kafka_spout() INTO s1;
    OK
    gungnir> FROM s1 FILTER field1 >= 10 INTO s2;
    OK
    gungnir> FROM s1 FILTER field1 < 10 INTO s3;
    OK
    gungnir> EXPLAIN;
    SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])
     -S-> PARTITION_1
    PARTITION_1(identity grouping)
     -S-> FILTER_2
     -S-> FILTER_3
    FILTER_2(field1 >= 10)
    FILTER_3(field1 < 10)

実行計画から、Tupleを投入した際にオペレータがどのように呼ばれるかを確認することができます。
それぞれのオペレータをVertex、それらをつなぐStreamをEdgeとするGraph構造になっています。
上記の例では、SPOUT_0・PARTITION_1・FILTER_2・FILTER_3の４つのオペレータをStreamで連結しています。
それぞれのオペレータの下の行に、そのオペレータからのびているStreamが表示されています。
SPOUT_0からのびているStreamは１つで、PARTION_1に接続しています。
PARTITION_1からのびているStreamは２つで、それぞれFILTER_2とFILTER_3に接続しています。

---

#### SUBMIT TOPOLOGY

Topologyを登録して起動します。

    gungnir> SUBMIT TOPOLOGY;

Topologyの起動は非同期で実行される為、必ず DESC TOPOLOGY を実行して、Topologyの状態が RUNNING（実行中）になっていることを確認してください。

> Example:

    gungnir> FROM userAction AS ua1 USING kafka_spout() FILTER field1 = 10 INTO s1;
    OK
    gungnir> EXPLAIN;
    …
    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir> DESC TOPOLOGY;  <-- 起動したかを確認
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"RUNNING", ...}

SUBMIT TOPOLOGYの実行後に、DESC TOPOLOGYを実行してTopology IDとTopologyの状態を確認しています。

---

#### DESC TOPOLOGY

登録したTopologyの情報を表示します。

    gungnir> DESC TOPOLOGY;

結果はJSON形式で出力されます。

> Example:
    gungnir> DESC TOPOLOGY;
    {"id":"5261606ee4b099995d4f460f","explain":"SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])\n ...","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z","summary":{"name":"gungnir_5261606ee4b099995d4f460f","status":"ACTIVE","uptimeSecs":403,"numWorkers":1,"numExecutors":3,"numTasks":3}}

* id は、Topology IDです。
* explain は。Topologyの実行計画です。
* status は、Topologyの状態です。
Topologyの状態に応じて、STARTING（起動中）-> RUNNING（実行中）-> STOPPING（停止中）-> STOPPED（停止状態）と変化します。
* owner は、Topologyを登録したユーザのユーザ名です。
* createTime は、Topologyを登録した日時です。
* summary は、Topologyの起動情報です。status がRUNNINGの場合のみに表示されます。

特定のTopologyの情報を表示したい場合は、情報を表示したいTopologyのTopology IDを指定します。

    gungnir> DESC TOPOLOGY topology_id;

> Example:

    gungnir> DESC TOPOLOGY 5261606ee4b099995d4f460f;
    {"id":"5261606ee4b099995d4f460f", ...}

---

#### SHOW TOPOLOGIES

登録したTopologyの一覧を表示します。

    gungnir> SHOW TOPOLOGIES;

結果はJSON形式で出力されます。

> Example:

    gungnir> SHOW TOPOLOGIES;
    [{"id":"5261606ee4b099995d4f460f","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z"}]

* id は、Topology IDです。
* status は、Topologyの状態です。
Topologyの状態に応じて、STARTING（起動中）-> RUNNING（実行中）-> STOPPING（停止中）-> STOPPED（停止状態）と変化します。
* owner は、Topologyを登録したユーザのユーザ名です。
* createTime は、Topologyを登録した日時です。

---

#### STOP TOPOLOGY

起動したTopologyを停止します。

    gungnir> STOP TOPOLOGY topology_id;

* topology_id には、停止するTopologyのTopology IDを指定します。

Topologyの停止は非同期で実行される為、必ず DESC TOPOLOGY を実行して、Topologyの状態が STOPPED （停止状態）になっていることを確認してください。

> Example:
    gungnir> STOP TOPOLOGY 5261606ee4b099995d4f460f;
    OK
    gungnir> DESC TOPOLOGY;  <-- 停止したかを確認
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"STOPPED", ...}

---

#### START TOPOLOGY

停止したTopologyを再起動します。

    gungnir> START TOPOLOGY topology_id;

* topology_id には、起動するTopologyのTopology IDを指定します。

Topologyの開始は非同期で実行される為、必ず DESC TOPOLOGY を実行して、Topologyの状態が RUNNING（実行中）になっていることを確認してください。

> Example:
    gungnir> START TOPOLOGY 5261606ee4b099995d4f460f;
    OK
    gungnir> DESC TOPOLOGY;  <-- 起動したかを確認
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"RUNNING", ...}

---

#### DROP TOPOLOGY

登録したTopologyを削除します。

    gungnir> DROP TOPOLOGY topology_id;

* topology_id には、削除するTopologyのTopology IDを指定します。

Topologyは停止している必要があります。削除する前に DESC TOPOLOGY を実行して、Topologyの状態が STOPPED になっていることを確認してください。

> Example:

    gungnir> DESC TOPOLOGY;  <-- 停止しているかを確認
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"STOPPED", ...}
    gungnir> DROP TOPOLOGY 5261606ee4b099995d4f460f;
    OK

---

#### CLEAR

登録したクエリをメモリから消去します。
クエリ（FROM〜）は、TopologyをSUBMITするまでメモリに置かれます。
クエリを打ちなおしたい場合は、CLEARコマンドを実行してメモリをクリアしてください。

    gungnir> CLEAR;

> Example:
    gungnir> EXPLAIN;
    SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])
    …
    gungnir> CLEAR;
    OK
    gungnir> EXPLAIN;
    FAILED: Operator isn't registered

---

#### SET

Topologyのプロパティを設定します。
設定したプロパティは、SUBMIT TOPOLOGY もしくは START TOPOLOGY で、起動するTopologyに反映されます。
プロパティは、STOP TOPOLOGY で初期化されてしまうので、START TOPOLOGY を実行する前に再設定する必要があります。

    gungnir> SET property_name = property_value;

> Example:
    gungnir> SET monitor = enable;

> Monitorログを出力するように、プロパティを設定します。

---

#### TRACK

Topologyの動作確認の為に、TopologyにJOINTupleを送信します。

    gungnir> TRACK tuple_name json_tuple; 

* tuple_name には、送信するTuple名を指定します。
* join_tuple には、送信するJSONTupleを記述します。

応答にSet-Cookieヘッダが返された場合は、その内容がCLIのメモリCookieに保存され、保存されたCookieは、次回のTRACKコマンドの実行時にCookieヘッダとして送信されます。

> Example:
    gungnir> TRACK userAction {field1:10,field2:"test"}; 

* Interactive Mode

TRACKコマンドには、インタラクティブにJSONTupleを編集できるモードがあります。
json_tuple を指定せずにTRACKコマンドを実行すると、送信するJSONTupleの入力を求められます。
JSONTupleのすべてのフィールドの入力が完了すると、編集したJSONTupleを送信します。

    gungnir> TRACK userAction;
    field1 (INT): 12345
    field2 (STRING): test
    TRACK userAction {"field1":12345,"field2":"test"}
    OK

LIST, MAP, STRUCT型のフィールドの編集は、以下のように行います。


    CREATE TUPLE userAction2 (f1 STRING, f2 LIST<INT>, f3 MAP<STRING, INT>, f4 STRUCT<m1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), m2 BOOLEAN>);

> Example:
    gungnir> TRACK userAction2;
    f1 (STRING): test
    f2 (LIST)
      0 (INT): 1
      1 (INT): 2
      2 (INT): 3
      3 (INT):   <-- [Enter key]を押して、LISTの編集を終了
    f3 (MAP)
      key (STRING): k1
      value (INT): v1
      value (INT): 1
      key (STRING): k2
      value (INT): 2
      key (STRING): k3
      value (INT): 3
      key (STRING):   <-- keyの入力で [Enter key]を押して、MAPの編集を終了
    f4 (STRUCT)
      m1 (TIMESTAMP(yyyy-MM-dd HH:mm:ss)): 2013-10-19 22:02:24
      m2 (BOOLEAN): false

    TRACK userAction2 {"f1":"test","f2":[1,2,3],"f3":{"k1":1,"k2":2,"k3":3},"f4":{"m1":"2013-10-19 22:02:24","m2":false}}
    OK

---

#### COOKIE

メモリCookieの内容を表示します。
メモリCookieは、CLIの終了とともにメモリから削除されます。

    gungnir> COOKIE;

> Exmaple:
    gungnir> COOKIE;
    {"name":"TID","value":"52558684e4b0f241c72d0365","domain":null,"path":null,"comment":null,"commentUrl":null,"discard":false,"ports":[],"maxAge":864000000,"version":1,"secure":false,"httpOnly":false}]

---

#### COOKIE CLEAR

メモリCookieの内容を削除します。
メモリCookieには、Serverから受け取ったTracking IDが格納されている為、メモリCookieの内容を削除することによって
Serverから新たなTracking IDを受け取ることができます。

    gungnir> COOKIE CLEAR;

> Example:

    gungnir> COOKIE CLEAR;
    OK
    gungnir> COOKIE;
    []

---

#### MONITOR

Topologyの動作確認の為に、TupleがTopologyの中をどのように流れたをMonitor機能で確認します。
Tupleが流れた際に出力されるMonitorログを表示することによって、確認を行います。

    gungnir> MONITOR topology_id;

* 確認するTopologyのTopology IDを指定します。

Monitor機能は、以下の手順で使用します。

- Monitorログを出力するようにプロパティを設定して、該当のTopologyをSUBMITします。
- 1で起動したTopologyのTopology IDを指定して、MONITORコマンドを実行します。
- 別のCLIを起動し、TRACKコマンドを使ってJOINTupleを送信します。JOINTupleは、curlコマンド等を使って送信することもできます。
- MONITORコマンドを実行しているCLIに、3で送信したTupleが流れることによって出力されたMonitorログが表示されます。

> Example:

    gungnir> SET monitor = enable;
    OK
    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir> DESC TOPOLOGY;
    {"id":"5261606ee4b099995d4f460f","explain":"SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])\n ...","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z","summary":{"name":"gungnir_5261606ee4b099995d4f460f","status":"ACTIVE","uptimeSecs":403,"numWorkers":1,"numExecutors":3,"numTasks":3}}
    gungnir> MONITOR 5261606ee4b099995d4f460f;

別のCLIを開いてTRACKコマンドを実行します。

    gungnir> track userAction {"field1":1234,"field2":"test"};

Monitorログが出力されます。

    gungnir> MONITOR 5261606ee4b099995d4f460f;
    {"time":"2013-10-19T14:12:36.856Z","source":"SPOUT_0","target":"PARTITION_1","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.880Z","source":"PARTITION_1","target":"FILTER_2","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.953Z","source":"PARTITION_1","target":"FILTER_3","tuple":{"tupleName":"userAction","values":[1234,"test"]}}

source のオペレータから target のオペレータに向かって、Tupleが流れているのが確認できます。tuple に、流れたTupleの内容が表示されています。
