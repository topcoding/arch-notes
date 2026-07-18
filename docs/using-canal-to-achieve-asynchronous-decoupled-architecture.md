本文介绍如何应用Canal实现异步、解耦的架构，后续有空再写文章分析Canal原理和源代码。

# Canal简介

Canal是用来获取数据库变更的中间件。
伪装自己为MySQL从库，拉取主库binlog并解析、处理。处理结果可发送给MQ，方便其他服务获取数据库变更消息，这一点非常有用。下面介绍一些典型用途。

![](/assets/images/https!img2020.cnblogs.com!blog!1247698!202111!1247698-20211126233517940-1393343247.png)

其中，Canal+MQ作为一个整体，从外界看来就是一个数据管道服务服务，如下图。
![](/assets/images/https!img2020.cnblogs.com!blog!1247698!202111!1247698-20211126010451933-1835138185.png)

# Canal典型用途

## 异构数据（如ES、HBase、不同路由key的DB）

通过Canal自带的adapter，同步异构数据至ES、HBase，而不用自行实现繁琐的数据转换、同步操作。这里的adapter就是典型的适配器模式，把数据转成相应格式，并写入异构的存储系统。

![](/assets/images/https!img2020.cnblogs.com!blog!1247698!202111!1247698-20211126234625791-113935225.png)

当然，也可以同步数据至DB，甚至构建一份按不同字段分片路由的数据库。
比如：下单时按用户id分库分表订单记录，然后借助Canal数据通道，构建一份按商家id分库分表的订单记录，用于B端业务（如商家查询自己接到哪些订单）。

![](/assets/images/https!img2020.cnblogs.com!blog!1247698!202111!1247698-20211126234423773-2066894541.png)

## 缓存刷新

缓存刷新的常规做法是，先更新DB，再删除缓存，再延迟删除（即cache-aside pattern+延迟双删），这种多步操作可能失败，而且实现相对复杂。借助Canal刷新缓存，使主服务、主流程无需关心缓存更新等一致性问题，保证最终一致性。
![](/assets/images/https!img2020.cnblogs.com!blog!1247698!202111!1247698-20211126010528362-909557615.png)

## 价格变化等重要业务消息

下游服务可立即感知价格变化。
常规做法是，先修改价格，再发出消息，此处的难点是要保证消息一定发送成功，以及如果发送不成功时如何处理。借助Canal，不用在业务层面担心消息丢失的问题。

## 数据库迁移

* 多机房数据同步
* 拆库
  虽然可以自己在代码中实现双写逻辑，然后对历史数据做处理，但是历史数据也可能被更新，需要不断迭代对比、更新，总之很复杂。

## 实时对账

常规做法是定时任务跑对账逻辑，时效性低，不能及时发现不一致问题。借助Canal，可实时触发对账逻辑。
大致流程如下：

* 接收数据变更消息
* 写入hbase作为流水记录
* 一段窗口时间过后，触发比较与对端数据做比较

# Canal客户端demo代码分析

以下示例是客户端连接Canal的例子，修改自官方github示例，楼主做了一些优化，并且在关键代码行中加入了注释。如果Canal把数据变更消息发送至MQ，写法有所不同，不同之处只是一个是订阅Canal，一个是订阅MQ，但是解析和处理逻辑基本相同。

```java
    public void process() {
        // 每批次处理的条数
        int batchSize = 1024;
        while (running) {
            try {
                // 连上Canal服务
                connector.connect();
                // 订阅数据（比如某个表）
                connector.subscribe("table_xxx");
                while (running) {
                    // 批量获取数据变更记录
                    Message message = connector.getWithoutAck(batchSize);
                    long batchId = message.getId();
                    int size = message.getEntries().size();
                    if (batchId == -1 || size == 0) {
                        // 非预期情况，需做异常处理
                    } else {
                        // 打印数据变更明细
                        printEntry(message.getEntries());
                    }

                    if (batchId != -1) {
                        // 使用batchId做ack操作：表明该批次处理完成，更新Canal侧消费进度
                        connector.ack(batchId);
                    }
                }
            } catch (Throwable e) {
                logger.error("process error!", e);
                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e1) {
                    // ignore
                }

                // 处理失败, 回滚进度
                connector.rollback();
            } finally {
                // 断开连接
                connector.disconnect();
            }
        }
    }

    private void printEntry(List<Entry> entrys) {
        for (Entry entry : entrys) {
            long executeTime = entry.getHeader().getExecuteTime();
            long delayTime = new Date().getTime() - executeTime;
            Date date = new Date(entry.getHeader().getExecuteTime());
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

            // 只关心数据变更的类型
            if (entry.getEntryType() == EntryType.ROWDATA) {
                RowChange rowChange = null;
                try {
                    // 解析数据变更对象
                    rowChange = RowChange.parseFrom(entry.getStoreValue());
                } catch (Exception e) {
                    throw new RuntimeException("parse event has an error , data:" + entry.toString(), e);
                }

                EventType eventType = rowChange.getEventType();

                logger.info(row_format,
                    new Object[] { entry.getHeader().getLogfileName(),
                            String.valueOf(entry.getHeader().getLogfileOffset()), entry.getHeader().getSchemaName(),
                            entry.getHeader().getTableName(), eventType,
                            String.valueOf(entry.getHeader().getExecuteTime()), simpleDateFormat.format(date),
                            entry.getHeader().getGtid(), String.valueOf(delayTime) });

                // 不关心查询，和DDL变更
                if (eventType == EventType.QUERY || rowChange.getIsDdl()) {
                    logger.info("ddl : " + rowChange.getIsDdl() + " ,  sql ----> " + rowChange.getSql() + SEP);
                    continue;
                }

                for (RowData rowData : rowChange.getRowDatasList()) {
                    if (eventType == EventType.DELETE) {
                        // 数据变更类型为 删除 时，打印变化前的列值
                        printColumn(rowData.getBeforeColumnsList());
                    } else if (eventType == EventType.INSERT) {
                        // 数据变更类型为 插入 时，打印变化后的列值
                        printColumn(rowData.getAfterColumnsList());
                    } else {
                        // 数据变更类型为 其他（即更新） 时，打印变化前后的列值
                        printColumn(rowData.getBeforeColumnsList());
                        printColumn(rowData.getAfterColumnsList());
                    }
                }
            }
        }
    }

    // 打印列值
    private void printColumn(List<Column> columns) {
        for (Column column : columns) {
            StringBuilder builder = new StringBuilder();
            try {
                if (StringUtils.containsIgnoreCase(column.getMysqlType(), "BLOB")
                    || StringUtils.containsIgnoreCase(column.getMysqlType(), "BINARY")) {
                    // get value bytes
                    builder.append(column.getName() + " : "
                                   + new String(column.getValue().getBytes("ISO-8859-1"), "UTF-8"));
                } else {
                    builder.append(column.getName() + " : " + column.getValue());
                }
            } catch (UnsupportedEncodingException e) {
            }
            builder.append("    type=" + column.getMysqlType());
            if (column.getUpdated()) {
                builder.append("    update=" + column.getUpdated());
            }
            builder.append(SEP);
            logger.info(builder.toString());
        }
    }
```
