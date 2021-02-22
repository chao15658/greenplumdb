# Greenplum锁视图gp_locks_on_relation改良



#### 作用：获取当前连接的数据库上持有、请求的锁信息

## 原始定义：

postgres=# \d gp_toolkit.gp_locks_on_relation
View "gp_toolkit.gp_locks_on_relation"
     Column      |  Type   | Modifiers
-----------------+---------+-----------
 lorlocktype     | text    |
 lordatabase     | oid     |
 lorrelname      | name    |
 lorrelation     | oid     |
 lortransaction  | xid     |
 lorpid          | integer |
 lormode         | text    |
 lorgranted      | boolean |
 lorcurrentquery | text    |
View definition:
 SELECT pgl.locktype AS lorlocktype, pgl.database AS lordatabase, pgc.relname AS lorrelname, pgl.relation AS lorrelation, pgl.transactionid AS lortransaction, pgl.pid AS lorpid, pgl.mode AS lormode, pgl.granted AS lorgranted, pgsa.current_query AS lorcurrentquery
   FROM pg_locks pgl
   JOIN pg_class pgc ON pgl.relation = pgc.oid
   JOIN pg_stat_activity pgsa ON pgl.pid = pgsa.procpid
  ORDER BY pgc.relname;



### 使用效果：

#### 获取Master节点持锁信息：

postgres=# select lorlocktype,lordatabase,lormode,lorgranted,lorrelname from gp_toolkit.gp_locks_on_relation  where lorrelname='test';
 lorlocktype | lordatabase |       lormode       | lorgranted | lorrelname
-------------+-------------+---------------------+------------+------------
 relation    |       12093 | AccessExclusiveLock | t          | test
(1 row)

#### 获取Segment持锁信息：

postgres=# select lorlocktype,lordatabase,lormode,lorgranted,lorrelname from gp_dist_random('gp_toolkit.gp_locks_on_relation')  where lorrelname='test';
 lorlocktype | lordatabase |       lormode       | lorgranted | lorrelname
-------------+-------------+---------------------+------------+------------
 relation    |       12093 | AccessExclusiveLock | t          | test
 relation    |       12093 | AccessExclusiveLock | t          | test
 relation    |       12093 | AccessExclusiveLock | t          | test
 relation    |       12093 | AccessExclusiveLock | t          | test
(4 rows)

如其定义一样，展示不直观。无法展示实例id，会话id，数据库名。



## 改良版（增加gp_segment_id,sess_id,datname）：

 create view locks as 
 SELECT pgl.gp_segment_id,pgl.locktype AS lorlocktype, pgl.database AS lordatabase,pgd.datname, pgc.relname AS lorrelname, pgl.relation AS lorrelation, pgl.transactionid AS lortransaction, pgl.pid AS lorpid, pgl.mode AS lormode, pgl.granted AS lorgranted, pgsa.sess_id,pgsa.current_query AS lorcurrentquery
   FROM pg_locks pgl
   JOIN pg_class pgc ON pgl.relation = pgc.oid
   JOIN pg_stat_activity pgsa ON pgl.pid = pgsa.procpid
   JOIN pg_database pgd on pgl.database=pgd.oid
  ORDER BY pgc.relname;



### 使用效果：

#### 获取Master某个对象持锁情况：

postgres=# select gp_segment_id,lorlocktype,lordatabase,lormode,lorgranted,sess_id,lorrelname,datname from locks where lorrelname='test';
 gp_segment_id | lorlocktype | lordatabase |       lormode       | lorgranted | sess_id | lorrelname | datname
---------------+-------------+-------------+---------------------+------------+---------+------------+----------
            -1 | relation    |       12093 | AccessExclusiveLock | t          |    1236 | test       | postgres
(1 row)

#### 获取Segment上的持锁状况（可展示某个实例id、会话id、数据库名）

postgres=# select gp_segment_id,lorlocktype,lordatabase,lormode,lorgranted,sess_id,lorrelname,datname from gp_dist_random('locks') where lorrelname='test';
 gp_segment_id | lorlocktype | lordatabase |       lormode       | lorgranted | sess_id | lorrelname | datname
---------------+-------------+-------------+---------------------+------------+---------+------------+----------
             2 | relation    |       12093 | AccessExclusiveLock | t          |    1236 | test       | postgres
             1 | relation    |       12093 | AccessExclusiveLock | t          |    1236 | test       | postgres
             3 | relation    |       12093 | AccessExclusiveLock | t          |    1236 | test       | postgres
             0 | relation    |       12093 | AccessExclusiveLock | t          |    1236 | test       | postgres
(4 rows)