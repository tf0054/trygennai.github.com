## LanguageManual DML

### FROM ★

FROM は、Tupleを入力先から読み込みます。

#### 入力先が外部の場合

<pre>
FROM schema_name AS schema_alias, ... USING spout_processor
</pre>

* schema_name には、Tuple名もしくはView名を指定します。
* schema_alias には、クエリ内で使用するTupleもしくはViewの別名を指定します。
* spout_processor には、読み込みに使用するプロセッサを指定します。
 
> Example:
<pre>
FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout()
</pre>

外部入力は、一つのTopologyに対して一つしか定義できません。

#### Spout Processor

kafka_spout

> TupleをKafkaから読み込みます。システムのデフォルトとして動作します。
>
> <pre>
kafka_spout()
</pre>

#### 入力先が内部（ストリーム）の場合

<pre>
FROM stream_name[(schema_alias, ...)], ...
</pre>

* stream_name には、入力先のストリーム名を指定します。

ストリームからすべてのTupleを読み込む場合

> Example:
<pre>
FROM s1, s2
</pre>

ストリームから特定のTupleのみを読み込む場合

> Example:
<pre>
FROM s1(ua1, ua2), s2(v1)
</pre>

---

### INTO ★

INTO は、ストリームを分岐・合流させます。

<pre>
INTO stream_name
</pre>

* stream_name には、出力するストリーム名を指定します。

> Example:
<pre>
FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1
</pre>

INTO を使って出力したストリームは、FROM で読み込みます。

> 分岐
> 
> <pre>
FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1;
FROM s1(ua1) ...
FROM s1(ua2) ...
FROM s1(v1) ...
</pre>
>
> 合流
> 
> <pre>
FROM s1(ua1) ... INTO s2;
FROM s1(ua2) ... INTO s3;
FROM s1(v1) ... INTO s4;
FROM s2, s3, s4 ...
</pre>

---

### JOIN ★

JOINは、外部データをフィールドとしてTupleに結合します。

<pre>
JOIN join_name ON join_condition TO join_fields USING fetch_processor

join_condition:
join_name.key_field = field [AND join_name.key_field = field AND join_name.key_field <> 0]

join_fields:
join_name.join_field AS field_alias, ...
</pre>

* join_name には、結合する名称を指定します。join_condition や join_fields で、外部データのフィールドを識別する為に使用します。
* join_condition には、結合条件を指定します。
結合データのキーフィールド = Tupleのフィールド を、結合データが一意になるように指定してください。
複合条件の場合は、AND で指定します。結合データのキーフィールドに対して、定数で条件を指定することもできます。
* join_fields には、結合するフィールドを指定します。
結合データのフィールド名 AS Tupleに結合する際のフィールド名 を指定します。フィールドはTupleに追加されます。

> Example:
<pre>
JOIN j1 ON j1.code1 = field1 AND j1.code2 = field2 AND j1.del = 0
  TO j1.name AS field10, j1.type AS field11
  USING mongo_fetch('db1', 'col1')
</pre>

---

### FILTER

FILTER は、単一のTupleに対してTupleの通過を判定します。

  FILTER condition

* condition には、フィルタの条件を指定します。

> condition の符号には、以下のものを指定します。
>
> * = もしくは ==
> * <> もしくは !=
> * >
> * >=
> * <
> * <=
> * LIKE
> * REGEXP
> * IN
> * ALL
> * BETWEEN
> * AND
> * OR
> * NOT

#### =, ==, <>, !=, >, >=, <, <=

> Example:
<pre>
field1 >= 10
</pre>

#### LIKE

LIKEで使用できるワイルドカードは、"%"（複数文字）と"_"（一文字）です。

> Example:
<pre>
field2 LIKE 't%'
</pre>

#### REGEXP

REGEXPで使用できる正規表現は、java/util/regex/Patternと同じ書式を採用しています。
http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html

> Example:
<pre>
field3 REGEXP '^[A-Z]{2}-[0-9]{4}$'
</pre>

#### IN, ALL

INは、LISTフィールドに対して値が一つでも含まれているかを調べます。

> Example:
<pre>
field4 IN ('tokyo', 'kyoto', 'osaka')
</pre>

ALLは、LISTフィールドに対して値がすべて含まれているかを調べます。

> Example:
<pre>
field4 ALL ('tokyo', 'kyoto', 'osaka')
</pre>

#### BETWEEN

> Example:
<pre>
field1 10 AND 100
</pre>

#### AND, OR, NOT

AND, OR, NOT は入れ子にすることが可能です。優先順位はNOT, AND, ORの順に処理されます。
優先順位は、()を使用して変更できます。

> Example:
<pre>
field1 <= 30 AND (field5 BETWEEN 10 AND 100 OR field2 LIKE 'A%')
</pre>

#### STRUCT型フィールドの比較

TupleのフィールドがSTRUCT型の場合は、フィールド値を以下のように比較します。

> Example:
<pre>
field6.member3 = 100
</pre>

#### LIST型フィールドの比較

TupleのフィールドがLIST型の場合は、フィールド値を以下のように比較します。

> Example:
<pre>
field4[0] = 'tokyo'
</pre>

#### MAP型フィールドの比較

TupleのフィールドがMAP型の場合は、フィールドの値を以下のように比較します。

> Example:
<pre>
field7['visa'] = true
</pre>

#### 定数 ★

条件に指定できる定数は、以下になります。

* 文字列
シングルクォートまたはダブルクォートでくくった文字列

* INT値
> <pre>
number only
</pre>
>
> Example:
> <pre>
2147483647
</pre>

* DOUBLE値
> <pre>
number.number
</pre>
>
> Example:
> <pre>
12.5
</pre>

* BIGINT値
> <pre>
numberL
</pre>
>
> Example:
> <pre>
9223372036854775807L
</pre>

* SMALLINT値
> <pre>
numberS
</pre>
>
> Example:
> <pre>
32767S
</pre>

* TINYINT値
> <pre>
numberY
</pre>
>
> Example:
> <pre>
255Y
</pre>

* FLOAT値
> <pre>
number.numberF
</pre>
>
> Example:
> <pre>
12.5F
</pre>

* BOOLEAN値
> <pre>
true|false
</pre>

---

### FILTER GROUP

FILTER GROUP は、複数のTupleに対してTupleの通過を判定します。

<pre>
FILTER GROUP EXPIRE period [STATE TO state_field]
  condition, ...
</pre>

* period には、フィルタの状態を保持する期間を指定します。
* state_field には、フィルタの状態を出力するフィールド名を指定します。フィルタを通過すると、指定したフィールド名で
Tupleに状態フィールドを追加します。STATE TO clause を省略した場合は、状態フィールドは追加しません。 ★
* condition には、フィルタ条件を指定します。カンマ区切りで、複数のTupleに対する条件を指定します。
カンマ区切りで指定した条件をすべて満たせば、Tupleはフィルタを通過します。
すべての条件を満たした場合、最後に到着したTupleがフィルタを通過し、フィルタの状態は初期化されます。

> Example:
<pre>
FILTER GROUP EXPIRE 7DAYS STATE TO fg_state
  ua1.field1 >= 10 AND ua1.field8 = true,
  (ua2.field1 <= 30 AND ua2.field2.member2 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
</pre>

> ua1, ua2, ua3の３つのTupleに対してフィルタを実行します。条件１と条件２（※）の両方を満たせば、Tupleはフィルタを通過します。
> （※）条件１はua1に対するフィルタで、条件２はua2とua3に対するフィルタです。
>
> 条件２は、ua2とua3の条件を"OR"で指定しているので、ua1かつua2の条件を満たすか、ua1かつua3の条件を満たすことで
> すべての条件は満たされます。
>
> 条件１もしくは条件２のいずれかを満たした状態を７日間保持します。
> 例えば、条件１を最後に満たした状態で、条件２の条件を満たすまでの期間が７日以内であれば、条件１と条件２は満たされますが、
> ８日以上の日数が経過していた場合、条件１の状態は初期化されてしまう為、条件２のみ満たしている状態になります。
>
> state_fieldに"fg_state"を指定しているので、フィルタの状態を"fg_state"フィールドとしてTupleに追加します。
> fg_stateは、条件１と条件２のそれぞれを満たした日時が格納されます。条件の数と等しいTIMESTAMPのLISTになります。

#### period

* 秒で指定
> <pre>
number(SECONDS|SEC)
</pre>
>
> Example:
> <pre>
30SECONDS
</pre>

* 分で指定
> <pre>
number(MINUTES|MIN)
</pre>
>
> Example:
> <pre>
55MIN
</pre>
  
* 時間で指定
> <pre>
number(HOURS|H)
</pre>
>
> Example:
> <pre>
55HOURS
</pre>

* 日で指定
> <pre>
number(DAYS|D)
</pre>
>
> Example:
> <pre>
15DAYS
</pre>

---

### EACH

EACH は、Tupleの集計や編集を実行します。

  EACH expr, ...

* exprには、集計関数もしくは編集関数、フィールドのアクセサを指定します。

#### 集計関数

* Tupleの到着数をカウントします。
>
> Example:
> <pre>
EACH count() AS cnt1
</pre>

* 到着したフィールドの値を合計します。
>
> Example:
> <pre>
EACH sum(field1) AS sum1
</pre>

* 到着したフィールドの値の平均を計算します。
>
> Example:
> <pre>
EACH avg(field1) AS avg1
</pre>

#### 編集関数

* フィールドの値がNULLであれば、代替えの値で置き換えます。
>
> Example:
> <pre>
EACH ifnull(field1, 0) AS field1
</pre>

* STRING型のフィールドの値を連結したフィールドを作成します。
>
> Example:
> <pre>
EACH concat(field1, '-', field2) AS new_field
</pre>

#### 関数の引数 ★

定数に関しては、関数の種類が増えてきてから改めて記述する。

#### フィールドのアクセサ

> Example:
<pre>
EACH field1, field6.member1 AS field10, field7['visa'] AS visa
</pre>

> field1はそのまま、field6.member1をfield10フィールドへ、field7['visa']をvisaフィールドへ抽出します。

---

### GROUP

BEGIN GROUP ... END GROUP で囲まれたクエリを、グループで実行します。

<pre>
BEGIN GROUP BY field, ...
[END GROUP | TO STREAM]
</pre>

* field には、グループ化するフィールドの名前を指定します。

#### EACH をグループで実行する

> Example:
<pre>
BEGIN GROUP BY user_name
EACH user_name, count() AS gc1
EMIT * USING mongo_persist('db1', 'col2', 'user_name');
END GROUP
</pre>

> user_name ごとに（ユーザごとに）Tupleがカウントされます。

#### FILTER GROUP をグループで実行する

> Example:
<pre>
BEGIN GROUP BY user_name
FILTER GROUP EXPIRE 1DAYS
  ua1.field1 >= 10 AND ua1.field8 = true,
  (ua2.field1 <= 30 AND ua2.field2.member3 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
EMIT * USING mongo_persist('db1', 'col3')
END GROUP
</pre>

> user_nameごとに（ユーザごとに）フィルタが判定されます。
> 特定のユーザが条件１と条件２を満たしているかを判定し、FILTER GROUP の状態はユーザごとに保持されます。

#### GROUP のネスト

GROUPはネストできます。

> Example:
<pre>
EACH ...  <- グループ化せずに実行
BEGIN GROUP BY date
  EACH ...  <- date ごとに実行される
  BEGIN GROUP BY area
    EACH ...  <- date + area ごとに実行される
  END GROUP 
END GROUP
EACH ...  <- グループ化せずに実行
</pre>

TO STREAM で、すべてのグループを解除します。

> Example:
<pre>
EACH ...  <- グループ化せずに実行
BEGIN GROUP BY date
  EACH ...  <- date ごとに実行される
  BEGIN GROUP BY area
    EACH ...  <- date + area ごとに実行される
TO STREAM
EACH ...  <- グループ化せずに実行
</pre>

END GROUP と TO STREAM は、グループ化を解除する必要がなければ省略可能です。 ★

---

### EMIT ★

EMITは、Tupleを外部へ出力します。

  EMIT output_field, ... USING emit_processor

* output_field には、出力するフィールド名を指定します。ワイルドカード（*）を指定できます。
* emit_processor には、出力に使用するプロセッサを指定します。

> Example:
<pre>
EMIT field1, field2, field3 USING mongo_persist('db1', 'col1')
</pre>

#### Emit Processor

* Kafka Emit Processor
> TupleをKafkaに出力します。
>
> <pre>
kafka_emit(topic_name)
</pre>
>
> * topic_name には、出力するTopic名を指定します。topic_name はプロセッサ変数に対応しています。
>
> Example:
> <pre>
kafka_emit('topic1')
</pre>

* Mongo Persist Processor

> TupleをMongoDBに出力します。
>
> <pre>
mongo_persist(db_name, collection_name [, key_names])
</pre>
>
> * db_name には、出力するDB名を指定します。db_name はプロセッサ変数に対応しています。
> * collection_name には、出力するCollection名を指定します。collection_name はプロセッサ変数に対応しています。
> * key_names には、出力するキーのフィールド名を指定します。複合キーの場合は配列で指定してください。
> key_names を指定した場合、出力はキーに対してupdateされます。
> key_names を指定しなかった場合は、出力はinsertになります。
>
> Example:
> <pre>
mongo_persist('db1', 'col1')  <- insert
mongo_persist('db1', 'col1', 'field2') <- field2 をキーとしてupdate
mongo_persist('db1', 'col1', ['field2', 'field3']) <- field2 + field3 を複合キーとしてupdate
</pre>

#### プロセッサ変数

Emit Processor の出力先の名称には、以下のプロセッサ変数を含めることができます。

${TOPOLOGY_ID} は、起動中のTopology IDに置き換えられます。
${ACCOUNT_ID} は、Topologyを起動したユーザのAccount IDに置き換えられます。

> Example:
<pre>
kafka_emit('topic_${TOPOLOGY_ID}')
</pre>
