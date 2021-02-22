# Greenplum sql调优

## 1、表分析（是否进行了表分析，统计信息是否准确）

## 2、查看数据倾斜（有倾斜、是否应当调整分布键）

## 3、阅读执行计划（是否走的是预期的计划，有没有走索引、进行分区裁剪）

## 4、查看耗时在哪，表扫描、重分布、广播。分布键选取是否合适。

## 5、是否存在性能瓶颈，查看nmon输出，如果查询运行期间存在大量cpu 空闲，IO busy 100%，说明有IO瓶颈，则需要考虑扩容每台物理机实例数。

## 6、参数有没有必要调整（导致执行计划走错了）

## 7、是否存在使用不当情况，例如带索引入库导致IO很高、update大表也会导致IO出现比较高的情况。应调整业务。

## 8、集群网络是否良好，使用千兆/万兆网卡（gpcheckperf可检测网络）





## 例子：

#### 1、数据倾斜问题排查

使用sql： select count(1),gp_segment_id from schema_name.table_name group by gp_segment_id order by 1 desc;

按照每个实例数据规模逆序排序。

一个精确查询。未调整分布键，耗时10s。当时查看执行计划时，主要耗时在表扫描上，查看当时的数据倾斜情况。

#### 原始sql：

sql1:

select * from zdwry.gas_hour
where RevisedStrength = '172.707000';

sql2:

select * from zdwry.gas_hour
where psname = '天津国电津能热电有限公司';

原始表数据分布状况：
data_center_result=# **select count(1),gp_segment_id from zdwry.gas_hour group by gp_segment_id order by 1 desc;**
  count   | gp_segment_id
----------+---------------
 37499985 |             1
 11137679 |             3
  8521066 |             0
  7589493 |             2
(4 rows)

调整分布键之后：
data_center_result=# select count(1),gp_segment_id from zdwry.gas_hour group by gp_segment_id order by 1 desc;
  count   | gp_segment_id
----------+---------------
 17044424 |             1
 16858398 |             2
 16000698 |             0
 14844703 |             3
(4 rows)

数据倾斜情况大大改善，具体调整办法如下。

## 分布键选取过程：

原始表按照分布键分组：

distinct有去重的功能。

data_center_result=# select count(distinct(regioncode))from zdwry.gas_hour;
 count
-------
    15
(1 row)

data_center_result=# select count(regioncode) from zdwry.gas_hour;
  count
----------
 64748223
(1 row)

发现重复率极高，只有15个不同值，可区分度为‭2.316665895216924e-7‬。

#### 查看sql1的where过滤字段的重复率

 select count(distinct(RevisedStrength))::float /count(RevisedStrength)::float from zdwry.gas_hour;
      ?column?
--------------------
 0.0110648750931744

#### sql2的过滤条件：

select count(distinct(psname))::float /count(psname)::float from zdwry.gas_hour;
       ?column?
----------------------
 2.82633239216465e-06
(1 row)

由以上可知，按照

这里选取RevisedStrength或psname分布的表重复率一定比按照原始字段分布的更加均匀。这里选RevisedStrength、psname、regioncode 3个字段值做一个联合分布键。

alter table zdwry.gas_hour set distributed by(RevisedStrength,psname,regioncode);

#### 结论：

sql1耗时由10s左右降低到了4-5s左右。

针对sql1按照RevisedStrength建立索引

create index idx_RevisedStrength on zdwry.gas_hour(RevisedStrength);

耗时由4-5s降低至18ms。



## 分布键选取：尽可能是的数据分布均匀，可先使用distinct对其中几个字段去重，选取重复率较低、经常join/过滤的字段做分布键。



## 实例2：

### 2、关联键强制类型转换，导致重分布

两张表关联键都是int类型，如果强制将两个int类型转换为其他类型，会导致两个表都重分布

postgres=# explain select * from test3 a join test4 b on a.a=b.a;

                                    QUERY PLAN
-----------------------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)  (cost=0.00..871.31 rows=40000 width=16)
   ->  Hash Join  (cost=0.00..868.44 rows=20000 width=16)
         Hash Cond: test3.a = test4.a
         ->  Table Scan on test3  (cost=0.00..431.21 rows=10000 width=8)
         ->  Hash  (cost=431.21..431.21 rows=10000 width=8)
               ->  Table Scan on test4  (cost=0.00..431.21 rows=10000 width=8)
 Settings:  enable_seqscan=off
 Optimizer status: PQO version 2.42.0
(8 rows)

如果强制将两个int类型转换为其他类型

postgres=# explain select * from test3 a join test4 b on a.a::numeric=b.a::numeric;
                                                QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice3; segments: 2)  (cost=0.00..871.81 rows=40000 width=16)
   ->  Hash Join  (cost=0.00..868.94 rows=20000 width=16)
         Hash Cond: test3.a::numeric = test4.a::numeric
         ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=0.00..431.61 rows=10000 width=8)
               Hash Key: test3.a::numeric
               ->  Table Scan on test3  (cost=0.00..431.21 rows=10000 width=8)
         ->  Hash  (cost=431.61..431.61 rows=10000 width=8)
               ->  Redistribute Motion 2:2  (slice2; segments: 2)  (cost=0.00..431.61 rows=10000 width=8)
                     Hash Key: test4.a::numeric
                     ->  Table Scan on test4  (cost=0.00..431.21 rows=10000 width=8)
 Settings:  enable_seqscan=off
 Optimizer status: PQO version 2.42.0
(12 rows)



## 实例3：

模拟方法：

手动更新统计信息

ostgres=# update pg_class set reltuple =1 , relpage=1 where relname='test3';
ERROR:  permission denied: "pg_class" is a system catalog
postgres=# set allow_system_table_mods='DML';
SET

postgres=# set **allow_system_table_mods**='DML';
SET

postgres=# update pg_class set reltuples =1, relpages=1 where relname='test3';
UPDATE 1

### 2、统计信息过期

需要手动进行analyze

如下，统计信息不准导致大表被重广播，耗时加大

当两张表的关联键其中有一个不是表的分布键时，Greenplum选择将分布键不是关联键的表按照关联键重分布或者将另外一张表广播给所有实例。

具体策略：

如下所示，该sql语句：

 SELECT count(*)   FROM test3 INNER JOIN test4                                                                                                          ON test3.a = test4.b;

其中test3按照a字段分布，test4也按照a字段分布。因此，test4的表其关联键不是分布键。内关联时遵循上述原则。

postgres=# select reltuples,relpages from pg_class where relname='test4';
 reltuples | relpages
-----------+----------
     20000 |       23
(1 row)

postgres=# select reltuples,relpages from pg_class where relname='test3';
 reltuples | relpages
-----------+----------
         1 |        1
(1 row)

postgres=# select count(*) from test3;
  count
---------
 1280000
(1 row)

postgres=# select count(*) from test4;
 count
-------
 20000
(1 row)

清理系统缓存

[root@node1 ~]# echo 3 >/proc/sys/vm/drop_caches
修改系统表pg_class

postgres=# set allow_system_table_mods='DML';
SET
postgres=# update pg_class set reltuples =1, relpages=1 where relname='test3';                                                       UPDATE 1
postgres=# explain analyze SELECT count(*)                                                                                           FROM test3 INNER JOIN test4                                                                                                          ON test3.a = test4.b;
                                                                                      QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------
-------------------------------------------------
 Aggregate  (cost=274.30..274.31 rows=1 width=8)
   Rows out:  1 rows with 1439 ms to end, start offset by 8.418 ms.
   ->  Gather Motion 2:1  (slice2; segments: 2)  (cost=274.25..274.29 rows=1 width=8)
         Rows out:  2 rows at destination with 1436 ms to first row, 1439 ms to end, start offset by 8.447 ms.
         ->  Aggregate  (cost=274.25..274.26 rows=1 width=8)
               Rows out:  Avg 1.0 rows x 2 workers.  Max 1 rows (seg0) with 1435 ms to end, start offset by 16 ms.
               ->  Hash Join  (cost=1.06..274.21 rows=8 width=0)
                     Hash Cond: test4.b = test3.a
                     Rows out:  Avg 640000.0 rows x 2 workers.  Max 640128 rows (seg0) with 1190 ms to first row, 1388 ms to end, sta
rt offset by 16 ms.
                     Executor memory:  30000K bytes avg, 30000K bytes max (seg0).
                     Work_mem used:  30000K bytes avg, 30000K bytes max (seg0). Workfile: (0 spilling)
                     (seg0)   Hash chain length 129.1 avg, 256 max, using 9918 of 524288 buckets.
                     ->  Seq Scan on test4  (cost=0.00..223.00 rows=10000 width=4)
                           Rows out:  Avg 10000.0 rows x 2 workers.  Max 10002 rows (seg0) with 0.192 ms to first row, 1.782 ms to en
d, start offset by 1205 ms.
                     ->  Hash  (cost=1.04..1.04 rows=1 width=4)
                           Rows in:  Avg 1280000.0 rows x 2 workers.  Max 1280000 rows (seg0) with 1181 ms to end, start offset by 24
 ms.
                           ->  **Broadcast** Motion 2:2  (slice1; segments: 2)  (cost=0.00..1.04 rows=1 width=4)
                                 Rows out:  Avg 1280000.0 rows x 2 workers at destination.  Max 1280000 rows (seg0) with 0.235 ms to
first row, 756 ms to end, start offset by 24 ms.
                                 ->  Seq Scan on test3  (cost=0.00..1.01 rows=1 width=4)
                                       Rows out:  Avg 640000.0 rows x 2 workers.  Max 640128 rows (seg0) with 0.104 ms to first row,
81 ms to end, start offset by 17 ms.
 Slice statistics:
   (slice0)    Executor memory: 386K bytes.
   (slice1)    Executor memory: 175K bytes avg x 2 workers, 175K bytes max (seg0).
   (slice2)    Executor memory: 74019K bytes avg x 2 workers, 74019K bytes max (seg0).  Work_mem: 30000K bytes max.
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 1447.442 ms
(29 rows)

#### 执行analyze更新统计信息

postgres=# analyze test3;
ANALYZE
postgres=# select reltuples,relpages from pg_class where relname='test3';
  reltuples  | relpages
-------------+----------
 1.28078e+06 |     1409
(1 row)

postgres=# explain analyze SELECT count(*)                                                                                           FROM test3 INNER JOIN test4                                                                                                          ON test3.a = test4.b;
                                                                                     QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------
 Aggregate  (cost=27897.68..27897.69 rows=1 width=8)
   Rows out:  1 rows with 743 ms to end, start offset by 16 ms.
   ->  Gather Motion 2:1  (slice2; segments: 2)  (cost=27897.62..27897.66 rows=1 width=8)
         Rows out:  2 rows at destination with 692 ms to first row, 743 ms to end, start offset by 16 ms.
         ->  Aggregate  (cost=27897.62..27897.63 rows=1 width=8)
               Rows out:  Avg 1.0 rows x 2 workers.  Max 1 rows (seg0) with 691 ms to end, start offset by 23 ms.
               ->  Hash Join  (cost=873.00..26296.64 rows=320196 width=0)
                     Hash Cond: test3.a = test4.b
                     Rows out:  Avg 640000.0 rows x 2 workers.  Max 640128 rows (seg0) with 16 ms to first row, 492 ms to end, start
offset by 23 ms.
                     Executor memory:  157K bytes avg, 157K bytes max (seg0).
                     Work_mem used:  157K bytes avg, 157K bytes max (seg0). Workfile: (0 spilling)
                     (seg0)   Hash chain length 1.0 avg, 2 max, using 4974 of 524288 buckets.
                     ->  Seq Scan on test3  (cost=0.00..14216.81 rows=640391 width=4)
                           Rows out:  Avg 640000.0 rows x 2 workers.  Max 640128 rows (seg0) with 0.122 ms to first row, 207 ms to en
d, start offset by 39 ms.
                     ->  Hash  (cost=623.00..623.00 rows=10000 width=4)
                           Rows in:  Avg 5000.0 rows x 2 workers.  Max 5001 rows (seg0) with 11 ms to end, start offset by 28 ms.
                           ->  **Redistribute Motion** 2:2  (slice1; segments: 2)  (cost=0.00..623.00 rows=10000 width=4)
                                 Hash Key: test4.b
                                 Rows out:  Avg 10000.0 rows x 2 workers at destination.  Max 15001 rows (seg0) with 0.122 ms to firs
t row, 8.932 ms to end, start offset by 28 ms.
                                 ->  Seq Scan on **test4**  (cost=0.00..223.00 rows=10000 width=4)
                                       Rows out:  Avg 10000.0 rows x 2 workers.  Max 10002 rows (seg0) with 0.125 ms to first row, 1.
444 ms to end, start offset by 29 ms.
 Slice statistics:
   (slice0)    Executor memory: 386K bytes.
   (slice1)    Executor memory: 175K bytes avg x 2 workers, 175K bytes max (seg0).
   (slice2)    Executor memory: 8675K bytes avg x 2 workers, 8675K bytes max (seg0).  Work_mem: 157K bytes max.
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 759.800 ms
(30 rows)



#### 3、分布键选择不当，导致查询速度慢



#### 4、运行时数据倾斜，关联键都不是分布键时：

当M+N>min(M,N) * 节点数时，两张表都重分布；

反之，小表广播

具体见Greenplum企业应用实战 P135页。

下面参数控制数据是重分布还是广播：

gp_segments_for_planner
假设在其成本和大小估计中，设置遗传查询优化器（计划器）主要段实例数。如果为0，则使用的值是实际的主要段数。该变量影响传统优化器对移动操作符中每个发送和接受进程处理的行数的估计。
若解释计划显示对大量数据集使用了广播移动操作符，需要尝试避免使用广播操作符。一种方法是使用gp_Segments_for_planner 配置参数增加移动数据的估计代价。
该变量告诉优化器在计算移动代价时使用多少个Segments。默认值是0，意味着使用实际Segment个数。增大这个数字，移动的代价会跟着增大，优化器会优先使用
分发移动操作符。例如设置gp_Segments_for_planner=100000 告诉优化器有100000个Segments。
相反为了优先使用广播移动操作符，为该值设置一个小数字，例如2。

参考：
http://www.wodefanwen.com/lhd_1ubia2dy4y4bptb11x4w7g2499ip7300mjg_10.html



5、表关联时有一张表关联键不是分布键

规则，

M：分布键和关联键不一致的表数据量，即需要广播的数据量

N：分布键和关联键不一致的表数据量，即需要广播的数据量

当M> N*节点数时，选择将分布键一致的表广播；反之，将分布键不一致的表重分布

110w*2>190w,因此，下面执行计划将test4重分布

postgres=# select count(*) from test3;

  count
---------
 1110100
(1 row)

postgres=# select count(*) from test4;
  count
---------
 1900100
(1 row)



postgres=# explain analyze select * from test3 join test4 on test3.a=test4.b;
                                                                             QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------
--------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=26214.96..117221.48 rows=2168930 width=16)
   Rows out:  333340 rows at destination with 305 ms to first row, 1822 ms to end, start offset by 19 ms.
   ->  Hash Join  (cost=26214.96..117221.48 rows=1084465 width=16)
         Hash Cond: test4.b = test3.a
         Rows out:  Avg 166670.0 rows x 2 workers.  Max 166675 rows (seg1) with 303 ms to first row, 1615 ms to end, start offset by
24958 ms.
         Executor memory:  17346K bytes avg, 17346K bytes max (seg0).
         Work_mem used:  17346K bytes avg, 17346K bytes max (seg0). Workfile: (0 spilling)
         (seg1)   Hash chain length 1.7 avg, 10 max, using 322388 of 524288 buckets.
         ->  **Redistribute Motion** 2:2  (slice1; segments: 2)  (cost=0.00..59140.84 rows=950814 width=8)
               Hash Key: test4.b
               Rows out:  Avg 950050.0 rows x 2 workers at destination.  Max 950050 rows (seg0) with 0.496 ms to first row, 772 ms to
 end, start offset by 51716 ms.
               ->  Seq Scan on test4  (cost=0.00..21108.28 rows=950814 width=8)
                     Rows out:  Avg 950050.0 rows x 2 workers.  Max 950051 rows (seg0) with 0.729 ms to first row, 114 ms to end, sta
rt offset by 51433 ms.
         ->  Hash  (cost=12329.98..12329.98 rows=555399 width=8)
               Rows in:  Avg 555050.0 rows x 2 workers.  Max 555053 rows (seg0) with 296 ms to end, start offset by 51420 ms.
               ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=8)
                     Rows out:  Avg 555050.0 rows x 2 workers.  Max 555053 rows (seg0) with 0.575 ms to first row, 54 ms to end, star
t offset by 51420 ms.
 Slice statistics:
   (slice0)    Executor memory: 386K bytes.
   (slice1)    Executor memory: 175K bytes avg x 2 workers, 175K bytes max (seg0).
   (slice2)    Executor memory: 41247K bytes avg x 2 workers, 41247K bytes max (seg0).  Work_mem: 17346K bytes max.
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 1861.818 ms
(26 rows)



给test4再插入40w条数据，再次提交sql时发现test3被广播了。

计算公式：

110w*2<230w

postgres=# insert into test4 select generate_series(1,400000),generate_series(1,400000);
INSERT 0 400000

postgres=# select count(*) from test4;
  count
---------
 2300100
(1 row)

postgres=# analyze test4;
ANALYZE
postgres=# select count(*) from test3;

  count
---------
 1110100
(1 row)

postgres=# explain analyze select * from test3 join test4 on test3.a=test4.b;
                                                                              QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------
----------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=56565.73..144570.15 rows=2013901 width=16)
   Rows out:  843440 rows at destination with 740 ms to first row, 2825 ms to end, start offset by 7.883 ms.
   ->  Hash Join  (cost=56565.73..144570.15 rows=1006951 width=16)
         Hash Cond: test3.a = test4.b
         Rows out:  Avg 421720.0 rows x 2 workers.  Max 421731 rows (seg0) with 738 ms to first row, 2431 ms to end, start offset by
51400 ms.
         Executor memory:  35940K bytes avg, 35940K bytes max (seg0).
         Work_mem used:  35940K bytes avg, 35940K bytes max (seg0). Workfile: (0 spilling)
         (seg0)   Hash chain length 2.8 avg, 18 max, using 408128 of 524288 buckets.
         ->  **Broadcast Motion** 2:2  (slice1; segments: 2)  (cost=0.00..45653.92 rows=1110798 width=8)
               Rows out:  Avg 1110100.0 rows x 2 workers at destination.  Max 1110100 rows (seg0) with 0.185 ms to first row, 838 ms
to end, start offset by 52138 ms.
               ->  Seq Scan on **test3**  (cost=0.00..12329.98 rows=555399 width=8)
                     Rows out:  Avg 555050.0 rows x 2 workers.  Max 555053 rows (seg0) with 0.134 ms to first row, 56 ms to end, star
t offset by 51409 ms.
         ->  Hash  (cost=25547.88..25547.88 rows=1150794 width=8)
               Rows in:  Avg 1150050.0 rows x 2 workers.  Max 1150051 rows (seg0) with 732 ms to end, start offset by 51406 ms.
               ->  Seq Scan on test4  (cost=0.00..25547.88 rows=1150794 width=8)
                     Rows out:  Avg 1150050.0 rows x 2 workers.  Max 1150051 rows (seg0) with 0.081 ms to first row, 68 ms to end, st
art offset by 51406 ms.
 Slice statistics:
   (slice0)    Executor memory: 386K bytes.
   (slice1)    Executor memory: 175K bytes avg x 2 workers, 175K bytes max (seg0).
   (slice2)    Executor memory: 65823K bytes avg x 2 workers, 65823K bytes max (seg0).  Work_mem: 35940K bytes max.
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 2876.962 ms

6、计算distinct

5.0测试需要关闭enable_hashagg 

postgres=# set enable_hashagg =off;

postgres=# explain select distinct b from test3;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=245647.55..268186.83 rows=901571 width=4)
   ->  GroupAggregate  (cost=245647.55..268186.83 rows=450786 width=4)
         Group By: test3.b
         ->  Sort  (cost=245647.55..247901.48 rows=450786 width=4)
               Sort Key: test3.b
               ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=123871.68..156472.80 rows=450786 width=4)
                     Hash Key: test3.b
                     ->  GroupAggregate  (cost=123871.68..138441.38 rows=450786 width=4)
                           Group By: test3.b
                           ->  Sort  (cost=123871.68..126648.67 rows=555399 width=4)
                                 Sort Key: test3.b
                                 ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=4)
 Settings:  enable_hashagg=off; optimizer=off
 Optimizer status: legacy query optimizer
(14 rows)

IO开销降低了268/68=4倍左右。

postgres=# show enable_hashagg ;
 enable_hashagg
----------------
 on
(1 row)

postgres=# explain select distinct b from test3;

##                                                 QUERY PLAN

-----------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=56560.92..67667.68 rows=901571 width=4)
   ->  HashAggregate  (cost=56560.92..67667.68 rows=450786 width=4)
         Group By: test3.b
         ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=15106.97..43200.24 rows=450786 width=4)
               Hash Key: test3.b
               ->  HashAggregate  (cost=15106.97..25168.82 rows=450786 width=4)
                     Group By: test3.b
                     ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=4)
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
(10 rows)

postgres=# explain select b from test3 group by b;
                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=56560.92..67667.68 rows=901571 width=4)
   ->  HashAggregate  (cost=56560.92..67667.68 rows=450786 width=4)
         Group By: test3.b
         ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=15106.97..43200.24 rows=450786 width=4)
               Hash Key: test3.b
               ->  HashAggregate  (cost=15106.97..25168.82 rows=450786 width=4)
                     Group By: test3.b
                     ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=4)
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
(10 rows)

GP4.3以后distinct和group by都采用了HASHAggregate方式聚合，性能差别不大了。

功能验证：

postgres=# select * from test8;
 a  |  b
----+-----
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
 26 | 260
 28 | 280
 26 | 260
 28 | 280
(14 rows)

postgres=# select distinct a from test8;
 a
----
  1
  3
  5
  7
  9
 26
 28
(7 rows)

改写为group by语法：

postgres=# select  a from test8 group by a;
 a
----
 26
 28
  1
  3
  5
  7
  9
(7 rows)



#### 总结：

GP4.3之前distinct采用分组聚合，4.3之后采用HASH聚合。因此，在4.3以前推荐将distinct改为group by。group by采用HASH聚合，适合聚合函数较少的情况。



#### HASH聚合

在内存中按照group by的字段在内存中建立HASH表，并在内存中维护该列表。字段重复率越低，使用的内存越大。

多个聚合函数就会维护多个HASH表，因此，HASH聚合不适合聚合函数太多的情况。



#### 分组聚合

按照group by字段排序，然后执行扫描。由于要排序，效率很差。

7、NOT IN改写成LEFT JOIN

NOT IN/IN这种语法会产生笛卡尔乘积，导致产生的中间结果是关联的两张表数据量乘积，导致耗时大大增加。

postgres=# set enable_hashjoin =off;
SET

postgres=# explain select * from test3 where b not in (select a from test3);                                                                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=61153.50..57934753999.66 rows=21 width=8)
   ->  Nested Loop Left Anti Semi Join (Not-In)  (cost=61153.50..57934753999.66 rows=11 width=8)
         Join Filter: public.test3.b = "NotIn_SUBQUERY".a
         ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=8)
         ->  Materialize  (cost=61153.50..85539.46 rows=1110798 width=4)
               ->  Broadcast Motion 2:2  (slice1; segments: 2)  (cost=0.00..56761.90 rows=1110798 width=4)
                     ->  Subquery Scan "NotIn_SUBQUERY"  (cost=0.00..23437.96 rows=555399 width=4)
                           ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=4)
 Settings:  enable_hashjoin=off; optimizer=off
 Optimizer status: legacy query optimizer
(10 rows)

改下语法：

IO消耗降低很多599/10=60倍左右。之前由5min都没有跑完，降至1.2s左右。

注意：空值处理

postgres=# explain select * from test3 a left join (select a from test3) b on a.b=b.a where b.a is null ;                                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Gather Motion 2:1  (slice2; segments: 2)  (cost=1000269961.66..1000296041.20 rows=1368526 width=12)
   ->  Merge Left Join  (cost=1000269961.66..1000296041.20 rows=684263 width=12)
         Merge Cond: a.b = test3.a
         Filter: test3.a IS NULL
         ->  Sort  (cost=146087.64..148864.63 rows=555399 width=8)
               Sort Key: a.b
               ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=0.00..34545.94 rows=555399 width=8)
                     Hash Key: a.b
                     ->  Seq Scan on test3 a  (cost=0.00..12329.98 rows=555399 width=8)
         ->  Materialize  (cost=123871.68..137756.65 rows=555399 width=4)
               ->  Sort  (cost=123871.68..126648.67 rows=555399 width=4)
                     Sort Key: test3.a
                     ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=4)
 Settings:  enable_hashjoin=off; optimizer=off
 Optimizer status: legacy query optimizer
(15 rows)

或改为NOT EXISTS语法：

NOT EXISTS类似NOT IN语法

postgres=# explain  select a ,b from test8 where NOT EXISTS(select a7 from test7 where a7=a);

##                                  QUERY PLAN

 Gather Motion 2:1  (slice1; segments: 2)  (cost=1.11..2.18 rows=4 width=8)
   ->  Hash Left Anti Semi Join  (cost=1.11..2.18 rows=2 width=8)
         Hash Cond: test8.a = test7.a7
         ->  Seq Scan on test8  (cost=0.00..1.02 rows=1 width=8)
         ->  Hash  (cost=1.05..1.05 rows=3 width=4)
               ->  Seq Scan on test7  (cost=0.00..1.05 rows=3 width=4)
                     Filter: a7 IS NOT NULL
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
(9 rows)

功能验证：

postgres=# select * from test7;
 a | b
---+----
 1 | 10
 3 | 30
 5 | 50
 7 | 70
 9 | 90
(5 rows)

postgres=# select * from test8;
 a  |  b
----+-----
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
 26 | 260
 28 | 280
(7 rows)

原始IN语句：

postgres=# select * from test8 where a not in (select a from test7); 

 a  |  b
----+-----
 26 | 260
 28 | 280
(2 rows)

改为left join后：

postgres=# select a.a,a.b from test8 a left join (select a from test7) b on a.a=b.a where b.a is null ;
 a  |  b
----+-----
 26 | 260
 28 | 280
(2 rows)



改为not exists

postgres=# select a ,b from test8 where NOT exists (select a7 from test7 where a7=a);
 a  |  b
----+-----
 26 | 260
 28 | 280
 26 | 260
 28 | 280
(4 rows)

#### 　　区别及应用场景

　 in 和 exists的区别: 如果子查询得出的结果集记录较少，主查询中的表较大且又有索引时应该用in, 反之如果外层的主查询记录较少，子查询中的表大，又有索引时使用exists。其实我们区分in和exists主要是造成了驱动顺序的改变(这是性能变化的关键)，如果是exists，那么以外层表为驱动表，先被访问，如果是IN，那么先执行子查询，所以我们会以驱动表的快速返回为目标，那么就会考虑到索引及结果集的关系了 ，另外IN时不对NULL进行处理。

　in 是把外表和内表作hash 连接，而exists是对外表作loop循环，每次loop循环再对内表进行查询。一直以来认为exists比in效率高的说法是不准确的。

参考：

https://www.cnblogs.com/cjm123/p/8177151.html

8.UNION改写为UNION ALL加GROUP BY

UNION去重排序，比较耗时

postgres=# explain insert into test5 select * from test4 union select * from test3;
                                                     QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Insert (slice0; segments: 2)  (rows=1706193 width=8)
   ->  Redistribute Motion 2:2  (slice2; segments: 2)  (cost=613935.19..639528.08 rows=1706193 width=8)
         Hash Key: test4.a
         ->  Unique  (cost=613935.19..639528.08 rows=1706193 width=8)
               Group By: test4.a, test4.b
               ->  Sort  (cost=613935.19..622466.15 rows=1706193 width=8)
                     Sort Key (Distinct): test4.a, test4.b
                     ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=0.00..72001.72 rows=1706193 width=8)
                           Hash Key: test4.a, test4.b
                           ->  Append  (cost=0.00..72001.72 rows=1706193 width=8)
                                 ->  Seq Scan on test4  (cost=0.00..25547.88 rows=1150794 width=8)
                                 ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=8)
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
(14 rows)

发现IO消耗降低63w/8w=8倍左右。

postgres=# explain insert into test5 select * from test4 union all select * from test3 group by a,b;
                                            QUERY PLAN
--------------------------------------------------------------------------------------------------
 Insert (slice0; segments: 2)  (rows=1601580 width=8)
   ->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=0.00..89392.16 rows=1601580 width=8)
         Hash Key: test4.a
         ->  Append  (cost=0.00..89392.16 rows=1601580 width=8)
               ->  Seq Scan on test4  (cost=0.00..25547.88 rows=1150794 width=8)
               ->  HashAggregate  (cost=20340.48..31812.69 rows=450786 width=8)
                     Group By: test3.a, test3.b
                     ->  Seq Scan on test3  (cost=0.00..12329.98 rows=555399 width=8)
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
(10 rows)

验证功能：

postgres=# select * from test7;
 a | b
---+----
 1 | 10
 3 | 30
 5 | 50
 7 | 70
 9 | 90
(5 rows)

postgres=# select * from test8;
 a  |  b
----+-----
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
 26 | 260
 28 | 280
(7 rows)

union，连接test7和test8，去重。

postgres=# select * from test7 union select * from test8;
 a  |  b
----+-----
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
 26 | 260
 28 | 280
(7 rows)

修改为union all + group by之后

postgres=# select * from (select * from test7 union all select * from test8 ) t group by a,b order by a;
 a  |  b
----+-----
  1 |  10
  3 |  30
  5 |  50
  7 |  70
  9 |  90
 26 | 260
 28 | 280
(7 rows)

注意：给结果集加括号，否则结果可能会不同。





#### 参考资料：

1、PostgresChina2018_王昊_Greenplum5_智能运维管理实例及展望.PDF

2、Greenplum企业应用实战.PDF