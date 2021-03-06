---
title: Troubleshoot TiCDC
summary: Learn how to troubleshoot issues you might encounter when you use TiCDC.
aliases: ['/docs/dev/ticdc/troubleshoot-ticdc/']
---

# Troubleshoot TiCDC

This document introduces the common issues and errors that you might encounter when using TiCDC, and the corresponding maintenance and troubleshooting methods.

## How do I choose `start-ts` when creating a task in TiCDC?

The `start-ts` of a replication task corresponds to a Timestamp Oracle (TSO) in the upstream TiDB cluster. TiCDC requests data from this TSO in a replication task. Therefore, the `start-ts` of the replication task must meet the following requirements:

- The value of `start-ts` is larger than the `tikv_gc_safe_point` value of the current TiDB cluster. Otherwise, an error occurs when you create a task.
- Before starting a task, ensure that the downstream has all data before `start-ts`. For scenarios such as replicating data to message queues, if the data consistency between upstream and downstream is not required, you can relax this requirement according to your application need.

If you do not specify `start-ts`, or specify `start-ts` as `0`, when a replication task is started, TiCDC gets a current TSO and starts the task from this TSO.

## Why can't some tables be replicated when I create a task in TiCDC?

When you execute `cdc cli changefeed create` to create a replication task, TiCDC checks whether the upstream tables meet the [replication restrictions](/ticdc/ticdc-overview.md#restrictions). If some tables do not meet the restrictions, `some tables are not eligible to replicate` is returned with a list of ineligible tables. You can choose `Y` or `y` to continue creating the task, and all updates on these tables are automatically ignored during the replication. If you choose an input other than `Y` or `y`, the replication task is not created.

## How do I handle replication interruption?

A replication task might be interrupted in the following known scenarios:

- The downstream continues to be abnormal, and TiCDC still fails after many retries.

    - In this scenario, TiCDC saves the task information. Because TiCDC has set the service GC safepoint in PD, the data after the task checkpoint is not cleaned by TiKV GC within the valid period of `gc-ttl`.
    - Handling method: You can resume the replication task via the HTTP interface after the downstream is back to normal.

- Replication cannot continue because of incompatible SQL statement(s) in the downstream.

    - In this scenario, TiCDC saves the task information. Because TiCDC has set the service GC safepoint in PD, the data after the task checkpoint is not cleaned by TiKV GC within the valid period of `gc-ttl`.
    - Handling procedures:
        1. Query the status information of the replication task using the `cdc cli changefeed query` command and record the value of `checkpoint-ts`.
        2. Use the new task configuration file and add the `ignore-txn-start-ts` parameter to skip the transaction corresponding to the specified `start-ts`.
        3. Stop the old replication task via HTTP API. Execute `cdc cli changefeed create` to create a new task and specify the new task configuration file. Specify `checkpoint-ts` recorded in step 1 as the `start-ts` and start a new task to resume the replication.

## How do I know whether a TiCDC replication task is interrupted?

- Check the `changefeed checkpoint` monitoring metric of the replication task (choose the right `changefeed id`) in the Grafana dashboard. If the metric value stays unchanged, or the `checkpoint lag` metric keeps increasing, the replication task might be interrupted.
- Check the `exit error count` monitoring metric. If the metric value is greater than `0`, an error has occurred in the replication task.
- Execute `cdc cli changefeed list` and `cdc cli changefeed query` to check the status of the replication task. `stopped` means the task has stopped and the `error` item provides the detailed error information. After the error occurs, you can search `error on running processor` in the TiCDC server log to see the error stack for troubleshooting.
- In some extreme cases, the TiCDC service is restarted. You can search the `FATAL` level log in the TiCDC server log for troubleshooting.

## What is `gc-ttl` in TiCDC?

Since v4.0.0-rc.1, PD supports external services in setting the service-level GC safepoint. Any service can register and update its GC safepoint. PD ensures that the key-value data smaller than this GC safepoint is not cleaned by GC. Enabling this feature in TiCDC ensures that the data to be consumed by TiCDC is retained in TiKV without being cleaned by GC when the replication task is unavailable or interrupted.

When starting the TiCDC server, you can specify the Time To Live (TTL) duration of GC safepoint through `gc-ttl`, which means the longest time that data is retained within the GC safepoint. This value is set by TiCDC in PD, which is 86,400 seconds by default.

## How do I handle the OOM that occurs after TiCDC is restarted after a task interruption?

If the replication task is interrupted for a long time and a large volume of new data has been written to TiDB, Out of Memory (OOM) might occur when TiCDC is restarted. In this situation, you can enable unified sorter, TiCDC's experimental sorting engine. This engine sorts data in the disk when the memory is insufficient. To enable this feature, pass `--sort-engine=unified` and `--sort-dir=/path/to/sort_dir` to the `cdc cli` command when creating a replication task. For example:

{{< copyable "shell-regular" >}}

```shell
cdc cli changefeed update -c [changefeed-id] --sort-engine="unified" --sort-dir="/data/cdc/sort"
```

> **Note:**
>
> + Since v4.0.9, TiCDC supports the unified sorter engine.
> + TiCDC (the 4.0 version) does not support dynamically modifying the sorting engine yet. Make sure that the changefeed has stopped before modifying the sorter settings.
> + Currently, the unified sorter is an experimental feature. When the number of tables is too large (>=100), the unified sorter might cause performance issues and affect replication throughput. Therefore, it is not recommended to use it in a production environment. Before you enable the unified sorter, make sure that the machine of each TiCDC node has enough disk capacity. If the total size of unprocessed data changes might exceed 1 TB, it is not recommend to use TiCDC for replication.

## What is the complete behavior of TiCDC garbage collection (GC) safepoint?

If a replication task starts after the TiCDC service starts, the TiCDC owner updates the PD service GC safepoint with the smallest value of `checkpoint-ts` among all replication tasks. The service GC safepoint ensures that TiCDC does not delete data generated at that time and after that time. If the replication task is interrupted, the `checkpoint-ts` of this task does not change and PD's corresponding service GC safepoint is not updated either. The Time-To-Live (TTL) that TiCDC sets for a service GC safepoint is 24 hours, which means that the GC mechanism does not delete any data if the TiCDC service can be recovered within 24 hours after it is interrupted.

## How do I handle the `Error 1298: Unknown or incorrect time zone: 'UTC'` error when creating the replication task or replicating data to MySQL?

This error is returned when the downstream MySQL does not load the time zone. You can load the time zone by running [`mysql_tzinfo_to_sql`](https://dev.mysql.com/doc/refman/8.0/en/mysql-tzinfo-to-sql.html). After loading the time zone, you can create tasks and replicate data normally.

{{< copyable "shell-regular" >}}

```shell
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql -p
```

```
Enter password:
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.
```

If you use MySQL in a special public cloud environment, such Alibaba Cloud RDS, and if you do not have the permission to modify MySQL, you need to specify the time zone using the `--tz` parameter:

1. Query the time zone used by MySQL:

    {{< copyable "sql" >}}

    ```sql
    show variables like '%time_zone%';
    ```

    ```
    +------------------+--------+
    | Variable_name    | Value  |
    +------------------+--------+
    | system_time_zone | CST    |
    | time_zone        | SYSTEM |
    +------------------+--------+
    ```

2. Specify the time zone when you create the replication task and create the TiCDC service:

    {{< copyable "shell-regular" >}}

    ```shell
    cdc cli changefeed create --sink-uri="mysql://root@127.0.0.1:3306/" --tz=Asia/Shanghai
    ```

    > **Note:**
    >
    > In MySQL, CST refers to the China Standard Time (UTC+08:00). Usually you cannot use `CST` directly in your system, but use `Asia/Shanghai` instead.

Be cautious when you set the time zone of the TiCDC server, because the time zone will be used for the conversion of time type. It is recommended that you use the same time zone in the upstream and downstream databases, and specify the time zone using `--tz` when you start the TiCDC server.

The TiCDC server chooses its time zone in the following priority:

1. TiCDC first uses the time zone specified by `--tz`.
2. When `--tz` is not available, TiCDC tries to read the time zone set by the `TZ` environment variable.
3. When the `TZ` environment variable is not available, TiCDC uses the default time zone of the machine.

## What is the default behavior of TiCDC if I create a replication task without specifying the configuration file in `--config`?

If you use the `cdc cli changefeed create` command without specifying the `-config` parameter, TiCDC creates the replication task in the following default behaviors:

* Replicates all tables except system tables
* Disables the Old Value feature
* Skips replicating tables that do not contain [valid indexes](/ticdc/ticdc-overview.md#restrictions)

## How do I handle the incompatibility issue of configuration files caused by TiCDC upgrade?

Refer to [Notes for compatibility](/ticdc/manage-ticdc.md#notes-for-compatibility).

## Does TiCDC support outputting data changes in the Canal format?

Yes. To enable Canal output, specify the protocol as `canal` in the `--sink-uri` parameter. For example:

{{< copyable "shell-regular" >}}

```shell
cdc cli changefeed create --pd=http://10.0.10.25:2379 --sink-uri="kafka://127.0.0.1:9092/cdc-test?kafka-version=2.4.0&protocol=canal" --config changefeed.toml
```

> **Note:**
>
> * This feature is introduced in TiCDC 4.0.2.
> * TiCDC currently supports outputting data changes in the Canal format only to Kafka.

For more information, refer to [Create a replication task](/ticdc/manage-ticdc.md#create-a-replication-task).

## How do I view the status of TiCDC replication tasks?

To view the status of TiCDC replication tasks, use `cdc cli`. For example:

{{< copyable "shell-regular" >}}

```shell
cdc cli changefeed list --pd=http://10.0.10.25:2379
```

The expected output is as follows:

```json
[{
    "id": "4e24dde6-53c1-40b6-badf-63620e4940dc",
    "summary": {
      "state": "normal",
      "tso": 417886179132964865,
      "checkpoint": "2020-07-07 16:07:44.881",
      "error": null
    }
}]
```

* `checkpoint`: TiCDC has replicated all data before this timestamp to downstream.
* `state`: The state of the replication task:

    * `normal`: The task runs normally.
    * `stopped`: The task is stopped manually or encounters an error.
    * `removed`: The task is removed.

> **Note:**
>
> This feature is introduced in TiCDC 4.0.3.

## How do I know whether the replication task is stopped manually?

You can know whether the replication task is stopped manually by using `cdc cli`. For example:

{{< copyable "shell-regular" >}}

```shell
cdc cli changefeed query --pd=http://10.0.10.25:2379 --changefeed-id 28c43ffc-2316-4f4f-a70b-d1a7c59ba79f
```

In the output of this command, `admin-job-type` shows the state of the replication task:

* `0`: In progress, which means that the task is not stopped manually.
* `1`: Paused. When the task is paused, all replicated `processor`s exit. The configuration and the replication status of the task are retained, so you can resume the task from `checkpiont-ts`.
* `2`: Resumed. The replication task resumes from `checkpoint-ts`.
* `3`: Removed. When the task is removed, all replicated `processor`s are ended, and the configuration information of the replication task is cleared up. Only the replication status is retained for later queries.

## Why does the latency from TiCDC to Kafka become higher and higher?

* Check [how do I view the status of TiCDC replication tasks](#how-do-i-view-the-status-of-ticdc-replication-tasks).
* Adjust the following parameters of Kafka:

    * Increase the `message.max.bytes` value in `server.properties` to `1073741824` (1 GB).
    * Increase the `replica.fetch.max.bytes` value in `server.properties` to `1073741824` (1 GB).
    * Increase the `fetch.message.max.bytes` value in `consumer.properties` to make it larger than the `message.max.bytes` value.

## When TiCDC replicates data to Kafka, does it write all the changes in a transaction into one message? If not, on what basis does it divide the changes?

No. According to the different distribution strategies configured, TiCDC divides the changes on different bases, including `default`, `row id`, `table`, and `ts`.

For more information, refer to [Replication task configuration file](/ticdc/manage-ticdc.md#task-configuration-file).

## When TiCDC replicates data to Kafka, can I control the maximum size of a single message in TiDB?

No. Currently TiCDC sets the maximum size of batch messages to 512 MB, and that of a single message to 4 MB.

## When TiCDC replicates data to Kafka, does a message contain multiple types of data changes?

Yes. A single message might contain multiple `update`s or `delete`s, and `update` and `delete` might co-exist.

## When TiCDC replicates data to Kafka, how do I view the timestamp, table name, and schema name in the output of TiCDC Open Protocol?

The information is included in the key of Kafka messages. For example:

```json
{
    "ts":<TS>,
    "scm":<Schema Name>,
    "tbl":<Table Name>,
    "t":1
}
```

For more information, refer to [TiCDC Open Protocol event format](/ticdc/ticdc-open-protocol.md#event-format).

## When TiCDC replicates data to Kafka, how do I know the timestamp of the data changes in a message?

You can get the unix timestamp by moving `ts` in the key of the Kafka message by 18 bits to the right.

## How does TiCDC Open Protocol represent `null`?

In TiCDC Open Protocol, the type code `6` represents `null`.

| Type | Code | Output Example | Note |
|:--|:--|:--|:--|
| Null | 6 | `{"t":6,"v":null}` | |

For more information, refer to [TiCDC Open Protocol column type code](/ticdc/ticdc-open-protocol.md#column-type-code).

## The `start-ts` timestamp of the TiCDC task is quite different from the current time. During the execution of this task, replication is interrupted and an error `[CDC:ErrBufferReachLimit]` occurs

Since v4.0.9, you can try to enable the unified sorter feature in your replication task, or use the BR tool for an incremental backup and restore, and then start the TiCDC replication task from a new time.

## How can I tell if a Row Changed Event of TiCDC Open Protocol is an `INSERT` event or an `UPDATE` event?

If the Old Value feature is not enabled, you cannot tell whether a Row Changed Event of TiCDC Open Protocol is an `INSERT` event or an `UPDATE` event. If the feature is enabled, you can determine the event type by the fields it contains:

* `UPDATE` event contains both `"p"` and `"u"` fields
* `INSERT` event only contains the `"u"` field
* `DELETE` event only contains the `"d"` field

For more information, refer to [Open protocol Row Changed Event format](/ticdc/ticdc-open-protocol.md#row-changed-event).

## How much PD storage does TiCDC use?

TiCDC uses etcd in PD to store and regularly update the metadata. Because the time interval between the MVCC of etcd and PD‘s default compaction is one hour, the amount of PD storage that TiCDC uses is proportional to the amount of metadata versions generated within this hour. However, in v4.0.5, v4.0.6, and v4.0.7, TiCDC has a problem of frequent writing, so if there are 1000 tables created or scheduled in an hour, it then takes up all the etcd storage and returns the `etcdserver: mvcc: database space exceeded` error. You need to clean up the etcd storage after getting this error. See [etcd maintaince space-quota](https://etcd.io/docs/v3.4.0/op-guide/maintenance/#space-quota) for details. It is recommended to upgrade your cluster to v4.0.9 or later versions.
