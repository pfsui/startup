## DolphinDB单节点基本操作入门教程
本文档适用于完成单节点安装后，通过web notebook或者gui连接到节点上，进行DolphinDB的基础操作。单节点安装请参考:https://2xdb.net/dolphindb/tutorials_cn/blob/master/standalone_server.md
### 1. 创建/删除数据库
#### 1.1 创建数据库
	  ``` 
	  db = database("C:/DolphinDB")
	  ``` 
  * 若目录C:/DolphinDB不存在，则自动创建该文件目录，并且作为该数据库ID，数据库成功创建并打开
  * 若目录C:/DolphinDB存在，且只包含`DolphinDB`类型的表及相关文件，则会打开该数据库
  * 若目录C:/DolphinDB存在，但包含的不是`DolphinDB`类型的表及相关文件，则数据库创建失败。需要清空`DophinDB`目录再次尝试。
#### 1.2. 删除数据库
	  ``` 
	  dropDatabase("C:/DolphinDB")
	  ``` 
   * 函数dropDatabase以数据库的ID作为参数
### 2. 创建/删除表
#### 2.1 创建表并保存到数据库
  下面介绍三种构建表的方法，a）table函数创建内存表；b）loadTable从database中加载表；c）loadText将磁盘上的csv文件转换为表
##### 2.1.1 用table函数创建内存表
* 创建一个简单内存表
	  ``` 
	  t1 = table(take(1..10, 100) as id, rand(10, 100) as x, rand(10.0, 100) as y)
	  ``` 
  * 使用 `table` 函数建立内存表，包含 `id`、`x`、`y` 三列，共 _100_ 行
  * __注意__ ： `t`  是一张 __内存表__ ，因此不会保存在磁盘上

* 查看表中的内容
	  ``` 
	  t1
	  ``` 
* 将内存表保存到数据库中
	  ``` 
	  db = database("C:/DolphinDB")
	  saveTable(db, t1)
	  ``` 
  * 使用 `saveTable`  函数将内存表 `t1`  保存到数据库 `db` 中 （默认以表名存入磁盘）
  * 在数据库路径下，生成了 `t1.tbl` 的表文件和 `t1` 的文件夹
  * 在 `t1` 文件夹下，生成了 `id.col`、`x.col`、`y.col` 三个列文件，分别存储表 `t1` 的三列
##### 2.1.2 loadTable从database中加载表
* 获取已存在的数据库的句柄
	  ``` 
	  db = database("C:/DolphinDB")
	  ``` 
* 从数据库中读取表名为 `t1` 的表
	  ```
	  t = loadTable(db, "t1")
	  ``` 
  * 或可使用 _newTable = loadTable(db, `t1)_
  * 在 `DolphinDB` 中，单引号加单词可表示字符串
* 查看表中的内容
	  ``` 
	  t
	  ``` 
* 使用 `typestr` 函数查看表的类型为 `IN-MEMORY TABLE` ，即内存表
	  ``` 
	  typestr(t)
	  ``` 
##### 2.1.3 loadText将磁盘上的csv文件转换为表
  假设有一个位于C盘的test.csv文件，我们需要把它加载到内存中。
	  ``` 
	  t = loadText(C:/test.csv)
	  ``` 
  * loadTest把csv文件转换为dolphindb的内存表，默认列以逗号(,)分割
  * t为内存表
#### 2.2 删除数据库中的表
	  ``` 
	  db = database("C:/DolphinDB")
	  dropTable(db, "tname"); 
	  ``` 
  * dropTable删除数据库中的表，第一个参数为db handle; 第二个参数为表名(字符串）
### 3. 内存表的增删改查 
  按照标准的sql语言操作
* 查询操作
  * 可直接使用表名
  * 可使用 `SQL` 中的 `select` 语句 - __DolphinDB 中的 SQL 只支持小写__
		``` 
		select * from t
		``` 
* 插入操作
  * 使用 `SQL` 中的 `insert`，并查看操作结果
		``` 
		insert into t values (5, 6, 2.5)
		t
		``` 
  * `DolphinDB` 是一款大数据系统，对批量插入有较好的支持
    * 使用 `append!` 函数可直接将 `tb` 的全部内容插入到 `ta` 中
    * 使用 `SQL` 中的聚集函数 `select count(*)` 查看前后 `ta` 的大小
		``` 
		ta = loadTable(db, "t1")
		tb = loadTable(db, "t1")
		select count(*) from ta
		select count(*) from tb
		ta.append!(tb)
		select count(*) from ta
		``` 
* 更新操作
  * 使用 `SQL` 中的 `update`，并查看操作结果
		``` 
		update t set y = 1000.1 where x = 5
		t
		``` 
* 删除操作
  * 使用 `SQL` 中的 `delete` ，并查看操作结果
		``` 
		delete from t where id=3
		t
		``` 
* 持久化
  * 由于我们操作的表 __位于内存__ 中，因此上述的表修改操作 __没有被记录到磁盘上__
  * 若需要对修改的表进行持久化，修改表后使用 `savaTable(db, t)` 函数

### 4. 更多高级内容
  * __独立服务器__ ,作为一个独立的工作站或服务器使用，无需配置。 详见教程:https://github.com/dolphindb/Tutorials_CN/blob/master/standalone_server.md
  * __单机集群搭建__ ,控制节点(controller)、代理节点（ agent）、数据节点(data node)部署在同一个物理机器上，详见教程:https://github.com/dolphindb/Tutorials_CN/blob/master/single_machine_cluster_deploy.md
  * __多机集群搭建__ ,在多个物理机器上部署 DolphinDB 集群。 详见教程:https://github.com/dolphindb/Tutorials_CN/blob/master/multi_machine_cluster_deploy.md
  * __内存数据库计算__ ,作为独立工作站使用，利用内存数据库的高性能，快速完成数据的加载，编辑和分析计算,详见教程：https://2xdb.net/dolphindb/tutorials_cn/blob/master/partitioned_in_memory_table.md
  * __分区数据库__ ，支持多种灵活的分区方式， 顺序分区，范围分区，值分区，列表分区，复合分区，详见教程:https://2xdb.net/dolphindb/tutorials_cn/blob/master/database.md
  * __脚本语言__ ，提供的脚本语言类似python + sql,易学，灵活，丰富的内置函数，详见教程：https://2xdb.net/dolphindb/tutorials_cn/blob/master/hybrid_programming_paradigms.md
  * __权限与安全配置__ ,提供了强大、灵活、安全的权限控制系统，以满足企业级各种应用场景，详见教程：https://2xdb.net/dolphindb/tutorials_cn/blob/master/ACL_and_Security.md
  