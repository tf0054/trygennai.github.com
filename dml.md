---
layout: manual
title: genn.ai
---

## LanguageManual DML

This page is a language manual of DML.

### FROM

FROM statement load a Tuple from input source.

#### When the input sources are external.

    FROM schema_name AS schema_alias, ... USING spout_processor

Users specify the followings.
* Tuple or View name in schema_name.
* Tuple or View name used in queries in schema_alias.
* Processor to load input in spout_processor.
 
> Example:
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout()

One topology does not have more than one exter input source.

#### Spout Processor

kafka_spout

> From Kafka, load Tuples. This is a default configuraiton of genn.ai.
>
    kafka_spout()

#### When input sources are internal (stream)

    FROM stream_name[(schema_alias, ...)], ...


* specify a stream name in stream_name.

The following is a example loading all the Tuples from a stream.

> Example:
    FROM s1, s2

The following is a example loading only the specified Tuples from a stream.

> Example:
    FROM s1(ua1, ua2), s2(v1)

---

### INTO

INTO statement is used for branching and merging a stream.

    INTO stream_name

* specify output stream name in stream_name.

> Example:
    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1

User can load a stream output by INTO with FROM statements.

#### Branching

    FROM userAction1 AS ua1, userAction2 AS ua2, view1 AS v1 USING kafka_spout() INTO s1;
    FROM s1(ua1) ...
    FROM s1(ua2) ...
    FROM s1(v1) ...

#### Merging

    FROM s1(ua1) ... INTO s2;
    FROM s1(ua2) ... INTO s3;
    FROM s1(v1) ... INTO s4;
    FROM s2, s3, s4 ...

---

### JOIN

JOIN adds external data into a filed of a Tuple.

    JOIN join_name ON join_condition TO join_fields USING fetch_processor
    
    join_condition:
    join_name.key_field = field [AND join_name.key_field = field AND join_name.key_field <> 0]
    
    join_fields:
    join_name.join_field AS field_alias, ...

Users specify the following arguments.
* A name to join in join_name. The name is used in join_condition or join_fields to identify the fields of external data.
* Condition in join_condition. Specify the key field (Tuple) of join data to make the data have unique id.
When conditions are complex, we use AND. We can also specify the condition with constants to the key field of join data.
* To join_fields, we specify fileds to join. Specifically we add the list of filed names and field name to be joined AS tuple. The field is added the specified Tuple.

> Example:
    JOIN j1 ON j1.code1 = field1 AND j1.code2 = field2 AND j1.del = 0
      TO j1.name AS field10, j1.type AS field11
      USING mongo_fetch('db1', 'col1')

---

### FILTER

Fileter statement judges whether the input tuple is flashed as the output or not.

    FILTER condition

* In condition, we add the condition for the filter.

 We can use the following signs in conditions.

* &#61; or &#61;&#61;
* <> or !&#61;
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

#### &#61;, &#61;&#61;, <>, !&#61;, >, >&#61;, <, <&#61;

> Example:
    field1 >= 10

#### LIKE

In LIKE statement, we can use "%" (multiple character) and "_"(single character).

> Example:
    field2 LIKE 't%'

#### REGEXP

In REGEXP, we can use the regex pattern described in java/util/regex/Pattern.
http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html

> Example:

    field3 REGEXP '^[A-Z]{2}-[0-9]{4}$'

#### IN, ALL

IN checks whether input LIST fields contains the at least one specified values.

> Example:

    field4 IN ('tokyo', 'kyoto', 'osaka')

ALL checks whether input LIST fields contains all the specified values.

> Example:

    field4 ALL ('tokyo', 'kyoto', 'osaka')

#### BETWEEN

> Example:

    field1 10 AND 100


#### AND, OR, NOT

AND, OR, NOT can be nested.  The priority is NOT, AND, OR.
We can customize the priority using parenthesis.

> Example:
    field1 <= 30 AND (field5 BETWEEN 10 AND 100 OR field2 LIKE 'A%')

#### Comparison between STRUCT types

When the field of Tuple is STRUCT type, gen.ai compare the field as follows.

> Example:
    field6.member3 = 100

#### Comparison between LIST types

When the Tuples are LIST type,  genn.ai checks the field values as follows. 

> Example:
    field4[0] = 'tokyo'

#### Comparison between MAP types

When the field of Tuple is MAP type, genn.ai checks the field as follows.

> Example:
    field7['visa'] = true

#### Constant

Genn.ai support the following constants.

* String
list of characters quoted with single or double quotation marks.

* INT
    number only

>
> Example:
    2147483647


* DOUBLE
    number.number

>
> Example:
    12.5


* BIGINT
    numberL

>
> Example:
    9223372036854775807L


* SMALLINT
    numberS

>
> Example:
    32767S


* TINYINT
    numberY

>
> Example:
    255Y


* FLOAT
    number.numberF

>
> Example:
    12.5F


* BOOLEAN値
    true|false


---

### FILTER GROUP

FILTER GROUP validates whether multiple input Tuples are flashed as the output or not.

    FILTER GROUP EXPIRE period [STATE TO state_field]
      condition, ...


* In period, we specify the keeping time range of filter.
* In state_field, we specify the output field to flush filter status. When input pass the filter,
genn.ai adds the status field into tuple with specified field name. when users do not specify STATE TO clause, the status field is not added.
* In condition, we specify fileter conditions. We can specify the multiple filters sepratated with commas for multple Tuples.
When all the conditions are satisfied, input Tuple passes the filter and also, the filter status is initialized.

> Example:
    FILTER GROUP EXPIRE 7DAYS STATE TO fg_state
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member2 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'


 Above example executes the filter to three Tuples, ua1, ua2, ua3. When the input Tuple satisfy both Condition 1 and 2 (\*), the input Tuple pass the filter.
 (\*) Condition 1 is a filter to ua1 and Condition 2 is a fliter to ua2 and ua3.

 Since Condition 2 specify the two conditions, ua2 and ua3 with "OR", Tuple should satisfiy the ua1 and ua2 or ua1 and ua3.

 Above example declares that the status keeps 7 days when condition 1 or condition 2 is satisfied.
 For example, when condition 1 is satisfied and then conditon 2 is satisfied in 7 days, the conditons 1 and 2 are satisfied.
 When the condition 2 is satisfied at 8 days later, the status of conditon 1 is initialize and therefore the satus is that only condtion 2 is satisfied.

 In the abave example, genn.ai add the filter status into "fg_state" field of the Tuple, since "fg_state" is specified in state_field.
 fg_state stores the list of times when condition 1 and condition 2 are satisfied, therefore the size of list is the same as the number of conditions.

#### period

The way to specify the period has the variations.

* Specify with second
    number(SECONDS|SEC)
>
> Example:
    30SECONDS


* Specify with minutes
    number(MINUTES|MIN)
>
> Example:
    55MIN
  
* Specify with hours
    number(HOURS|H)
>
> Example:
    55HOURS


* Specify with days
    number(DAYS|D)

> Example:
    15DAYS

---

### EACH

EACH executes the editing or aggregation of Tuples.

  EACH expr, ...

* In expr, users specify a aggregation function, edit function or a field accessor.

#### Aggregation function

* Count the number of Tuples.
>
> Example:
    EACH count() AS cnt1


* Compute the summation of input field values.
>
> Example:
    EACH sum(field1) AS sum1

* Comput the mean of input field values.
>
> Example:
    EACH avg(field1) AS avg1

#### Edit function

* When the field is NULL, the value is overried with specified value.
>
> Example:
    EACH ifnull(field1, 0) AS field1

* This function concatenates the values of STRING type fields.
>
> Example:
    EACH concat(field1, '-', field2) AS new_field


#### Arugments of functions

As for the constants, we will describe them after the number of supported functions is increased.

#### Accessor of fields

> Example:
    EACH field1, field6.member1 AS field10, field7['visa'] AS visa

The above example just outputs field1, stores field6.member1 into the field10 field, and extracts field7&#91;'visa'] into visa field.

---

### GROUP

GROUP executes the queries srrounded with BEGIN GROUP ... END GROUP as a group.

    BEGIN GROUP BY field, ...
      [END GROUP | TO STREAM]

* In field, we specify the field name for the group.

#### execute EACH as a group

> Example:
    BEGIN GROUP BY user_name
    EACH user_name, count() AS gc1
    EMIT * USING mongo_persist('db1', 'col2', 'user_name');
    END GROUP


The above example counts the number of tuples for each user.

#### Execute FILTER GROUP as a group

> Example:
    BEGIN GROUP BY user_name
    FILTER GROUP EXPIRE 1DAYS
      ua1.field1 >= 10 AND ua1.field8 = true,
      (ua2.field1 <= 30 AND ua2.field2.member3 BETWEEN 2 AND 7) OR ua3.field5 LIKE 'A%'
    EMIT * USING mongo_persist('db1', 'col3')
    END GROUP


Above example executes filter for each user. Specifically, the example checks whether the both condition 1
and condition 2 are satisfied with each user and then the FILTER GROUP status of each user is stored.

#### Nesting of GROUP

GROUP can be nested.

> Example:
    EACH ...  <- executes without creating group
    BEGIN GROUP BY date
      EACH ...  <- executes for each date
      BEGIN GROUP BY area
        EACH ...  <- execute for date + area
      END GROUP 
    END GROUP
    EACH ...  <- executes without creating group


TO STREAM free all the groups.

> Example:
    EACH ...  <- executes without creating group
    BEGIN GROUP BY date
      EACH ...  <- executes for each date
      BEGIN GROUP BY area
        EACH ...  <- execute for date + area
    TO STREAM
    EACH ...  <- executes without creating group


END GROUP and TO STREAM can be omitted when goups may not be free.

---

### EMIT

EMIT flush Tuples

  EMIT output_field, ... USING emit_processor

* Users add a list of output field in output_field. Wild cards can be specified (&#42;).
* In emit_processor, users specify a processor to use flush output.

> Example:
    EMIT field1, field2, field3 USING mongo_persist('db1', 'col1')

#### Emit Processor

* Kafka Emit Processor  

Emit Processor flush Tuple into Kafka.

    kafka_emit(topic_name)

 * Users specify the Topic name in topic_name. topic_name represents a processor variable.

> Example:

    kafka_emit('topic1')

* Mongo Persist Processor

Mong Persist Procssor flush Tuples inpo MongoDB.

    mongo_persist(db_name, collection_name [, key_names])

* In db_name, users specify the output DB name. db_name corresponds a processor variable.
* In collection_name, users specify output Collection name. collection_name correspond a processor variable.
* In key_names users specify output key field name. When key is compounded, specify with the array.
When key_names is  specified, output is updated to the key.
When key_names is not specified, output become instert.

> Example:
    mongo_persist('db1', 'col1')  <- insert
    mongo_persist('db1', 'col1', 'field2') <- update with field2 as the key
    mongo_persist('db1', 'col1', ['field2', 'field3']) <- update with field2 + field3 as the key


#### Processor variable

For output name of Emit Processor, the following processor variables can be contained.

${TOPOLOGY_ID} is replaced with the id of running Topology.
${ACCOUNT_ID} is replaced wiht the id of user account who launched the running Topology.

> Example:
    kafka_emit('topic_${TOPOLOGY_ID}')

