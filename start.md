##                                       Single Node Tutorial

  完成单节点安装后，可以直接通过web notebook或者gui连接到该节点上执行操作。单节点安装参考:xxx

### 1. 创建数据库
  ```
  db = database("C:/DolphinDB")
  ```
  
  * 若目录C:/DolphinDB不存在，则创建该文件目录，并且作为该数据库ID
  * 若目录C:/DolphinDB存在，且只包含 `DolphinDB` 类型的表及相关文件，则会打开该数据库
  * 若目录C:/DolphinDB存在，但包含的不是 `DolphinDB` 类型的表及相关文件，则抛出异常

### 2. 删除数据库
  ```
  dropDatabase("C:/DolphinDB")
  ```
   * 以数据库的ID,即创建数据库时传入的路径作为参数

### 3. 创建内存表并保存到数据库

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


---

### 4. 从磁盘上读取表到内存中

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
  
### 5. 删除数据库中的表

  ```
  db = database("C:/DolphinDB")
  dropTable(db, "tname"); 
  ```
  * dropTable删除数据库中的表，第一个参数为db handle; 第二个参数为表名(字符串）

  
### 6. 从磁盘上加载csv文件到dolphindb数据库
   
  假设有一个位于C盘的test.csv文件，我们需要把它加载到内存中。
  ```
  t = loadText(C:/test.csv)
  ```
  * loadTest把csv文件转换为dolphindb的内存表，默认列以逗号(,)分割
  * t为内存表

### 7. 表的增删改查 
  
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

    * 使用 `append!` 函数可直接将 `table_b` 的全部内容插入到 `table_a` 中
    * 使用 `SQL` 中的聚集函数 `select count(*)` 查看前后 `table_a` 的大小

    ```
    table_a = loadTable(db, "t1")
    table_b = loadTable(db, "t1")
    select count(*) from table_a
    select count(*) from table_b
    table_a.append!(table_b)
    select count(*) from table_a
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

### 8. 表的增删改查 

