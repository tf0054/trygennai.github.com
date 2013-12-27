---
layout: manual
title: genn.ai
---

## LanguageManual DDL

This page describes the DDL language supported by genn.ai.

### CREATE TUPLE

`CREATE TUPLE` statement defines schema.

Schemas are used to process the following tasks.

- Checking the valition of input tuples.
- Conversion from JSONTuple into GungnirTuple.
- Specifying the input by `FROM statements`.

The following is the syntax of `CREATE TUPLE` statement.

    CREATE TUPLE schema_name
        (field_name [field_type], ...)
        [PARTITIONED BY parition_field, ...]

We specify the followings.
* the Tuple name in schema_name.
* field name in field_name.
* field type in field_type. if users do not specify the type, the type is automatically detected.
* field name for partition key in parition_field. The field is used to split Tuples into patititions.
When `PARTITIONED BY clause` is not specified, Tuples are randomly partitioned.

> Example:

    CREATE TUPLE userAction1 (
        _tno,
        _tid,
        _time,
        field1 INT,
        field2 STRING,
        field3 STRING,
        field4 LIST<STRING>,
        field5 TINYINT,
        field6 STRUCT<member1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), member2 LIST<BIGINT>>,
        field7 MAP<STRING, BOOLEAN>,
        field8 BOOLEAN,
        field9 TIMESTAMP,
        field10 MAP<INT, STRING>
        )
        PARTITIONED BY _tno

#### Field Types

For field type, users can specify the the followings.

* Numeric Types

TINYINT
, SMALLINT
, INT
, BIGINT
, FLOAT
, DOUBLE

* Date/Time Types

TIMESTAMP

* String Types

STRING

* Misc Types

BOOLEAN

* Complex Types
LIST
, MAP
, STRUCT

#### TINYINT

In GungnirTuple, the value is treated as a Byte object in Java.

In JSONTuple, field value is described with digits.

#### SMALLINT

In GungnirTuple, the value is treated as Short object in Java.

In JSONTuple, the field value is described with digits.

#### INT

In GungnirTuple, the value is treated as Integer object in Java.

In JSONTuple, the field value is described with digits.
When a user specify a value in JSONTuple field and in addtion the field is not added in the schema,
the value is converted into INT（Integer）or BIGINT（Long） depending on the figure length.

#### BIGINT

In GungnirTuple, the value is treated as a Long object in Java.

In JSONTuple, field values are written in digits.

When a user specify a value in JSONTuple field and in addtion the field is not added in the schema,
the value is converted into INT（Integer）or BIGINT（Long） depending on the figure length.

#### FLOAT

In GungnirTuple, the value is treated as a Float object in Java.

In JSONTuple, field values are written in decimal.

#### DOUBLE

In GungnirTuple, the value is treated as a Double object in Java.

In JSONTuplem, field values are written in decimal.

When a user specify a decimal value in JSONTuple field, and in addtion the field is not added in the schema,
the value is converted into DOUBLE (Double).

#### TIMESTAMP

* TIMESTAMP

In GungnirTuple, the value is treated as java.util.Date in Java.

In JSONTuplem, field values are written in epoch time.

> Example:
    field:1382086720

* TIMESTAMP (date_format)

In date_format, users specify date format.
genn.ai adopt the format decribed in [java.text.SimpleDateFormat](
http://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html).

In GungnirTuple, the values are treated as ava.util.Date.

> Example:


    field TIMESTAMP('yyyy-MM-dd HH:mm:ss')

In JSONTuple, user write the values with specified format with date_format.

> Example:

    field:"2013-10-18 18:07:25"

#### STRING

In GungnirTuple, the values are treated as a String object in Java.

In JSONTuple, The value are described as a string surrounded with double quotation marks.

#### BOOLEAN

In GungnirTuple, the values is a Boolean object in Java.

IN JSONTuple, the value are written as true or false.

#### LIST

In GungnirTuple, the value is treated as java.util.List object in Java.
Numeric Types、TIMESTAMP、STRING、BOOLEAN  can be a element type of lists.

> Example:

    field LIST<STRING>

In JSONTuple, the values are described as a Array.

> Example:

    field:["tokyo","kyoto","osaka"]

#### MAP

GungnirTupleでは、Javaのjava.util.Mapとして扱われます。
MAPのキーと値の型には、Numeric Types、TIMESTAMP、STRING、BOOLEAN のいずれかを指定します。

> Example:

        field MAP<STRING, INT>
       
In JSONTuple, the element of values are written as Objects.

> Example:
    field:{"078-8220":1, "061-3601":2, "127-0001":3}

When user add a field with Numeric Types key, the value should be surrounded with double quotation marks.

> Example:

    field MAP<INT, DOUBLE> --> field:{"1":0.6, "200":2.3, "560":3.0}

#### STRUCT

In GungnirTuple, the value are treated as ai.genn.gungnir.tuple.Struct object.


        STRUCT<field_name [field_type], ...>


> Example:
        field STRUCT<member0 STRING, member1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), member2 LIST<BIGINT>>


In JSONTuple, the values of fields are described as Objects.

> Example:
        field:{member0:"gennai", member1:"2013-10-18 18:28:34", member2:[10000, 20000, 30000]}

#### JSONTuple example

The following is userAction1 described in the previous section in JSONTuple.

        {
          field1:12345,
          field2:"xxx",
          field3:"yyy",
          field4:["aaa","bbb","ccc"],
          field5:1,
          field6:{
            member1:"2013-10-18 18:28:34",
            member2:[10000, 20000, 30000]
          },
          field7:{"A":true, "B":false, "C":false},
          field8:true,
          field9:1382086720,
          field10:{"11":"lll", "22":"mmm"}
        }

#### Special fields

Users can define the following special fields in schema.

* &#x5f;tid
* &#x5f;tno
* &#x5f;time

These names are reserved and therefore users should not use them.

* &#x5f;tid

tid is a field to store Tracking ID.
Adding &#x5f;tid field into a schema, Tracking IDs are inserted in Tuples (the type is STRING).

* &#x5f;tno

This field stores Tracking No.
Adding &#x5f;tno to a schema, The value of Tracking No (INT) is inserted.

* &#x5f;time

time is a field to store the time when the Tuple was stored.
Adding the &#x5f;time field into a schema, stored time (TIMESTAMP) is inserted into the Tuple.

#### Tracking ID と Tracking No

Tracking ID is unique id (24-length string）to identify the source of Tuple.
Tracking No is the value which is numeric converted from Tracking No.

Tracking ID and Tracking No are automatically inserted when the user add &#x5f;tid and &#x5f;tno fields in the JSONTuple.
Generated  Tracking ID and Tracking No are inserted into a Tuple and returned with the Set-Cookie header (only Tracking ID is returned) to the source.
The source of post submits the given Tracing ID with the Cookie header in the next JSONTuple submit.
The Tracing ID in the submit Cookie header shows the JSONTuples were submit from the same source.
Searching with submit Tracing ID get Tracing No and then &#x5f;tid and &#x5f;tno are inserted into the Tuple.
Although Both Tracking ID and Tracking No can be used to detect the the Tuple identity, users would
think that Tracking No is easier to identify the Tuples.

---

### SHOW TUPLES

This statement shows the list of defined Tuple schemas.

        SHOW TUPLES

The results are shown in JSON as follows.

> Result:

       [
         {"name":"userAction1","owner":"user@genn.ai","createTime":"2013-10-18T02:14:00.241Z"},
         {"name":"userAction2","owner":"user@genn.ai","createTime":"2013-10-17T02:16:34.898Z"}
       ]

* name field shows Tuple name.
* owner field shows the user who careate the Tuple.
* createTime field shows the creation time of the Tuple.

---

### DESC TUPLE

This statement shows the information of specified Tuple schema.

        DESC TUPLE schema_name


* We add a Tuple name into schema_name int the above statement.

> Example:
        DESC TUPLE userAction1


genn.ai gives the output in JSON format.

> Result:

      {
        "name":"userAction2","fields":{"field1":{"type":"BIGINT"},"field2":
        {"type":"STRING"},"field3":{"type":"STRING"}},"partitioned":       
        ["field1"],"owner":"user@genn.ai","createTime":"2013-09-13T01:35:55.667Z"
      }

In the above JSON, the followings is the meanings of each block.
* name field shows Tuple name.
* fields contains the list of fields in the Tuple.
* owner field shows the user who careate the Tuple.
* partitioned shows partition key field of the Tuple.
* createTime field shows the creation time of the Tuple.


---

### DROP TUPLE

This statement remove the specified Tuple schema.

         DROP TUPLE schema_name

* We add Tuple name into schema_name.

> Example:

        DROP TUPLE userAction1


---

### CREATE VIEW

This statement defines a view of specified Tuple. User can define a Tuple schema with another name.


        CREATE VIEW view_schema_name AS FROM tuple_schema_name FILTER condition;

We specify the followings.
* the name of View in view_schema_name.
* the original Tuple name into tuple_schema_name.
* the condition to connect Tuple to View condition into condition.

> Example:


        CREATE VIEW viewAction1 AS FROM userAction1 FILTER field3 = 'CATEGORY-1'
        CREATE VIEW viewAction2 AS FROM userAction1 FILTER field3 = 'CATEGORY-2'
        CREATE VIEW viewAction3 AS FROM userAction1 FILTER field3 = 'CATEGORY-3'


> The above examples define three views on the basis of Tuple schema, userAction1 depending of the value of field3.

---

### SHOW VIEWS

This statement shows the list of defined Views.

        SHOW VIEWS


genn.ai outputs the resuls in JSON format.

> Result:
> 
    [
      {"name":"viewAction1","owner":"user@genn.ai","createTime":"2013-10-19T03:19:22.241Z"},        
      {"name":"viewAction2","owner":"user@genn.ai","createTime":"2013-10-19T03:19:56.898Z"},        
      {"name":"viewAction3","owner":"user@genn.ai","createTime":"2013-10-19T03:19:34.898Z"}
    ]


* name field shows the View name.
* owner field shows the user who create the view.
* createTime field shows the creation time of the View.

---

### DESC VIEW

This statement shows the infomation of specified View.

        DESC VIEW schema_name

* We add View name into schema_name.

> Example:
        DESC VIEW viewAction1


genn.ai shows the results in JSON format.

> Result:
> 
    {
        "name":"viewAction1","from":"userAction1","filter":"field3 = CATEGORY-1","owner":"user@genn.ai","createTime":"2013-10-19T03:19:22.241Z"
    }

* name field shows the View name.
* from field shows the original Tuple of the View.
* filter fields shows the condition to connect Tuple and View.
* owner field shows the user who create the view.
* createTime field shows the creation time of the View.

---

### DROP VIEW

This statement deletes the specified View.

        DROP VIEW schema_name

* We add View name into schema_name.

> Example:

        DROP VIEW viewAction1

