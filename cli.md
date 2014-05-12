---
layout: manual
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

This command is used to exit from Gungnir CLI.

    gungnir> quit;

---

#### EXPLAIN

Explain command shows the excution plan of topology.

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

We can see how operators in exution plan are applied to input tuples.
A operation plan represents a graph whose vertices are operators, and edges between vertices are streams.
In the above example, four operators (SPOUT_0, PARTITION_1, FILTER_2, FILTER_3) are conneted with streams.
The stream from each operator is shown in the next line of the operator.
For example, we can see that the stream derived from SPOUT_0 is onely one and it coneect to PARTITION_1 and PARTITION_1
has two streams and they connect to FILTER_2 and FILTER_3 respectively.

---

#### SUBMIT TOPOLOGY

This command registries a topology and launch it.

    gungnir> SUBMIT TOPOLOGY;

Since the launch process of a topology is execution asynchronouslly, users should confirm that the state
of the topology is "RUNNING" by the "DESC TOPOLOGY" command.

> Example:

    gungnir> FROM userAction AS ua1 USING kafka_spout() FILTER field1 = 10 INTO s1;
    OK
    gungnir> EXPLAIN;
    …
    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir> DESC TOPOLOGY;  <-- confirm the state
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"RUNNING", ...}

After the execution of "SUBMIT TOPOLOGY" command, above example confirms the status of the toplogy with the "DESC TOPOLOGY" command.

---

#### DESC TOPOLOGY

This command shows the regeistrated topology information.

    gungnir> DESC TOPOLOGY;

The result is shown in JSON format.

> Example:
    gungnir> DESC TOPOLOGY;
    {"id":"5261606ee4b099995d4f460f","explain":"SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])\n ...","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z","summary":{"name":"gungnir_5261606ee4b099995d4f460f","status":"ACTIVE","uptimeSecs":403,"numWorkers":1,"numExecutors":3,"numTasks":3}}

The followings are the meanings of blocks in the above result.
* "id" is Topology ID.
* "explain" is execution plan of Topology.
* "status" represents the state of topology.
"status" takes the one of the four values (STARTING, RUNNING, STOPPING, STOPPED) and shows the status of topology.
* "owner" represents the user name who register the Topology.
* "createTime" shows the registration time of topology.
* "summary" is the status information of the  Topology. This block is shown only when "status" is RUNNING.

When we specify the block name, we get the value of specified block.

    gungnir> DESC TOPOLOGY topology_id;

> Example:

    gungnir> DESC TOPOLOGY 5261606ee4b099995d4f460f;
    {"id":"5261606ee4b099995d4f460f", ...}

---

#### SHOW TOPOLOGIES

This command shows the list of registered topology.

    gungnir> SHOW TOPOLOGIES;

The resuls are shown in JSON format.

> Example:

    gungnir> SHOW TOPOLOGIES;
    [{"id":"5261606ee4b099995d4f460f","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z"}]

The followins are the meanings of blocks in the above result.
* "id"  shows topology id.
* "status" shows the status of topology.
"status" takes the one of the four values (STARTING, RUNNING, STOPPING, STOPPED) and shows the status of topology.
* "owner" represents the user name who register the Topology.
* "createTime" shows the registration time of topology.

---

#### STOP TOPOLOGY

This command stops the specified topology.

    gungnir> STOP TOPOLOGY topology_id;

* Users specify the id of topology in topology_id.

The stop process of the specified topology is execution asynchronouslly. Therefore after the stop command, users should confirm that the state
of the topology is "STOPPEDG" by the "DESC TOPOLOGY" command.

> Example:
    gungnir> STOP TOPOLOGY 5261606ee4b099995d4f460f;
    OK
    gungnir> DESC TOPOLOGY;  <-- confirm the state
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"STOPPED", ...}

---

#### START TOPOLOGY

This command restarts stopped topology.

    gungnir> START TOPOLOGY topology_id;

* Users specify the id of topology in topology_id.

The start process of the specified topology is execution asynchronouslly. Therefore after the start command, users should confirm that the state
of the topology is "RUNNING" by the "DESC TOPOLOGY" command.

> Example:
    gungnir> START TOPOLOGY 5261606ee4b099995d4f460f;
    OK
    gungnir> DESC TOPOLOGY;  <-- confirm status
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"RUNNING", ...}

---

#### DROP TOPOLOGY

This command deletes registrated topology.

    gungnir> DROP TOPOLOGY topology_id;

* Users specify the id of topology in topology_id.

To delete a topology, the topology need to be stopped. Before delete a toplogy, users should confirm taht the status of topology is "STOPPED" with "DESC TOPOLOGY" command.

> Example:

    gungnir> DESC TOPOLOGY;  <-- 停止しているかを確認
    {"id":"5261606ee4b099995d4f460f","explain":...","status":"STOPPED", ...}
    gungnir> DROP TOPOLOGY 5261606ee4b099995d4f460f;
    OK

---

#### CLEAR

This command removes query from memory.
Queries (FROM ...) exist on memory until the topology is submitted.
When you want to remove the query in order to submit another query, users submit CLEAR command.

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

This command is used to set a property of topology.
The submitted propertes are reflected with "SUBMIT TOPOLOGY" or "START TOPOLOGY" command.
The reflected properties cleared with "STOP TOPOLOGY", therefore users should set the propeties again before "START TOPOLOGY".
The following is the form of SET command.

    gungnir> SET property_name = property_value;

> Example:
    gungnir> SET monitor = enable;

> The above command the a monitor property to flush Monitor log.

---

#### TRACK

This command submits a JOINTuple to validate the topology.

    gungnir> TRACK tuple_name json_tuple; 

We specify the followings.
* tuple name in tuple_name
* JSONTuple in join_tuple.

When a Set-Cookie header is returned in the response, the contents are stored in the Cookie of CLI memory.
The stored Cookie is submitted as the Cookie header of next TRACK command.

> Example:
    gungnir> TRACK userAction {field1:10,field2:"test"}; 

* Interactive Mode

The TRACK command has interactive mode in which uerss can edit JSONTuple interactively.
When a user sumit TRACK commadn without json_tuple, CLI reqest the user to input a JSONTuple to submit.
After editing JSONTuple, the user submit edited JSONTuple.

    gungnir> TRACK userAction;
    field1 (INT): 12345
    field2 (STRING): test
    TRACK userAction {"field1":12345,"field2":"test"}
    OK

The following is a example of a JSONTuple with LIST, MAP, STRUCT fields.

    CREATE TUPLE userAction2 (f1 STRING, f2 LIST<INT>, f3 MAP<STRING, INT>, f4 STRUCT<m1 TIMESTAMP('yyyy-MM-dd HH:mm:ss'), m2 BOOLEAN>);

> Example:
    gungnir> TRACK userAction2;
    f1 (STRING): test
    f2 (LIST)
      0 (INT): 1
      1 (INT): 2
      2 (INT): 3
      3 (INT):   <-- push [Enter key] and finish editing LIST (f2)
    f3 (MAP)
      key (STRING): k1
      value (INT): v1
      value (INT): 1
      key (STRING): k2
      value (INT): 2
      key (STRING): k3
      value (INT): 3
      key (STRING):   <-- push [Enter key] and finish editing MAP (f3)
    f4 (STRUCT)
      m1 (TIMESTAMP(yyyy-MM-dd HH:mm:ss)): 2013-10-19 22:02:24
      m2 (BOOLEAN): false

    TRACK userAction2 {"f1":"test","f2":[1,2,3],"f3":{"k1":1,"k2":2,"k3":3},"f4":{"m1":"2013-10-19 22:02:24","m2":false}}
    OK

---

#### COOKIE

COOKIE commands show the contents of on memory Cookie.
The on memory Cookie is removed when CLI is finished.

    gungnir> COOKIE;

> Exmaple:
    gungnir> COOKIE;
    {"name":"TID","value":"52558684e4b0f241c72d0365","domain":null,"path":null,"comment":null,"commentUrl":null,"discard":false,"ports":[],"maxAge":864000000,"version":1,"secure":false,"httpOnly":false}]

---

#### COOKIE CLEAR

This command is for remove on memory Cookies.
Since, on memory Cookie stores the traking ids, users can get another tracking id from the server after removing on memory Cookies.

    gungnir> COOKIE CLEAR;

> Example:

    gungnir> COOKIE CLEAR;
    OK
    gungnir> COOKIE;
    []

---

#### MONITOR

With MONITOR command, users know how the input Tuple is processed in the topology. Specifically
the Monitor log on the input tuple is flushed with this command.

    gungnir> MONITOR topology_id;

* Users specify Topology ID.

We use Monitor with the follwoing steps.

- Set the proerty to flush Monitor log and sebumit the topology.
- Specify the Topology id in the first step, run MONITOR command.
- Open another CLI, and then submit JOINTuple with TRACK command. JOINTuple can be submitted with the curl command.
- Check the CLI running MONITOR command, and see the Monitor log for the submitted tuple in the third step.

> Example:

    gungnir> SET monitor = enable;
    OK
    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir> DESC TOPOLOGY;
    {"id":"5261606ee4b099995d4f460f","explain":"SPOUT_0(kafka_spout(), kafka_spout(), [userAction(field1 INT, field2 STRING)])\n ...","status":"RUNNING","owner":"user@genn.ai","createTime":"2013-10-18T16:23:09.901Z","summary":{"name":"gungnir_5261606ee4b099995d4f460f","status":"ACTIVE","uptimeSecs":403,"numWorkers":1,"numExecutors":3,"numTasks":3}}
    gungnir> MONITOR 5261606ee4b099995d4f460f;

Open another CLI and run TRACK command.

    gungnir> track userAction {"field1":1234,"field2":"test"};

Monitor log is flushed in CLI.

    gungnir> MONITOR 5261606ee4b099995d4f460f;
    {"time":"2013-10-19T14:12:36.856Z","source":"SPOUT_0","target":"PARTITION_1","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.880Z","source":"PARTITION_1","target":"FILTER_2","tuple":{"tupleName":"userAction","values":[1234,"test"]}}
    {"time":"2013-10-19T14:12:36.953Z","source":"PARTITION_1","target":"FILTER_3","tuple":{"tupleName":"userAction","values":[1234,"test"]}}

We can see the tuple flows from source operator into target operator. The content of tuple is shown in the above JSON log.
