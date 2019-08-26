
# SQL的解析

1. 提交的SQL语句， PhoenixSQLLexer执行词法解析。注意这里的PhoenixSQLLexer是从src/antlr3/PhoenixSQL.g，经过antlr的翻译，生成的java代码。  

2. 根据PhoenixSQLParser的解析确定com.salesforce.phoenix.jdbc.PhoenixStatement.ExecutableStatement(Interface)的类型，目前有如下几类：
- 增删数据：ExecutableAddColumnStatement、ExecutableDropColumnStatement
- 创建/删除表格：ExecutableCreateTableStatement、ExecutableDropTableStatement
- Select操作：ExecutableSelectStatement
- 导入数据：ExecutableUpsertStatement
- 解释执行：ExecutableExplainStatement

3. 执行(2)中提供的实例化的 PhoenixStatement.ExecutableStatement提供executeQuery方法：
创建QueryCompiler。
执行compile过程。(识别limit、having、where、order、projector等操作，生成ScanPlan）
封装Scanner，并根据识别出的修饰词，对于结果进行修饰，整合出ResultIterator的各种功能的实现，具体在com.salesforce.phoenix.iterator包下。
该SQL对应的包装类为：OrderedAggregatingResultIterator.//它是如何组织数据，保证数据按照DESC或者ASC的方式展示？


对于创建表格的逻辑：  
1）解析SQL，翻译可执行的ExecutableCreateTableStatement，实例化MutationPlan。

2）创建MetaDataClient对象，将解析出的Statement转换成PTable的模型，更新SYSTEM.TABLE中的内容.（如果SYSTEM.TABLE不存在，还需要创建该表）

3）调用PhoenixConnection.addTable操作，这里会根据ConnectionQueryServicesImpl执行相关的服务。

4）加载Coprocessor。
```java
descriptor.addCoprocessor(ScanRegionObserver.class.getName(), phoenixJarPath, 1, null);
descriptor.addCoprocessor(UngroupedAggregateRegionObserver.class.getName(), phoenixJarPath, 1, null);
descriptor.addCoprocessor(GroupedAggregateRegionObserver.class.getName(), phoenixJarPath, 1, null);
descriptor.addCoprocessor(HashJoiningRegionObserver.class.getName(), phoenixJarPath, 1, null);
```
这里加载的Coprocessor有：  
- ScanRegionObserver: 封装RegionObserver.postScannerOpen接口，捕获出现的异常。即在scanner开启之后，做基本遍历，属于基础类实现。  
- UngroupedAggregateRegionObserver:
- GroupedAggregateRegionObserver:
- HashJoiningRegionObserver:
会在RegionCoprocessorHost的组织下，分别执行这四个类的doPostScanOpen操作，会根据QueryPlan以及Statement中包含的信息，进行功能筛选和组装，最终被返回的结果，是已经按照需求处理过的，从而实现类似于GroupBy、Sort等操作。
2）

Coprocessor机制 ：  
包括两部分，Observer和Endpoint  
Observer有RegionObserver、WALObserver、MasterObserver。用来实现固定执行点的”插桩”的功能，有点像关系型数据库当中的触发器的功能。
这里以RegionObserver的实现为例，介绍一下其中实现细节。  
1）为Table加载Observer接口的实现类。  
2）客户端调用某个操作的位置时，调用接口。例如，RegionObserver的postScannerOpen()会在执行scannerOpen之后执行。  
3）每一个Region设置一个RegionCoprocessorHost，负责管理加载到该Region的Coprocessor。  
4）每一个Region设置一个RegionCoprocesorEnvironment，封装在ObserverContext当中，作为执行Coprocessor的上下文环境。  
Endpoint不同于Observer，虽然它也是被加载到Region上，但是它的执行方式，是由Client端借助Table.coprocessorExec执行，是client到Regions的一次或者多次RPC操作，有时可能还需要在Client端对获取到的数据进行合并。可以查看一例：使用Coprocessor进行RowCount统计 http://www.binospace.com/index.php/make-your-hbase-better-2/  


# reference
1. 深入分析HBase-Phoenix执行机制与原理 https://www.cnblogs.com/leeeee/p/7276324.html