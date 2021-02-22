# 调优开关

### gp_enable_explain_allstat

设置为true or false。用于打印每个segment的执行耗时详情。

### explain_memory_verbosity

设置为SUPPRESS, SUMMARY, and DETAIL。用于打印内存使用详情。





gp_enable_explain_allstat的应用：

postgres=# set gp_enable_explain_allstat=on;
SET

postgres=# explain analyze select count(*),a from test5 group by a limit 100;
                                                                           QUERY PLAN

--------------------------------------------------------------------------------------------------------------------
---------------------------------------------
 Limit  (cost=3.50..6.75 rows=100 width=12)
   Rows out:  100 rows with 1267 ms to end, start offset by 9.239 ms.
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=3.50..6.75 rows=100 width=12)
         Rows out:  100 rows at destination with 1267 ms to end, start offset by 9.287 ms.
         ->  Limit  (cost=3.50..4.75 rows=50 width=12)
               Rows out:  Avg 100.0 rows x 2 workers.  Max 100 rows (seg0) with 1266 ms to end, start offset by 10 m
s.
               allstat: seg_firststart_total_ntuples/seg0_10 ms_1266 ms_100/seg1_14 ms_1809 ms_100//end
               ->  HashAggregate  (cost=3.50..4.75 rows=50 width=12)
                     Group By: a
                     Rows out:  Avg 100.0 rows x 2 workers.  Max 100 rows (seg0) with 1266 ms to end, start offset b
y 10 ms.
                     Executor memory:  30298K bytes avg, 30298K bytes max (seg0).
                     allstat: seg_firststart_total_ntuples/seg0_10 ms_1266 ms_100/seg1_14 ms_1809 ms_100//end
                     ->  Seq Scan on test5  (cost=0.00..3.00 rows=50 width=4)
                           Rows out:  Avg 2450100.0 rows x 2 workers.  Max 2450103 rows (seg0) with 0.152 ms to firs
t row, 305 ms to end, start offset by 11 ms.
                           allstat: seg_firststart_total_ntuples/seg0_11 ms_305 ms_2450103/seg1_14 ms_312 ms_2450097
//end
 Slice statistics:
   (slice0)    Executor memory: 322K bytes.
   (slice1)    Executor memory: 30464K bytes avg x 2 workers, 30464K bytes max (seg0).
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 1823.072 ms
(23 rows)



#### 格式解读：

  allstat: seg_firststart_total_ntuples/seg0_11 ms_305 ms_2450103

seg表示实例号

firststart 启动时间

完成时间

totoal_ntuples 从该实例读取的总行数

如seg0_11 ms_305 ms_2450103表示：

seg0启动时间11ms；完成时间305ms；数据量245103行。

seg1_14 ms_312 ms_2450097解读同上。



explain_memory_verbosity的应用：

postgres=# set explain_memory_verbosity=detail;
SET
postgres=# explain analyze select count(*),a from test5 group by a limit 1;
                                                                                                            QUERY PLAN

---------------------------------------------------------------------------------------------------------------
 Limit  (cost=3.50..3.53 rows=1 width=12)
   Rows out:  1 rows with 1852 ms to end, start offset by 0.398 ms.
   Memory:  5K bytes.
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=3.50..3.53 rows=1 width=12)
         Rows out:  1 rows at destination with 1852 ms to end, start offset by 0.406 ms.
         slice 1, seg 0
           Root: Peak/Cur 0/0bytes. Quota: 0bytes.
             Top: Peak/Cur 0/0bytes. Quota: 0bytes.
               Main: Peak/Cur 4888/4744bytes. Quota: 0bytes.
                 Executor: Peak/Cur 130736/108160bytes. Quota: 0bytes.
                   X_Motion: Peak/Cur 5984/5984bytes. Quota: 102400bytes.
                     X_Limit: Peak/Cur 4664/4664bytes. Quota: 102400bytes.
                       X_Agg: Peak/Cur 30966496/30966496bytes. Quota: 130764800bytes.
                         X_TableScan: Peak/Cur 40920/23912bytes. Quota: 102400bytes.
                 Deserializer: Peak/Cur 8480/1008bytes. Quota: 0bytes.
                 Deserializer: Peak/Cur 10272/5888bytes. Quota: 0bytes.
             X_Alien: Peak/Cur 0/0bytes. Quota: 0bytes.
             MemAcc: Peak/Cur 1296/1296bytes. Quota: 0bytes.
             Rollover: Peak/Cur 1046512/1012752bytes. Quota: 0bytes.
             SharedHeader: Peak/Cur 8320/8320bytes. Quota: 0bytes.
         slice 1, seg 1
           Root: Peak/Cur 0/0bytes. Quota: 0bytes.
             Top: Peak/Cur 0/0bytes. Quota: 0bytes.
               Main: Peak/Cur 4888/4744bytes. Quota: 0bytes.
                 Executor: Peak/Cur 130736/108160bytes. Quota: 0bytes.
                   X_Motion: Peak/Cur 5984/5984bytes. Quota: 102400bytes.
                     X_Limit: Peak/Cur 4664/4664bytes. Quota: 102400bytes.
                       X_Agg: Peak/Cur 30966496/30966496bytes. Quota: 130764800bytes.
                         X_TableScan: Peak/Cur 40920/23912bytes. Quota: 102400bytes.
                 Deserializer: Peak/Cur 8480/1008bytes. Quota: 0bytes.
                 Deserializer: Peak/Cur 10272/5888bytes. Quota: 0bytes.
             X_Alien: Peak/Cur 0/0bytes. Quota: 0bytes.
             MemAcc: Peak/Cur 1296/1296bytes. Quota: 0bytes.
             Rollover: Peak/Cur 1047128/1013368bytes. Quota: 0bytes.
             SharedHeader: Peak/Cur 8320/8320bytes. Quota: 0bytes.
         Memory:  14K bytes.
         ->  Limit  (cost=3.50..3.51 rows=1 width=12)
               Rows out:  Avg 1.0 rows x 2 workers.  Max 1 rows (seg0) with 2014 ms to end, start offset by 0.332 ms
.
               Memory:  5K bytes avg, 5K bytes max (seg0).
               allstat: seg_firststart_total_ntuples/seg0_0.332 ms_2014 ms_1/seg1_4.572 ms_1850 ms_1//end
               ->  HashAggregate  (cost=3.50..4.75 rows=50 width=12)
                     Group By: a
                     Rows out:  Avg 1.0 rows x 2 workers.  Max 1 rows (seg0) with 2014 ms to end, start offset by 0.
337 ms.
                     Executor memory:  30298K bytes avg, 30298K bytes max (seg0).
                     Memory:  30241K bytes avg, 30241K bytes max (seg0).
                     allstat: seg_firststart_total_ntuples/seg0_0.337 ms_2014 ms_1/seg1_4.577 ms_1850 ms_1//end
                     ->  Seq Scan on test5  (cost=0.00..3.00 rows=50 width=4)
                           Rows out:  Avg 2450100.0 rows x 2 workers.  Max 2450103 rows (seg0) with 0.153 ms to firs
t row, 314 ms to end, start offset by 0.422 ms.
                           Memory:  40K bytes avg, 40K bytes max (seg0).
                           allstat: seg_firststart_total_ntuples/seg0_0.422 ms_314 ms_2450103/seg1_4.636 ms_317 ms_2
450097//end
 Slice statistics:
   (slice0)    Executor memory: 322K bytes.  Peak memory: 1568K bytes.  Vmem reserved: 1024K bytes.
   (slice1)    Executor memory: 30464K bytes avg x 2 workers, 30464K bytes max (seg0).  Peak memory: 31404K bytes avg x 2 workers, 31404K bytes max (seg1).  Vmem reserved: 30720K bytes avg x 2 workers, 30720K bytes max (seg0).
 Statement statistics:
   Memory used: 128000K bytes
 Settings:  optimizer=off
 Optimizer status: legacy query optimizer
 Total runtime: 2019.450 ms
(58 rows)



#### 参见：

https://yq.aliyun.com/articles/277981