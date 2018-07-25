## 单机 Tutorial

### 1. 下载安装

---

略

---

### 2. 启动

---

略

---

### 3. 基本功能实例

---

#### 3.1 创建数据库

* 数据库类型

  * __无分区的数据库__ - 适用于远小于本地服务器内存的表
  * __本地分布式数据库__ - 适用于与本地服务器内存大小基本相同的表
  * __在分布式文件系统上（DFS）的分布式数据库__ - 部署在多个服务器上，适用于远大于单个服务器内存的表

* 我们推荐使用 __分布式文件系统上的分布式数据库__

* 下面我们用一个例子建立一个 _无分区数据库_

  ```
  db = database("C:/DolphinDB")
  ```

  * 若文件夹不存在，则在该目录下创建文件夹并保存数据库文件
  * 若文件夹已存在，且只包含 `DolphinDB` 类型的表及相关文件，则会打开该数据库
  * 若文件夹已存在，但包含的不是 `DolphinDB` 类型的表及相关文件，则抛出异常

* 对于其余类型的数据库操作，请查阅文档 ...... ->Link

---

#### 3.2 创建表并保存

* 建立一个简单内存表

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
  saveTable(db, t1)
  ```

  * 使用 `saveTable`  函数将内存表 `t1`  保存到数据库 `db` 中 （默认以表名存入磁盘）
  * 在数据库路径下，生成了 `t1.tbl` 的表文件和 `t1` 的文件夹
  * 在 `t1` 文件夹下，生成了 `id.col`、`x.col`、`y.col` 三个列文件，分别存储表 `t1` 的三列

* 对于分布式表的创建和保存，请查阅文档 ... -> LINK

---

#### 3.3 从磁盘上读取表到内存中

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

* 查看表中的内容，与 _3.1_ 中的表 `t1` 相同

  ```
  t
  ```

* 使用 `typestr` 函数查看表的类型为 `IN-MEMORY TABLE` ，即内存表

  ```
  typestr(t)
  ```

* 对于分布式表的读取，请查阅文档 ... -> LINK

---

#### 3.4 表的增删改查

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
    * 了解更多内置函数，请参考文档 ----> LINK

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
  * 若您有数据持久化的需求，请在修改表后使用 `savaTable(db, t)` 函数

* 更多

  * 在 `DolphinDB` 中，我们对标准 `SQL` 进行了优化和扩充，更多用法请参考文档 --->LINK

---

