##### 加载表
```
t =loadText("/home/hduser1/Yanting/数据/taqsmall.csv");
```

##### 显示表的基本信息
```
schema(t)		//column name, typeString, typeInt */
size(t)			//row number
columns(t)		//column number
```

##### 使用SQL语句查进行增删改查
```
select top 5 * from t
select top 5 Symbol, Date from t

update t  set OFR = OFR / 2
update t set symbol = 'B' where symbol == 'A'
insert into t values(['A'], [2007.08.01], [07:21:45], [37.03], [19.0], [ 6],[ 6], [12],[ 'P'] )  //插入一行
insert into t values(['A', 'B'], [2007.08.01, 2007.08.02], [07:21:45, 07:21:46], [37.03, 37.04],[19.0, 19.1], [ 6, 5],[ 6, 5], [12, 11],[ 'P', 'A'] )  //插入二行
```
##### 也可以使用DolphinDB的内置函数对表进行操作，例如：
```
drop!(t, "OFR")    		           //删除表t中OFR列
columnOFR = t.Date                //单独选出某Date列
```

##### save,仅仅update并不会改变数据在磁盘中的存储，需要将数据写到磁盘中
```
saveText(t, "/home/hduser1/Yanting/数据/taqsmall.csv");
```

##### 如果两个表的数据结构一样，可以将其中一个表插入到另一个表中; 如果两个表的数据结构不一样，可以用table函数构造新表。
```
t1 = loadText("/home/hduser1/Yanting/数据/taqsmallCopy.csv");
tableInsert(t, t1)
newtable = table(t1.Symbol as Symbol, t1.Date as Date, t1.Time as Time, t1.BID as BID, t1.BIDSIZ as BIDSIZ, t1.OFRSIZ as OFRSIZ, t1.MODE as MODE, t1.EX as EX)
tableInsert(t, newtable)
```

##### 使用database数据库, 把刚刚加载的表 t 存储到这个数据库中,(注：saveTable不适用dfs类型的表）
```
db = database("/home/hduser1/Yanting/数据/mydb")
saveTable(db, t, "mytable")
db.saveTable(t, "mytable")     //this will overload 
dropTable(db, "mytable");      //drop this table from db
```

##### 当一个表被加入数据库后，就可以使用loadTable来加载了
```
db = database("/home/hduser1/Yanting/数据/mydb")
mt=loadTable(db, "mytable");
saveTable(db, mt)           // a new table named mt
saveTable(db, mytable)   //overload origin mytable
```

##### 当我们需要加载一个很大的数据的时候,数据的大小超过了内存,如：20G; 可以创造本地分区表来分区,本地分区表一样可以用来进行map、reduce的计算。loadTextEx 函数把输入数据文件USPrices.csv转换为序列域类型的具有两个分区的位于“/home/hduser1/Yanting/数据/seqdb”文件夹中的分布式表pt.并且把pt的元数据加载到内存中。 loadTextEx要求表的行数大于等于65535.
```
db = database("/home/hduser1/Yanting/数据/seqdb", SEQ, 5)
pt =  loadTextEx(db, "pt",  , "/home/hduser1/Yanting/数据/USPrices.csv")
```


##### enjoy DolphinDB :)

