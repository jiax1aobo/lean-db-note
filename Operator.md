
|算子|解释|说明|
|:-|:-|:-|
|indexscan|索引扫描|当表中创建了索引,并使用索引字段进行查询时,会进行索引扫描|
|tablescan|顺序表扫描|tablescan负责从磁盘中以连续块的形式从磁盘中读取数据页.一般在SQL查询中,有几张表就要有几个tablescan操作|
|project|投影|从表中根据查询字段选择相关的列|
|filter|过滤/谓词|filter会根据where条件中的筛选条件,筛选出符合的记录.其中过滤条件也叫谓词逻辑.project和filter算子在表的列和行两个唯独对数据进行限定,大大缩减数据量,降低资源消耗,是SQL优化时常用的方法|
|exchange|交换|在分布式数据库中,tablescan等操作是分布式进行的,将各结点结果汇总的过程就是exchange,其中localExchange无网络IO,remoteExchange有网络传输|
|join|连接|表的笛卡尔积|
|aggregation|聚合|对数据做分组聚合,统计分析|
|values|数值|有时SQL中数据不是从表中查询出来的,而是给定的数值,values操作就是将这些标识符转为数值,如`select 1+1, DATA '2001-08-22', ARRAY[1,2,3];`|
|scalar|标量|根据策略,给定一个结果值,如`case when`|



19,185.87

15,273

53,329.5
