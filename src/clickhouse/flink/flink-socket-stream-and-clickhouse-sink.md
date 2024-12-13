# Flink Socket Stream and ClickHouse Sink

# Setting Up a Flink Job with ClickHouse Sink

<!-- toc -->

In this blog post, we'll walk through the setup of a Flink job that utilizes a socket stream as the source and sinks the data into ClickHouse. This guide is designed to be straightforward, making it easy for you to follow along.

## Introduction

Flink is a powerful stream processing framework, and ClickHouse is a fast open-source columnar database management system. Together, they can handle real-time data processing efficiently.

## Flink Job Overview

The code for our Flink job is simple and easy to understand.

First, we create a `StreamExecutionEnvironment` and set up a socket stream that listens on port `7777`.

Next, we initialize global parameters and configure ClickHouse settings, including host, username, and password. Finally, we add a ClickHouse sink to the source using the `addSink` method.

Here's the complete Java code for the Flink job:

```java
public static void main(String[] args) throws Exception {
    // Create execution environment
    final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // Set up checkpointing and state backend
    env.setStateBackend(new FsStateBackend("file:///tmp/ckp"));
    env.enableCheckpointing(10000, CheckpointingMode.EXACTLY_ONCE);

    // Initialize global parameters
    Map<String, String> globalParameters = new HashMap<>();
    globalParameters.put(ClickHouseClusterSettings.CLICKHOUSE_USER, "myuser");
    globalParameters.put(ClickHouseClusterSettings.CLICKHOUSE_PASSWORD, "mypassword");
    globalParameters.put(ClickHouseClusterSettings.CLICKHOUSE_HOSTS, "<http://127.0.0.1:8123/>");
    globalParameters.put(TIMEOUT_SEC, String.valueOf(TIMEOUT_SEC));
    globalParameters.put(ClickHouseSinkConst.IGNORING_CLICKHOUSE_SENDING_EXCEPTION_ENABLED, "true");

    // ClickHouse cluster properties
    // sink common
    globalParameters.put(ClickHouseSinkConst.TIMEOUT_SEC, "1");
    // globalParameters.put(ClickHouseSinkConst.FAILED_RECORDS_PATH, "/tmp/clickhouse-failed-records");
    globalParameters.put(ClickHouseSinkConst.NUM_WRITERS, "2");
    globalParameters.put(ClickHouseSinkConst.NUM_RETRIES, "2");
    globalParameters.put(ClickHouseSinkConst.QUEUE_MAX_CAPACITY, "2");
    globalParameters.put(ClickHouseSinkConst.IGNORING_CLICKHOUSE_SENDING_EXCEPTION_ENABLED, "false");

    // Set global parameters
    ParameterTool gParameters = ParameterTool.fromMap(globalParameters);
    env.getConfig().setGlobalJobParameters(gParameters);

    // Create socket stream
    DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

    // Process the input stream
    SingleOutputStreamOperator<String> dataStream = inputStream.map(new MapFunction<String, String>() {
        @Override
        public String map(String data) throws Exception {
            String[] split = data.split(",");
            User user = User.of(Integer.parseInt(split[0]), split[1], Integer.parseInt(split[2]));
            return User.convertToCsv(user);
        }
    });

    // Set up ClickHouse sink
    Properties props = new Properties();
    props.put(ClickHouseSinkConst.TARGET_TABLE_NAME, "default.user");
    props.put(ClickHouseSinkConst.MAX_BUFFER_SIZE, "10000");
    ClickHouseSink sink = new ClickHouseSink(props);

    // Add sink to the data stream
    dataStream.addSink(sink);
    dataStream.print();
}
```

## Building the Flink Job

To build the JAR file for our Flink job, simply run the following Maven command:

```
mvn package
```

## Deploying the Job

After setting everything up, you can submit the Flink job to your local cluster. However, you might encounter an error during execution:

```java
2022-12-01 14:20:22,595 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph       [] - Source: Socket Stream -> Map -> (Sink: Unnamed, Sink: Print to Std. Out) (1/1) (63f5fb779bef06be84104994f2834919) switched from INITIALIZING to FAILED on localhost:44615-8508ab @ localhost (dataPort=45279).
java.lang.NullPointerException: null
        at com.google.common.base.Preconditions.checkNotNull(Preconditions.java:787) ~[?:?]
        at ru.ivi.opensource.flinkclickhousesink.model.ClickHouseSinkCommonParams.<init>(ClickHouseSinkCommonParams.java:33) ~[?:?]
        at ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseSinkManager.<init>(ClickHouseSinkManager.java:24) ~[?:?]
        at ru.ivi.opensource.flinkclickhousesink.ClickHouseSink.open(ClickHouseSink.java:39) ~[?:?]
        at org.apache.flink.api.common.functions.util.FunctionUtils.openFunction(FunctionUtils.java:34) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.open(AbstractUdfStreamOperator.java:100) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.api.operators.StreamSink.open(StreamSink.java:46) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.runtime.tasks.RegularOperatorChain.initializeStateAndOpenOperators(RegularOperatorChain.java:107) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreGates(StreamTask.java:700) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$SynchronizedStreamTaskActionExecutor.call(StreamTaskActionExecutor.java:100) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreInternal(StreamTask.java:676) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.streaming.runtime.tasks.StreamTask.restore(StreamTask.java:643) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.runtime.taskmanager.Task.runWithSystemExitMonitoring(Task.java:948) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.runtime.taskmanager.Task.restoreAndInvoke(Task.java:917) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:741) ~[flink-dist-1.15.1.jar:1.15.1]
        at org.apache.flink.runtime.taskmanager.Task.run(Task.java:563) ~[flink-dist-1.15.1.jar:1.15.1]
        at java.lang.Thread.run(Thread.java:829) ~[?:?]
```

Oops, `NPE`! What's going wrong with this simple example?

After digging into the source code of [flink-clickhouse-sink](https://github.com/ivi-ru/flink-clickhouse-sink), particularly these files in error stack: `ClickHouseSink.java` `ClickHouseSinkManager.java` `ClickHouseSinkCommonParams.java`.

I noticed the error occurred because one of the global parameters was not set.

```java
public ClickHouseSinkCommonParams(Map<String, String> params) {
    Preconditions.checkNotNull(params.get(IGNORING_CLICKHOUSE_SENDING_EXCEPTION_ENABLED),
            "Parameter " + IGNORING_CLICKHOUSE_SENDING_EXCEPTION_ENABLED + " must be initialized");

    this.clickHouseClusterSettings = new ClickHouseClusterSettings(params);
    this.numWriters = Integer.parseInt(params.get(NUM_WRITERS));
    this.queueMaxCapacity = Integer.parseInt(params.get(QUEUE_MAX_CAPACITY));
    this.maxRetries = Integer.parseInt(params.get(NUM_RETRIES));
    this.timeout = Integer.parseInt(params.get(TIMEOUT_SEC));
    this.failedRecordsPath = params.get(FAILED_RECORDS_PATH);
    this.ignoringClickHouseSendingExceptionEnabled = Boolean.parseBoolean(params.get(IGNORING_CLICKHOUSE_SENDING_EXCEPTION_ENABLED));

    Preconditions.checkNotNull(failedRecordsPath); // 🙋🙋🙋🙋🙋🙋🙋🙋🙋 error because of this not null check
    Preconditions.checkArgument(queueMaxCapacity > 0);
    Preconditions.checkArgument(numWriters > 0);
    Preconditions.checkArgument(timeout > 0);
    Preconditions.checkArgument(maxRetries > 0);
}
```

After commenting out the line with `FAILED_RECORDS_PATH` and setting the correct path, as follows:

```java
globalParameters.put(ClickHouseSinkConst.FAILED_RECORDS_PATH, "/tmp/clickhouse-failed-records");
```

I rebuilt the jar file using the `mvn clean package` command, and submitted the job again. Finally, the job transitioned to the `RUNNING` state.

```java
==> log/flink-dudu-taskexecutor-2-dudumac15.local.log <==
2022-12-01 14:31:46,521 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseWriter [] - Building components
2022-12-01 14:31:46,525 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseWriter$WriterTask [] - Start writer task, id = 0
2022-12-01 14:31:46,525 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseWriter$WriterTask [] - Start writer task, id = 1
2022-12-01 14:31:46,526 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseSinkScheduledCheckerAndCleaner [] - Build Sink scheduled checker, timeout (sec) = 1
2022-12-01 14:31:46,526 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseSinkManager [] - Build sink writer's manager. params = ClickHouseSinkCommonParams{clickHouseClusterSettings=ClickHouseClusterSettings{hostsWithPorts=[<http://127.0.0.1:8123/>], credentials='dXNlcjE6dG9wc2VjcmV0', authorizationRequired=true, currentHostId=0}, failedRecordsPath='/tmp/clickhouse-failed-records', numWriters=2, queueMaxCapacity=2, ignoringClickHouseSendingExceptionEnabled=false, timeout=1, maxRetries=2}
2022-12-01 14:31:46,527 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseSinkBuffer [] - Instance ClickHouse Sink, target table = default.user, buffer size = 10000
2022-12-01 14:31:46,529 INFO  org.apache.flink.runtime.taskmanager.Task                    [] - Source: Socket Stream -> Map -> (Sink: Unnamed, Sink: Print to Std. Out) (1/1)#0 (1a44fb4fea1b183589b7cb60dd4cbf1e) switched from INITIALIZING to RUNNING.

==> log/flink-dudu-standalonesession-2-dudumac15.local.log <==
2022-12-01 14:31:46,532 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph       [] - Source: Socket Stream -> Map -> (Sink: Unnamed, Sink: Print to Std. Out) (1/1) (1a44fb4fea1b183589b7cb60dd4cbf1e) switched from INITIALIZING to RUNNING.

==> log/flink-dudu-taskexecutor-2-dudumac15.local.log <==
2022-12-01 14:31:46,532 INFO  org.apache.flink.streaming.api.functions.source.SocketTextStreamFunction [] - Connecting to server socket localhost:7777

==> log/flink-dudu-standalonesession-2-dudumac15.local.log <==
2022-12-01 14:31:49,304 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 1 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1669876309294 for job 9f280a1432d17e6cadbf92841fb89a1a.
2022-12-01 14:31:49,348 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 1 for job 9f280a1432d17e6cadbf92841fb89a1a (0 bytes, checkpointDuration=51 ms, finalizationTime=3 ms).

==> log/flink-dudu-taskexecutor-2-dudumac15.local.out <==
(1, 'dudu', 2 )

==> log/flink-dudu-taskexecutor-2-dudumac15.local.log <==
2022-12-01 14:31:57,582 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseWriter$WriterTask [] - Ready to load data to default.user, size = 1
2022-12-01 14:31:57,773 INFO  ru.ivi.opensource.flinkclickhousesink.applied.ClickHouseWriter$WriterTask [] - Successful send data to ClickHouse, batch size = 1, target table = default.user, current attempt = 0
```

The Flink cluster log displays all processes. After it reaches the `RUNNING` state, it connects to socket port `7777` on localhost and triggers checkpoints periodically.

Notice that we have set Flink to read data from a socket. To accept user input, we need to run the following `nc` command.

```bash
nc -lk 7777
```

After entering some input, such as `1,tom,20`, you can verify that the data has been successfully written to the ClickHouse table.

Connect to clickHouse and select all records from the `user` table:

```sql
SELECT *
FROM user

Query id: b94d136f-20dc-40bf-97d5-8b7243453b57

┌─id─┬─name─┬─age─┐
│  1 │ tom  │  20 │
└────┴──────┴─────┘
```

Great! Flink successfully reads the data from the socket stream and sinks it to the Clickhouse sink.
