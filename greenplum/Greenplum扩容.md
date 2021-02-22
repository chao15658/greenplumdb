# Greenplum扩容

## 扩容分为以下两个阶段

### 第一阶段：

gpexpand -i input_file 执行实例初始化

### 第二阶段：

gpexpand -d 00:20:00 进行数据重分布



## 相关系统表

### gpexpand.status

存储整体扩容进度信息

status字段含义:

​     以下属于初始化阶段：

​     SETUP

​      SETUP DONE

​      以下属于重分布阶段：

​      EXPANSION STARTED

​      EXPANSION STOPPED

​      COMPLETED

```
 myexpand=# select  * from gpexpand.status;

      status       |          updated           

-------------------+----------------------------

SETUP             | 2018-09-18 11:17:29.807489

SETUP DONE        | 2018-09-18 11:17:35.294699

EXPANSION STARTED | 2018-09-18 11:18:02.816792
```



### gpexpand.status_detail

表包含了有关系统扩展操作所涉及的表的状态的信息。用户可以查询此表以确定正在扩展的表的状态，或查看已完成表的开始和结束时间。

```
select  distinct(status) from gpexpand.status_detail where dbname='gpadmincloud';

   status    

\-------------

COMPLETED

NOT STARTED

IN PROGRESS
```

### gpexpand.expansion_progress

记录数据库表重分布速度等信息。

包含下面信息：

```
myexpand=# select  * from gpexpand.expansion_progress;

             name             |         value         

------------------------------+-----------------------

Bytes Done                   | 53412116448

Estimated Time to Completion | 00:16:55.504644

Tables In Progress           | 1

Bytes Left                   | 59420929408

Bytes In Progress            | 142668912

Tables Left                  | 229

Tables Expanded              | 498

Estimated Expansion Rate     | 55.9369898011315 MB/s
```

实际测试时gpexpand.expansion_progress只有Tables Left、Tables Expanded 这两个字段。测试数据库在GP5.0.0的开源版本上进行的。

参考链接：

https://www.jianshu.com/p/c62febb076fd

