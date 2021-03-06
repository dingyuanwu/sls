# 日志投递MaxCompute后，如何检查数据完整性 {#concept_bb2_3n1_hfb .concept}

在日志服务数据投递MaxCompute场景下，需要在MaxCompute表分区维度上检查数据完整性，即MaxCompute表中某个分区中数据是否已经完整。

## 使用保留字段`__partition_time__`作为表分区列，如何判断分区数据是否已完整 {#section_osm_kn1_hfb .section}

`__partition_time__`由日志的time字段计算得到，由日志**真实时间**按照时间格式字符串向下取整得出。其中，日志真实时间既不是投递数据的时间，也不是日志写入服务端时间。

举例：日志时间为2017-05-19 10:43:00，分区字段格式字符串配置为`yyyy_MM_dd_HH_mm`，每1小时投递一次。那么：无论该日志是什么时刻写入服务端，这条日志会存入MaxCompute的`2017_05_19_10_00`分区，计算细节参考[MaxCompute投递字段说明](../../../../cn.zh-CN/用户指南/数据投递/通过DataWorks投递数据到MaxCompute.md)。

如果不考虑写入了历史数据等问题，在日志实时写入的情况下，有以下两种方法判断分区数据是否已完整：

-   通过控制台或API/SDK判断（推荐）

    使用[API](../../../../cn.zh-CN/API 参考/日志库相关接口/GetShipperStatus.md)、[SDK](../../../../cn.zh-CN/SDK 参考/Java SDK.md)或者控制台获取指定Project/Logstore投递任务列表。例如API返回任务列表如下。控制台会对该返回结果进行可视化展示。

    ```
    {
      "count" : 10,
      "total" : 20,
      "statistics" : {
          "running" : 0,
          "success" : 20,
          "fail" : 0 
      }
      "tasks" : [
          ...
          {
              "id" : "abcdefghijk",
              "taskStatus" : "success",
              "taskMessage" : "",
              "taskCreateTime" : 1448925013,
              "taskLastDataReceiveTime" : 1448915013,
              "taskFinishTime" : 1448926013
          },
          {
              "id" : "xfegeagege",
              "taskStatus" : "success",
              "taskMessage" : "",
              "taskCreateTime" : 1448926813,
              "taskLastDataReceiveTime" : 1448930000,
              "taskFinishTime" : 1448936910
          }
      ]
    }
    ```

    `taskLastDataReceiveTime`表示该批任务中最后一条日志到达服务端时的机器系统时间，对应控制台的`接收日志数据时间`，根据该参数判断时间为T以前的数据是否已经全部投递到MaxCompute表。

    -   如果`taskLastDataReceiveTime` < `T + 300s`（300秒是为了容忍数据发送服务端发生错误重试）以前的每个投递任务状态都是success，说明T时刻的数据都已经入库。
    -   如果任务列表中有ready/running状态任务，说明数据还不完整，需要等待任务执行结束。
    -   如果任务列表中有failed状态任务，请查看原因并在解决后重试任务。您可以尝试修改投递配置，以解决投递问题。
-   通过MaxCompute分区粗略估计

    比如在MaxCompute中以半小时做一次分区，投递任务为每30分钟一次，当表中包含以下分区：

    ```
    2017_05_19_10_00
    2017_05_19_10_30
    ```

    当发现分区2017\_05\_19\_11\_00出现时，说明11:00之前分区数据已经完整。

    该方法不依赖API，**判断方式简单但结果并不精确**，仅用作粗略估计。


## 使用自定义日志字段作为表分区列，如何判断分区数据已完整 {#section_etm_kn1_hfb .section}

比如日志中有一个字段date，取值：20170518，20170519，在配置投递规则时将date列映射到表分区列。

这种情况下，需要考虑date字段与写入服务端时间差，结合[使用保留字段](#)方法，根据接收日志数据时间判断。

## 投递成功，但MaxCompute表数据有缺失，应如何解决 {#section_b1g_c41_hfb .section}

MaxCompute投递任务状态成功，但表中数据缺失，一般有以下原因：

-   表分区列映射的日志服务字段的名称不存在。此时投递过去的列值为null，而MaxCompute表不允许分区列值为null。
-   表分区列映射的日志服务字段的值包含`/`或其他特殊符号。MaxCompute将这些字符作为保留字，不允许在分区列中出现。

遇到这些情况时，投递策略为忽略异常的日志行，并继续任务，在该次任务中其它分区正确的日志行可以成功同步。

因此，在配置字段映射不当的情况下，可能出现任务成功但是表中缺少数据的情况，请修改分区列配置。建议使用保留字段`__partition_time__`作为分区。

更多细节请参考[MaxCompute投递相关限制说明](../../../../cn.zh-CN/用户指南/数据投递/通过DataWorks投递数据到MaxCompute.md)。

