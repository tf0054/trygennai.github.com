---
layout: manual
title: genn.ai
---
# genn.ai Tutorial
This document is a quick start of genn.nai. Readers quickly go through the genn.ai environments.

## Requirements

This tutorial assumes the following environments.

**OS**
- Mac OSX 10.8.5
- CentOS 5.8 x86_64

**Java**

- Oracle JDK 1.6.0_43 x86\_64
 - set `JAVA_HOME`


## Prepare environment
In order to access genn.ai, you need not only build the environment described in the previous seciton, but also take the following steps.

１. Get GitHub account

To get a key from genn.ai, users need to get their Github account.  If you do not have Github account, get your account in the following page.

[https://github.com/](https://github.com/)  

２. Get access key

- To access genn.ai, you need to get the access key. To get the access key, access [http://dev.genn.ai/](http://dev.genn.ai/),
then click "Sign Up" then the acess key for you is shown.

![gennai-signup](img_tutorial/01_signup.png "Signup")

- Login with your GitHub account.

![gennai-signup](img_tutorial/02_GithubLogin.png "Signup")

- Approve the permission which is on genn.ai accessing GitHub.

- When you approve it, the browser shows genn.ai page and shows the published access key. We go through this genn.ai tutorial with the access key.

![gennai-signup](img_tutorial/03_getkey.png "Signup")
    

３. Down load client library file

Down load the genn.ai library file whose link exists in the genn.ai page.

４. Decompress client library

Decompress the downloaded file (gungnir-client.tar.gz) in a arbitary directory (following example decompresses the file in /usr/local).

After the decompression, Set environment variable, `PATH`.

        # cd /usr/local
        # tar xzvf ~/gungnir-client.tar.gz
        # export PATH=/usr/local/gungnir-client/bin:$PATH

The configuration file file, gungnir.yaml has the following content.

        # cat /usr/local/gungnir-client/conf/gungnir.yaml
        > ...
        > gungnir.thrift.server: "dev.genn.ai:9190"
        > gungnir.rest.server: "dev.genn.ai:9191"
        > ...

Now we finished building the genn.ai environment.

## Try genn.ai!
In this section, we try genn.ai with a example with minimum event processing.

### Connect genn.ai
To connect genn.ai, we use **gungnir** command in the extracted package in the previous section. Just specifying the user account
and access key to the command, we connect to the genn.ai's world!

In the following example, the command is given user account, **gennaitaro** and access key, **167668259f3e** which is given
in the previous section using Github service.

    $ gungnir -u gennaitaro -p 167668259f3e
    Nov 14, 2013 12:14:41 AM com.twitter.finagle.Init$ apply
    INFO: Finagle version 6.5.1 (rev=57de9b06e9d9456abaa98a5b02f085cc029cde41) built at 20130626-111057
    Gungnir server connected... dev.genn.ai/54.238.99.212:9190
    Welcome gennaitaro
    　
    gungnir> 

Finished the connection with genn.ai, the command shows the prompt gungir and wait the input. We call this command prompt as gungir console.

### Run command
Now we are in genn.ai world. Let's run some commands.
The following example runs a command, `EXPLAIN` to show execution plan of topology.

    gungnir> EXPLAIN;
    gungnir>

As we can see, there is no response, since we have not registered any process. Here it is ok except that we do not have any errors.

### Create schema
As the first step, let's create a schema. Here schema is a data structure (JSON) to be pushed to genn.ai. Before we push data to genn.ai,
we need to specify the JSON structure.

In the following exmaple, we create a schema, **userAction**. userAction has two STRING type fields (userId and hotelId).
To create a schema, we run `CREATE TOUPLE` command as follows.

    gungnir> CREATE TUPLE userAction (
    gungnir>   userId STRING,
    gungnir>   hotelId STRING
    gungnir> );
    OK
    gungnir>

Next we register another schema, **commitAction** which has four fields (two STRING type filelds, one TIMESTAMP type field, one INT type field).

    gungnir> CREATE TUPLE commitAction (
    gungnir>   userId STRING,
    gungnir>   hotelId STRING,
    gungnir>   checkin TIMESTAMP('yyyy-MM-dd HH:mm:ss'),
    gungnir>   nights INT
    gungnir> );
    OK
    gungnir>

Now we show the list of registered schemas.
To show the list of schema, we run the `SHOW TOUPLES` command.

    gungnir> SHOW TUPLES;
    [\
     {\
        "name":"userAction",\
        "owner":"gennaitaro",\
        "createTime":"2013-11-01T00:00:00.000Z"\
     },\
     {\
        "name":"commitAction",\
        "owner":"gennaitaro",\
        "createTime":"2013-11-01T00:00:00.000Z"\
     }\
    ]
    gungnir>

As we see, the two shemas (userAction and commitAction) are registered in genn.ai.

> NOTE: "\\" means lines are not breaked in the real console prompt.

Next, we check the detailed structure of registered schema.
To show the specified schema we run the `DESC TOUPLE` command.

    gungnir> DESC TUPLE userAction;
    {\
        "name":"userAction",\
        "fields":{\
            "userId":{"type":"STRING"},\
            "hotelId":{"type":"STRING"}\
        },\
        "partitioned":"-",\
        "owner":"gennaitaro",\
        "createTime":"2013-11-01T00:00:00.000Z"\
    }
    gungnir> DESC TUPLE commitAction;
    {\
        "name":"commitAction",\
        "fields":{\
            "userId":{"type":"STRING"},\
            "hotelId":{"type":"STRING"},\
            "checkin":{"type":"TIMESTAMP","pattern":"yyyy-MM-dd HH:mm:ss"},\
            "nights":{"type":"INT"}\
        },\
        "partitioned":"-",\
        "owner":"gennaitaro",\
        "createTime":"2013-11-01T00:00:00.000Z"\
    }
    gungnir>.

We can see that not only registered fields (such as name, userid), but alos other auto generated fields
such as owner, createTime, partitioned. The fields are described in [DDL reference](./ddl.html).

### Create topology

Toplogy is the internal logic of genn.ai, which defines input sources and procedures of input data.
Users create toplogy with a query language provided by genn.ai.

Now let's create a topology.

First we can specify input data soruce with `FROM ～ USING ～` sentence.

    FROM userAction AS ua USING kafka_spout()
    FILTER ua.userId REGEXP '^[A-Z]{2}[012][0-9]{7}$'
    EMIT userId, hotelId USING kafka_emit('${TOPOLOGY_ID}_user')

Currently genn.ai stores the input data into **Kafka** and therefore any toplogy load input data from Kafka.
As we see that the first sentence specify Kafka with `Using kafka_spout()`.

> NOTE: You may do not know Kafka. Do not worry. At this point of the tutorial, you just think Kafka as a JSON pool.

Sentence `FROM userAction AS ua` orders to interpret input JSON from Kafka with userAction schema.

    FROM userAction AS ua USING kafka_spout()

In the next sentence, the above example describes the specific input procedures.
The sentence describes that only the data with UserId field matching the pattern `^[A-Z]{2}[012][0-9]{7}$` are flushed.

In FILTER, we can use various conditions such as `==`, `!=`, `LIKE`.

    FILTER ua.userId REGEXP '^[A-Z]{2}[012][0-9]{7}$'

Last sentence, `EMIT ... USING ...` describes that output data are stored in Kafka.
Last sentence also add the **name** `${TOPOLOGY_ID}_user` into the output.

`${TOPOLOGY_ID}` is the id which is assigned when the topology is registered.
As for **name**, the value is used to get output.

    EMIT userId, hotelId USING kafka_emit('${TOPOLOGY_ID}_user')

Now, let's create a topology from gungnir console.

    gungnir> FROM userAction AS ua USING kafka_spout()
    gungnir> FILTER ua.userId REGEXP '^[A-Z]{2}[012][0-9]{7}$'
    gungnir> EMIT userId, hotelId USING kafka_emit('${TOPOLOGY_ID}_user')
    gungnir> ;
    OK
    gungnir>

Next, use `EXPLAIN` command to confirm the excution plan of created topology.

    gungnir> EXPLAIN;
    SPOUT_0(\
      kafka_spout(), [userAction(userId STRING, hotelId STRING)]\
    )\
     -S-> PARTITION_1\
    PARTITION_1(identity grouping)\
     -S-> FILTER_2\
    FILTER_2(userAction.userId REGEXP ^[A-Z]{2}[012][0-9]{7}$)\
     -S-> EMIT_3\
    EMIT_3(kafka_emit(${TOPOLOGY_ID}_user), [userId, hotelId])
    gungnir> 

Although we have create a topology, the created topology are not able to be
processed with genn.ai before the topology is not submitted to genn.ai.

Following command submit the created topology into genn.ai and then genn.ai prepares the
logic defined by the topology to process stream data.

    gungnir> SUBMIT TOPOLOGY;
    OK
    gungnir>

> COMPLEMENT:
> The topologies created with `FROM` are not registered in genn.ai. The topology is on session work memory in genn.ai. To make use of the topology, users need to submit it.

When the topology is successfuly registered, we confirm whether the status of topology become running as expected.

To check the status of topology, we run `DESC TOPOLOGY` command.

In the following example, we can see the status is **RUNNING** and the topology ready to be excuted.

    gungnir> DESC TOPOLOGY;
    { \
        "id":"5284a5b7e4b08627b67aecd3", \
        "explain":"SPOUT_0(\
            kafka_spout(), \
            [ \
                userAction( \
                    userId STRING, \
                    hotelId STRING \
                ), \
                commitAction( \
                    userId STRING, \
                    hotelId STRING, \
                    checkin TIMESTAMP(yyyy-MM-dd HH:mm:ss), nights INT \
                ) \
            ] \
            )￥n \
            -S-> PARTITION_1\nPARTITION_1(identity grouping)￥n \
            -S-> EMIT_2￥n \
            EMIT_2( \
               kafka_emit(${TOPOLOGY_ID}_user), \
               [userId, hotelId, name, image] \
        )", \
        "status":"RUNNING", \
        "owner":"gennaitaro", \
        "createTime":"2013-11-01T00:00:00.000Z", \
        "summary":{ \
            "name":"gungnir_5284a5b7e4b08627b67aecd3", \
            "status":"ACTIVE", \
            "uptimeSecs":37, \
            "numWorkers":1, \
            "numExecutors":3, \
            "numTasks":3 \
        } \
    }
    
    gungnir>


Note that **topology id** is needed to run command for manipulating the submitted topologies.
Users should have a memo on the value such as stopping / starting a topology.

> INFO:
> 
> - Stop and remove topology
>
>         gungnir> STOP TOPOLOGY 5284a5b7e4b08627b67aecd3;
>         OK
>         gungnir> DROP TOPOLOGY 5284a5b7e4b08627b67aecd3;
>         OK
>         gungnir>
>         
> 
> - Remove topology on session work memory
>
>         gungnir> CLEAR;
>         OK
>         gungnir> DESC TOPOLOGY;
>         FAILED: Topology is not registered
>         gungnir>
>

### Review what we have done

Now The topology submitted with `SUBMIT TOPLOGY` is ready to run. The followin is the image of whole system including the topology.

![all](img_tutorial/10_diagram.png "all")

When we put the data, the data is flowed into the topology and processed, and then the results are stored into Kafka.
In the later section, we will see how to show the stored data in Kafka.

### Submit Data

Users can submit data with the REST interface or gungnir console. We will try both.

First run `TRACK` command for debbuging, and try to submit data with **gungnir console**.

    gungnir> TRACK userAction {"userId":"AA01234567", "hotelId":"226979"};
    
    OK
    gungnir> 

When you can submit successfuly as the above example, next we will try to send the data with **REST interface**.
The fowwloing is the url of REST interface.

`http://dev.genn.ai:9191/gungnir/v1.0/track/user-id/schema-name`

In the above url, users change the **user-id** and **schema-name** following their environment.
We can confirm the **user id** with the `DESC USER` command.

    gungnir> DESC USER;
    {\
      "id":"5271d4c9e4b08627b67aeccd",\
      "name":"gennaitaro",\
      "password":"0A042lHaOOWdvEyYCK7r1piXIT6TYwUVD6V99RNitig=",\
      "createTime":"2013-11-01T00:00:00.000Z"\
    }
    gungnir> 

We can create the following url using the **user id** and the output **schema name** (in the above example, "userAction"). 

http://dev.genn.ai:9191/gungnir/v1.0/track/**5271d4c9e4b08627b67aeccd**/**userAction**

Let's submit data! We run the following curl command with another terminal.

> NOTE: please do not terminate the previous gungnir console.

    $ curl -v -H "Content-type: application/json" -X POST \
        -d '{"userId":"AA11234567", "hotelId":"226979"}' \
        http://dev.genn.ai:9191/gungnir/v1.0/track/5271d4c9e4b08627b67aeccd/userAction
    >
    >* About to connect() to dev.genn.ai port 9191
    >*   Trying 54.238.99.212... connected
    >* Connected to dev.genn.ai (54.238.99.212) port 9191
    ...(省略)...
    $ 
    
When the data was successfuly submitted, we can get the following HTTTP response.

    HTTP/1.1 200 OK
    Content-Length: 0
    Date: Fir, 01 Nov 2013 12:00:00 GMT

The execution of the sending data has two steps. genn.ai first load the JSON into Kafka, and then the each topology get the loaded data.

### Check the result

As described in the previous section, the output is stored in Kafka. To confirm the stored output data, we use **kafka-consumer.sh**
script which exists in bin directory as the gungnir command.
This script requests **output name**, which forms `${TOPOLOGY_ID}_user`.

The`${TOPOLOGY_ID}` in the output name is replaced with the topology id which is shown with `DESC TOPOLOGY;` command as a JSON block.
For instance, `${TOPOLOGY_ID}` is replaced with the user id, `5284a5b7e4b08627b67aecd3` in the follwoing example.

Now let's extract the results with the following command.

    $ kafka-consumer.sh 5284a5b7e4b08627b67aecd3_user
    2013-11-01 12:00:00,000 kafka.tools.SimpleConsumerShell$ \
                       INFO (Logging.scala:61) Starting consumer...

    > 2013-11-01 12:00:00,000 kafka.tools.SimpleConsumerShell$ \
    >    INFO (Logging.scala:61) consumed: {"userId":"AA01234567","hotelId":"226979"}
    > 2013-11-01 12:00:00,000 kafka.tools.SimpleConsumerShell$ \
    >    INFO (Logging.scala:61) consumed: {"userId":"AA11234567","hotelId":"226979"}
    > ...

We can see that the two data submitted in the section Submit Data are returned.
Note that, the kafka-consumer.sh script continue to poling data and not finished
all the data are flushed. When new data is sent to genn.nai, the results are shown in the console.

> NOTE: the command is finished with \[CTRL\]+\[C\].

Let's add new data with the following curl command. Please do not kill kafka-consumer.sh and open another terminal and run genn.ai console.

    $ curl -v -H "Content-type: application/json" -X POST \
        -d '{"userId":"AA21234567", "hotelId":"226979"}' \
        http://dev.genn.ai:9191/gungnir/v1.0/track/5271d4c9e4b08627b67aeccd/userAction


Then, we can see that the new data is added in kafka-consumer.sh console as follows.

    > 2013-11-19 22:30:19,317 kafka.tools.SimpleConsumerShell$ \
    >    INFO (Logging.scala:61) consumed: {"userId":"AA21234567","hotelId":"226979"}


In the last, we add a data with userId as "AA31234567". The output should not be flushed on the data in kafka-consume.sh console.

    $ curl -v -H "Content-type: application/json" -X POST \
        -d '{"userId":"AA31234567", "hotelId":"226979"}' \
        http://dev.genn.ai:9191/gungnir/v1.0/track/5271d4c9e4b08627b67aeccd/userAction

The above results show that the topology works as expected.

## End of tutorial

We have go throuh genn.ai and learn how easy to handle stream data with genn.ai.
For detailed usages, please check other doucments.
