---
layout: manual_ja
title: genn.ai
---

# genn.ai DML

> genn.aiは、大きくRESTサーバとクエリサーバから構成され、
	前者RESTサーバは、httpにて外部からjsonのデータを受け取りStormに渡します(正確には、そのためにKafkaに格納します)。
	後者クエリサーバは、genn.ai独自の **クエリ** で書かれたイベント処理ロジックをコンパイルし、Stormに登録します。
	(この処理ロジックでは、通常、最初にKafkaからデータを読み出します)

> genn.aiでは、このjson形式で受け取るデータを **タプル** (Tuple)と呼び、コンパイルにより出来上がるものは **トポロジ** (Topology)と呼びます。
	(タプルについてはStormの用語をそのまま借りています)

> 本ページのDMLとは、後者クエリサーバが担当するクエリ(の文法)のことを指します。
    前者の **タプル** については [DDL](ddl_ja.html)のページをご参照下さい。

## FROM

FROM を使い、Tupleの入力元をスキーマとともに指定します。

### 入力先が外部の場合

    FROM schema_name AS schema_alias, ... USING spout_processor


* schema_name には、Tuple名もしくはView名を指定します。
* schema_alias には、クエリ内で使用するTupleもしくはViewの別名を指定します。
* spout_processor には、読み込みに使用するプロセッサを指定します。

> Example:
>
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout()


外部入力は、一つのTopologyに対して一つしか定義できません。

#### Kafka Spout Processor

TupleをKafkaから読み込みます。システムのデフォルトとして動作します。

    kafka_spout()


### 入力先が内部（ストリーム）の場合

    FROM stream_name[(schema_alias, ...)], ...

* ストリームとは別に定義したトポロジからの入力のことです。
* stream_name には、入力したいストリーム名を指定します。

ストリームからすべてのTupleを読み込む場合

> Example:
>
    FROM s1, s2


ストリームから特定のTupleのみを読み込む場合

> Example:s1ストリームからua1, ua2タプルのみ読み込み、s2ストリームからはv1タプルのみ読み込む場合
>
    FROM s1(ua1, ua2), s2(v1)

### TupleJoin

複数のTupleをフィールドの値を元に結合します。

#### 外部入力からTupleを読み込んで結合

外部入力からTupleを読み込む時点で、複数のTupleを結合します。

    FROM (schema_name_1
      JOIN schema_name_2 ON join_condition
      TO join_field
      EXPIRE period
    ) AS schema_alias USING spout_processor

    join_condition:
    schema_name_1.key_field = schema_name_2.key_field

    join_field:
    schema_name_1.join_field [AS field_alias, ...]

* schema_name_1, schema_name_2は結合したいTupleを指定します。
* join_conditionにはTupleの結合条件を指定します。複数条件の場合にはANDで指定します。
* join_fieldには結合後のTupleが保持するフィールドを指定します。結合後のフィールド名称が被らない場合には、ワイルドカードを使用する事やaliasを省略することが可能です。
* periodには、Tupleを保持する期間を指定します。指定した時間経過後にJoin対象のTupleが読み込まれると、Tuple Joinは実行されません。

> Example: ua1, ua2, ua3の3つのTupleを結合して、ua4のTupleを作成
>
    FROM (ua1
      JOIN ua2 ON ua1.field1 = ua2.field3
      JOIN ua3 ON ua1.field2 = ua3.field5
      TO ua1.*, ua2.field4 AS field4, ua3.field7 AS field7
      EXPIRE 1min
    ) AS ua4 USING kafka_spout()

#### 内部入力(ストリーム)からTupleを読み込んで結合

内部入力から、一部のTupleに対してTupleを結合します。

    FROM (stream_name_1[(schema_name_1, ...)]
      JOIN stream_name_2[(schema_name_2, ...)] ON join_condition
      TO join_field
      EXPIRE period
    ) AS schema_alias

    join_condition:
    schema_name_1.key_field = schema_name_2.key_field

    join_field:
    schema_name_1.join_field [AS field_alias, ...]

* 内部入力に複数のTupleが含まれる場合、schema_nameを指定して絞り込む必要があります。
* join_fieldには結合後のTupleが保持するフィールドを指定します。結合後のフィールド名称が被らない場合には、ワイルドカードを使用する事やaliasを省略することが可能です。
* periodには、Tupleを保持する期間を指定します。指定した時間経過後にJoin対象のTupleが読み込まれると、Tuple Joinは実行されません。

> Example: ua1,ua2をストリームs1から読み込み、ua3をストリームs2から読み込んで結合
>
    FROM (s1(ua1)
      JOIN s1(ua2) ON ua1.field1 = ua2.field3
      JOIN s2(ua3) ON ua1.field2 = ua3.field5
      TO ua1.*, ua2.field4 AS field4, ua3.field7 AS field7
    ) AS ua4

---

## INTO

INTO は、ストリームを分岐・合流させます。

    INTO stream_name

* stream_name には、出力するストリーム名を指定します。

> Example:
>
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1

INTO を使って出力したストリームは、FROM で読み込みます。

### 分岐, 合流

分岐の例を以下に示します。

    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1;
    FROM s1(ua1) ...
    FROM s1(ua2) ...
    FROM s1(v1) ...

合流の例を以下に示します。

    FROM s1(ua1) ... INTO s2;
    FROM s1(ua2) ... INTO s3;
    FROM s1(v1) ... INTO s4;
    FROM s2, s3, s4 ...

---

## JOIN

JOINは、外部データをフィールドとしてTupleに結合します。

    JOIN join_name ON join_condition TO join_fields USING fetch_processor

    join_condition:
    join_name.key_field = field [AND join_name.key_field = field AND join_name.key_field <> 0]

    join_fields:
    join_name.join_field [AS field_alias, ...]


* join_name には、結合する名称を指定します。
join_condition や join_fields で、外部データのフィールドを識別する為に使用します。
* join_condition には、結合条件を指定します。
結合データのキーフィールド = Tupleのフィールド を、結合データが一意になるように指定してください。
複合条件の場合は、AND で指定します。結合データのキーフィールドに対して、定数で条件を指定することもできます。
* join_fields には、結合するフィールドを全て指定します。
結合データのフィールド名 AS Tupleに結合する際のフィールド名 を指定するか、フィールド名だけを指定します。
フィールドはTupleに追加されます。
* field_alias は、省略可能です。省略すると元データのフィールド名で結合されます。

> Example: MongoDBデータとの結合
>
    JOIN j1 ON j1.code1 = field1 AND j1.code2 = field2 AND j1.del = 0
      TO j1.name AS field10, j1.type AS field11
      USING mongo_fetch('db1', 'col1')

### MongoDBデータとの結合

#### Mongo Fetch Processor

今現在、MongoDB上にあるデータとの結合が可能です。
なお、MondoDBから受け取られたデータはEXPIRE句を合わせて指定することでキャッシュされます。
(EXPIRE句の指定が無い場合はキャッシュは行われません)

指定した時間の間、fetchした内容をキャッシュします。キャッシュしている間に実行されたJOINは、キャッシュから結合フィールドを取得します。
指定した時間が過ぎると、ふたたびFetchProcessorを実行してキャッシュを更新します。

キャッシュは結合キーごとに保存されます。

    JOIN join_name ON join_condition
      TO join_fields
      EXPIRE period
      USING mongo_fetch(db_name, collection_name)

* db_nameは、入力とするDB名を指定します。
* collection_nameは、入力とするCollection名を指定します。

> Example:
>
    JOIN j1 ON j1.code1 = field1 AND j1.code2 = field2 AND j1.del = 0
      TO j1.name AS field10, j1.type AS field11
      EXPIRE 1min
      USING mongo_fetch('db1', 'col1')


### Webサービスとの結合

#### Web Fetch Processor

JSON形式のレスポンスを返すWebサービスとの結合か可能です。
結合条件をクエリパラメータに変換し、指定したURLにアクセスします。
取得したJSON形式のレスポンスデータの一部を抜き出し、Tupleと結合します。

    JOIN join_name ON join_condition
      TO join_fields
      EXPIRE period
      USING web_fetch(url, replace, path)

* urlには、データを取得するURLを指定します。${query}変数を使用して、URLにクエリパラメータを追加します。
* replaceには、クエリパラメータの置換ルールを配列で指定します。[変換前文字列1, 変換後文字列1, 変換前文字列2, 変換後文字列2, ...]の形式で指定します。
* pathには、取得したJSONデータのルートパスを指定します。

> Example: Solrのデータと結合する
>
    JOIN books ON books.title = ccc AND books.author = ddd
      TO books.id AS book_id, books.price AS price
      EXPIRE 10min
      USING web_fetch('http://localhost:3000/solr/select?q=${query}', [' = ',':', ' AND ', '+AND+'], 'response.docs[0]')
>
> 置換ルール1: ' = 'を':'に置換
> 置換ルール2: ' AND 'を'+AND+'に置換
>
> 元の結合条件:
>
    books.title = ccc AND books.author = ddd
> 置換ルール適用後の結合条件:
>
    books.title:ccc+AND+books.author:ddd
> ※ ccc, dddの各フィールドは、Tupleの各フィールドの値に置き換えられます。
>
> アクセスURL:
>
    http://localhost:3000/solr/select?q=books.title:ccc+AND+books.author:ddd
> 
> レスポンスデータ: pathで指定したパスをルートパスとしてJSONのパースを行います。
>
    {
      response:{
        docs:[
          {　←★ここから読み始める
            id:"978-1423103349",
            title:"The Sea of Monsters",
            author:"Rick Riordan",
            price:6.49
          }
        ]
      }
    }

---

## FILTER

FILTER は、単一のTupleに対して通過の可否を判定します。

    FILTER condition

* condition に、フィルタの条件を指定します。

condition の符号には、以下のものを指定します。

* &#61; もしくは &#61;&#61;
* <> もしくは !&#61;
* &gt;
* &gt;=
* &lt;
* &lt;=
* LIKE
* REGEXP
* IN
* ALL
* BETWEEN
* AND
* OR
* NOT

### &#61;, &#61;&#61;, <>, !&#61;, >, >&#61;, <, <&#61;

論理比較を行います。

> Example:
>
    field1 >= 10

### LIKE, REGEXP

LIKEで使用できるワイルドカードは、"%"（複数文字）と"_"（一文字）です。

> Example:
>
    field2 LIKE 't%'

REGEXPで使用できる正規表現は、[java/util/regex/Patternと同じ書式](http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html)
を採用しています。

> Example:
>
    field3 REGEXP '^[A-Z]{2}-[0-9]{4}$'

### IN, ALL, BETWEEN

INは、LISTフィールドに対して値が一つでも含まれているかを調べます。

> Example:
>
    field4 IN ('tokyo', 'kyoto', 'osaka')

ALLは、LISTフィールドに対して値がすべて含まれているかを調べます。

> Example:
>
    field4 ALL ('tokyo', 'kyoto', 'osaka')

BETWEENは、INT型での範囲を指定します。

> Example:
>
    field1 10 AND 100


### AND, OR, NOT

AND, OR, NOT は入れ子にすることが可能です。優先順位はNOT, AND, ORの順に処理されます。
優先順位は、()を使用して変更できます。

> Example:
>
    field1 <= 30 AND (field5 BETWEEN 10 AND 100 OR field2 LIKE 'A%')

### その他(比較)

以下、基本型以外での比較方法を示します。

#### STRUCT型フィールドの比較

TupleのフィールドがSTRUCT型の場合は、フィールド値を以下のように比較します。

> Example:
>
    field6.member3 = 100

#### LIST型フィールドの比較

TupleのフィールドがLIST型の場合は、フィールド値を以下のように比較します。

> Example:
>
    field4[0] = 'tokyo'

#### MAP型フィールドの比較

TupleのフィールドがMAP型の場合は、フィールドの値を以下のように比較します。

> Example:
>
    field7['visa'] = true


#### 定数

条件に指定できる定数は、以下になります。

* 文字列

    シングルクォートまたはダブルクォートでくくった文字列

* INT値

    number only

    > Example:
    >
      2147483647

* DOUBLE値

    number.number

    > Example:
    >
      12.5

* BIGINT値

    numberL

    > Example:
    >
      9223372036854775807L

* SMALLINT値

    numberS

    > Example:
    >
      32767S

* TINYINT値

    numberY

    > Example:
    >
      255Y

* FLOAT値

    number.numberF

    > Example:
    >
      12.5F

* BOOLEAN値

    > Example:
    >
      true|false


---

## FILTER GROUP

FILTER GROUP は、複数のTupleに対してTupleの通過を判定します。

    FILTER GROUP EXPIRE period [STATE TO state_field]
      condition, ...


* period には、フィルタの状態を保持する期間を指定します。
* state_field には、フィルタの状態を出力するフィールド名を指定します。フィルタを通過すると、指定したフィールド名で
Tupleに状態フィールドを追加します。STATE TO clause を省略した場合は、状態フィールドは追加しません。
* condition には、フィルタ条件を指定します。カンマ区切りで、複数のTupleに対する条件を指定します。
カンマ区切りで指定した条件をすべて満たせば、Tupleはフィルタを通過します。
すべての条件を満たした場合、最後に到着したTupleがフィルタを通過し、フィルタの状態は初期化されます。

> Example:
>
    FILTER GROUP EXPIRE 7DAYS STATE TO fg_state
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member2 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'


 ua1, ua2, ua3の３つのTupleに対してフィルタを実行します。
 条件１と条件２（※）の両方を満たせば、Tupleはフィルタを通過します。
 （※）条件１はua1に対するフィルタで、条件２はua2とua3に対するフィルタです。

 条件２は、ua2とua3の条件を"OR"で指定しているので、ua1かつua2の条件を満たすか、ua1かつua3の条件を満たすことで
 すべての条件は満たされます。

 条件１もしくは条件２のいずれかを満たした状態を７日間保持します。
 例えば、条件１を最後に満たした状態で、条件２の条件を満たすまでの期間が７日以内であれば、条件１と条件２は満たされますが、
 ８日以上の日数が経過していた場合、条件１の状態は初期化されてしまう為、条件２のみ満たしている状態になります。

 state_fieldに"fg_state"を指定しているので、フィルタの状態を"fg_state"フィールドとしてTupleに追加します。
 fg_stateは、条件１と条件２のそれぞれを満たした日時が格納されます。
 条件の数と等しいTIMESTAMPのLISTになります。

### 状態の保持

period を用いて、フィルタの状態をどれだけ保持するか、を指定します。

* 秒で指定

    number(SECONDS|SEC)

    > Example:
    >
      30SECONDS

* 分で指定

    number(MINUTES|MIN)

    > Example:
    >
      55MIN

* 時間で指定

    number(HOURS|H)

    > Example:
    >
      55HOURS

* 日で指定

    number(DAYS|D)

    > Example:
    >
      15DAYS

---

## EACH

EACH は、Tupleの集計や編集を実行します。

    EACH expr, ...

* exprには、集計関数や編集関数、四則演算、フィールドのアクセサを指定します。

### フィールドのアクセサ

> Example:
>
    EACH field1, field6.member1 AS field10, field7['visa'] AS visa

field1はそのまま、field6.member1をfield10フィールドへ、field7&#91;'visa'&#93;をvisaフィールドへ抽出します。


### 集計関数

#### count

Tupleの到着数をカウントします。

    count([DISTINCT] [field])

* fieldを指定すると、指定したフィールドが存在するTupleのみカウント対象となります。
* DISTINCTを指定すると、指定したフィールドの一意なTupleをカウントします。

> Example: 到着したTupleを全てカウント
>
    EACH count() AS cnt1

> Example: 到着したTupleのうち、field1フィールドが存在するTupleをカウント
>
    EACH count(field1) AS cnt2

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleをカウント
>
    EACH count(DISTINCT field1) AS cnt3

> Example: 複数のフィールドに対して、一意なTupleをカウント
>
    EACH count(DISTINCT concat(field1, ifnull(field2, -))) AS cnt4


#### sum

到着したフィールドの値を合計します。

    sum([DISTINCT] field)

* fieldの指定は必須です。指定したフィールドが存在するTupleの値を合計します。
* DISTINCTを指定すると、指定したフィールドの一意なTupleの値を合計します。

> Example: 到着したTupleのうち、field1フィールドが存在するTupleの値を合計
>
    EACH sum(field1) AS sum1

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleの値を合計
>
    EACH sum(DISTINCT field1) AS sum2

#### avg

到着したフィールドの値の平均を計算します。

    avg([DISTINCT] field)

* fieldの指定は必須です。指定したフィールドが存在するTupleの値の平均を計算します。
* DISTINCTを指定すると、指定したフィールドの一意なTupleの値の平均を計算します。

> Example: 到着したTupleのうち、field1フィールドが存在するTupleの値の平均を計算
>
    EACH avg(field1) AS avg1

> Example: 到着したTupleのうち、field1フィールドが存在し、かつ一意なTupleの値の平均を計算
>
    EACH avg(DISTINCT field1) AS avg2

### 編集関数

#### ifnull

フィールドの値がNULLであれば、代替の値で置き換えます。

> Example:
>
    EACH ifnull(field1, 0) AS field1

#### concat

STRING型のフィールドの値を連結したフィールドを作成します。

> Example:
>
    EACH concat(field1, '-', field2) AS new_field

#### split

フィールドの値を区切り文字列もしくは正規表現によってリストに変換します。

    split(string str, string pattern)

* strには、対象フィールドを指定します。
* patternには、区切り文字列もしくは正規表現で指定します。

> Example:
>
    EACH split(field1, ',') AS new_list

#### regexp_extract

フィールドの値から正規表現によって値を抜き出します。

    regexp_extract(string subject, string pattern, int index)

* subjectには、対象フィールドを指定します。
* patternには、正規表現パターンを指定します。
* indexには、抜き出すグループの番号を指定します。

> Example:
>
    EACH regexp_extract(field1, '(\d{4})-(\d{2})-(\d{2})', 1) AS new_field

#### parse_url

フィールドの値(URL)をパースして、一部を返します。

    parse_url(string urlString, string partToExtract [, string keyToExtract])

* urlStringには、パースするURLフィールドを指定します。
* partToExtractには、下記のいずれかを指定します。
    * HOST
    * PATH
    * QUERY
    * REF
    * PROTOCOL
    * AUTHORITY
    * FILE
    * USERINFO
* keyToExtractには、partToExtractがQUERYの場合、取得するパラメータ名称を指定します。

> Example:
>
    EACH parse_url(urlField, 'QUERY', 'k1') AS new_field

#### cast

フィールドの値を指定した型に変換します。

    cast(expr as type)

* exprには、対象フィールドを指定します。
* typeには、下記の型のみ指定可能です。
    * TIMESTAMP
    * TINYINT
    * SMALLINT
    * INT
    * BIGINT
    * FLOAT
    * DOUBLE
    * BOOLEAN

> Example:
>
    EACH cast(field1 AS TIMESTAMP('yyyyMMddHHmmss')) AS new_field

![Alt text](/img/underconstruction.png)
定数に関しては、関数の種類が増えてきてから改めて記述する。

#### date_format

TIMESTAMP型のフィールドを、フォーマットした文字列(STRING)に変換します。

    date_foramt(field AS string)

* fieldには、TIMESTAMP型のフィールドを指定します。
* stringには、パターン文字列を指定します。

パターン文字列には、[java/text/SimpleDateFormat](https://docs.oracle.com/javase/jp/6/api/java/text/SimpleDateFormat.html)と同じ書式を採用しています。

> Example:
>
    EACH date_format(field1, 'yyyy-MM-dd-HH-mm') field2

> Example: 日時文字列(yyyyMMdd)をTIMESTAMP型に変換し、日付のみ取得
>
    EACH date_format(cast(field1 AS TIMESTAMP('yyyyMMdd')) AS 'dd') AS field2

### 四則演算

数値型フィールドでは下記の四則演算が可能です。

* 加算(+)
* 減算(-)
* 乗算(*)
* 除算(/)
* 除算(DIV)
* 剰余(MOD, %)

#### 数値型フィールド同士の演算

数値型フィールド同士での四則演算が可能です。

> Example:
>
    EACH field1 + field2 AS field3
    EACH field1 DIV field2 AS field3

#### 数値型フィールドと数値の演算

数値型フィールドと数値の四則演算が可能です。

> Example:
>
    EACH field1 * 2 AS field3

#### 関数を組み合わせた演算

数値型フィールドと集計関数を組み合わせた四則演算が可能です。

> Example:
>
    EACH sum(field1 * 10) * count() AS field3

#### 演算の優先順位

括弧を用いて、四則演算の優先順位を変更する事が可能です。

> Example:
>
    EACH field1 * (field2 + 123) AS field3

---

## LIMIT

Tupleの流れを制限します。
現在、時間による制限と、タプル数による制限の方法があります。

    LIMIT [FIRST|LAST] period

* periodには、時間もしくはTuple数を指定します。
* FIRSTは、指定範囲内に最初に到着したTupleを後続の処理に渡します。
* LASTは、指定範囲内の最後に到着したTupleを後続の処理に渡します。

### 時間による流量制限

* 時間の起点は、最初にTupleが到着した時間です。
* 起点は指定時間が経過した以降にリセットされます。

最初に到着したTupleを後続に送り、以降30分はTupleを送らない。

> Example:
>
    LIMIT FIRST EVERY 30min

Tupleが最初に到着してから、30分間に到着した最後のTupleを後続に送る。

> Example:
>
    LIMIT LAST EVERY 30min

* 時間の起点は、最初にTupleが届いた時になります。起点は30分経過後にリセットされます。
* 30分経過したかどうかは、30分経過後に初めてTupleが届いた時点で判断されます。　　

### Tuple数による流量制限

5件のTupleの中で、最初に到着したTupleを後続に送る。

> Example:
>
    LIMIT FIRST EVERY 5

5件のTupleの中で、最後に到着したTupleを後続に送る。

> Example:
>
    LIMIT LAST EVERY 5

 * いずれもカウンタは５件受け取るごとにクリアされます。

#### 補足

LIMITを使うことによって、
LIMIT FIRSTであれば、１度EMIT（通知）したTupleを一定時間EMITさせないように制限したり、
LIMIT LASTであれば、最初にアクションし始めてから、４時間の間で一番最後に行ったアクションを抽出したりすることができます。

ただし、LIMIT LASTの場合、Tupleの通過のトリガーが、時間経過後に初めてきたTupleになるので、通過までタイムラグが発生します。
例えば、30分間の最後にTupleが届き、次のTupleが届くまでに１週間かかった場合、その最後のTupleが送られるのは１週間後になります。

---

## SLIDE

集計関数によるスライド集計を実行します。
間隔は時間もしくはTuple数を指定することが可能です。
スライドはTupleの到着時にのみ実行されます。

    SLIDE period [BY time_field| count] expr

* periodには、スライド時間もしくはタプル数を指定します。
* exprには、集計関数や編集関数、四則演算、フィールドのアクセサを指定します。

### 時間でスライド

    SLIDE LENGTH period BY time_field expr

* time_fieldには、起点となるTIMESTAMP型のフィールドを指定します。

> Example:
>
    SLIDE LENGTH 10sec BY _time sum(field1) AS new_field

LENGTH スライド時間 BY 起点となるフィールド名 集計関数
といった書式で指定します。

* 起点となるフィールド名には、Timestamp型のフィールドを指定します。
  _timeフィールドも指定できます。（スキーマに_timeフィールドを指定していた場合）
* 集計関数は複数指定できます。現状では、sum(), avg(), count()の３つの集計関数が使用できます。

### Tuple数でスライド

> Exapmle:
>
    SLIDE LENGTH 100 sum(field2) AS new_field

LENGTHにスライドするTupleの数を指定します。

スライドはTupleの到着時にのみ実行されます。（スライドによる再計算も到着時のみ）

処理的には、Tupleが到着する度にTupleをウィンドウ領域に貯めつつ、都度集計を行なっていきます。
その際、ウィンドウ幅から除外されたTuple（※）を集計結果から減算します。
（※）スライドして集計対象から外れたTuple

ウィンドウ領域に保存するTupleは、集計に必要なフィールドのみを選択しています。

![Alt text](/img/underconstruction.png)
スライドに使用する領域は現在メモリになっていますが、将来的に外部DBへの差し替えを予定。


---

## SNAPSHOT

一定間隔による集計を実行します。間隔は時間もしくはTuple数を指定することが可能です。

    SNAPSHOT EVERY period expr

* periodには、時間・Tuple数を指定します。
* exprには、集計関数や編集関数、四則演算、フィールドアクセサを指定します。

### 時間ベースでの実行

* cron形式でも指定することができます。
* 集計値は指定時間毎にリセットされます。

cronの書式は、[Quartz](http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/crontrigger)の書式を採用しています。

> Example: 7分毎に実行
>
    SNAPSHOT EVERY 7min sum(field1) AS new_field
    SNAPSHOT EVERY "* */7 * * * ? *" sum(field1) AS new_field

集計はTupleが送られてくる度に実行されますが、集計結果は７分に１回だけ次の処理に送られます。
下段の例は、時間をcron形式で指定したものです。（注：ダブルクォートでくくる必要があります）
７分の間に１度もTupleが送られてこなかった場合は、集計結果が生成されない為、Tupleは次の処理に送られません。
集計結果は７分毎にリセットされます。

> Example: 毎日0時に日次の集計結果を返す
>
    SNAPSHOT EVERY "0 0 0 * * ? *" sum(bbb) AS s, count() AS c

### Tuple数ベースでの実行

* 指定したTuple数の集計を実行し、後続の処理に結果を送ります。

> Example:
>
    SNAPSHOT EVERY 10 sum(field1) AS new_field

指定した回数分のTupleが届いたタイミングで、集計結果が次の処理に送られます。
集計結果は10件毎にリセットされます。

---

## GROUP

BEGIN GROUP ... END GROUP で囲まれたクエリを、グループで実行します。

    BEGIN GROUP BY field, ...
      [END GROUP | TO STREAM]

* field には、グループ化するフィールドの名前を指定します。

### EACH をグループで実行する

> Example:
>
    BEGIN GROUP BY user_name
    EACH user_name, count() AS gc1
    EMIT * USING mongo_persist('db1', 'col2', 'user_name');
    END GROUP

user_name ごとに（ユーザごとに）Tupleがカウントされます。

### FILTER GROUP をグループで実行する

> Example:
>
    BEGIN GROUP BY user_name
    FILTER GROUP EXPIRE 1DAYS
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member3 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
    EMIT * USING mongo_persist('db1', 'col3')
    END GROUP

user_nameごとに（ユーザごとに）フィルタが判定されます。
特定のユーザが条件１と条件２を満たしているかを判定し、FILTER GROUP の状態はユーザごとに保持されます。

### GROUP のネスト

GROUPはネストできます。

> Example:
>
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
      END GROUP
    END GROUP
    EACH ...  <- グループ化せずに実行

TO STREAM で、すべてのグループを解除します。

> Example:
>
    EACH ...  <- グループ化せずに実行
    BEGIN GROUP BY date
      EACH ...  <- date ごとに実行される
      BEGIN GROUP BY area
        EACH ...  <- date + area ごとに実行される
    TO STREAM
    EACH ...  <- グループ化せずに実行

END GROUP と TO STREAM は、グループ化を解除する必要がなければ省略可能です。

---

## EMIT

EMITは、Tupleを外部へ出力します。

  EMIT output_field, ... USING emit_processor

* output_field には、出力するフィールド名を指定します。ワイルドカード（&#42;）を指定できます。
* emit_processor には、出力に使用するプロセッサを指定します。

> Example:
>
    EMIT field1, field2, field3 USING mongo_persist('db1', 'col1')

### 出力先が外部の場合

#### Kafka Emit Processor

TupleをKafkaに出力します。

    kafka_emit(topic_name[, mode])


* topic_name には、出力するTopic名を指定します。topic_name はプロセッサ変数に対応しています。
* mode には、jsonもしくはcsvを指定できます。省略した場合はjsonが適用されます。

> Example:
>
    kafka_emit('topic1')

また、Kafka上の書き出しは標準ではJSONになりますが、CSVでの書き出しも可能です。
このとき、出力の並びはEXPLAINで確認できる並びとなります。

> Example:
>
    EMIT * USING kafka_emit('topic1', 'csv');


#### Mongo Persist Processor

TupleをMongoDBに出力します。

    mongo_persist(db_name, collection_name [, key_names])


* db_name には、出力するDB名を指定します。db_name はプロセッサ変数に対応しています。
* collection_name には、出力するCollection名を指定します。collection_name はプロセッサ変数に対応しています。
* key_names には、出力するキーのフィールド名を指定します。複合キーの場合は配列で指定してください。
 key_names を指定した場合、出力はキーに対してupdateされます。
 key_names を指定しなかった場合は、出力はinsertになります。

> Example:
>
    mongo_persist('db1', 'col1')  <- insert
    mongo_persist('db1', 'col1', 'field2') <- field2 をキーとしてupdate
    mongo_persist('db1', 'col1', ['field2', 'field3']) <- field2 + field3 を複合キーとしてupdate

#### Web Emit Processor

Tupleを他のRESTサーバに向けて送信します。

    web_emit(url_name)


 * url_name には、出力する先のURLを指定します。

現在、Elasticsearchに格納する場合、第二引数に"es"を指定すると、[blukインサート](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bulk.html) に対応した書式で投げ込みます(第三引数にメタデータを指定して下さい)。

> Example:
>
    EMIT * USING web_emit('http://localhost:9200/_bulk', 'es', {'index':'test', 'type':'xyz'});

#### Schema Persist Processor

Tupleを同一アカウントの他のスキーマに出力します。

    EMIT fields TO tuple_name;

* tuple_name には、出力先のスキーマの名称を指定します。
* 出力先のスキーマと型、順序を一致させる必要があります。

> Example: スキーマ定義
>
    CREATE TUPLE tuple1 (aaa STRING, bbb INT, ccc INT, ddd STRING, _time);
    CREATE TUPLE tuple2 (aaa STRING, bbb INT, ccc INT, ddd STRING);
    CREATE TUPLE tuple3 (bbb INT, ccc INT, ddd STRING);
> Example 出力側
>
    FROM tuple1, tuple2 USING kafka_spout()
    ...
    EMIT bbb, ccc, ddd TO tuple3;

> Example: 入力側
>
    FROM tuple3 USING kafka_spout()
    ...


#### プロセッサ変数

Emit Processor の出力先の名称には、以下のプロセッサ変数を含めることができます。

${TOPOLOGY_ID} は、起動中のTopology IDに置き換えられます。
${ACCOUNT_ID} は、Topologyを起動したユーザのAccount IDに置き換えられます。

> Example:
>
    kafka_emit('topic_${TOPOLOGY_ID}')
