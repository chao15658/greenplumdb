<<<<<<< HEAD
命令
dd
dd是Linux/UNIX下的一个非常有用的命令，作用是用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。
例子：将file1截取大小为1024个4K的块数据，从第100块的偏移量位置开始截取，输出到/opt/file2
dd if=/opt/file1 bs=4K count=1024 skip=100 of=/opt/file2 

  1.if=文件名：输入文件名，缺省为标准输入。即指定源文件。<if=inputfile>
  3.ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。
  obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。
  bs=bytes：同时设置读入/输出的块大小为bytes个字节。
  4.cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。
  5.skip=blocks：从输入文件开头跳过blocks个块后再开始复制。
  6.seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
  注意：通常只用当输出文件是磁盘或磁带时才有效，即备份到磁盘或磁带时才有效。
  7.count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。
  8.conv=conversion：用指定的参数转换文件。
  ascii：转换ebcdic为ascii
  ebcdic：转换ascii为ebcdic
  ibm：转换ascii为alternateebcdic
  block：把每一行转换为长度为cbs，不足部分用空格填充
  unblock：使每一行的长度都为cbs，不足部分用空格填充
  lcase：把大写字符转换为小写字符
  ucase：把小写字符转换为大写字符
  swab：交换输入的每对字节
  noerror：出错时不停止
  notrunc：不截短输出文件
  sync：将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。
  of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file 

truncate
  将某个文件截取到指定大小。
split 将文件按照行、大小分割

nmon
linux系统性能监控命令，主要监控内存、CPU、磁盘读写（IO）、网络传输实时信息，也可监控一段时间CPU变化状况。开源工具，可下载。

ifconfig/ip addr
打印IP地址信息
wondersharper 网络限速工具

free -h
显示缓存占用情况
缓存清理drop_cache
echo 3 > /proc/sys/vm/drop_caches

ping
判断2个机器之间网络连通与否
netstat
查看端口占用情况：netstat -apn | grep 9000
top
ps

tree
以树状图形式显示目录结构。
ldd
 ldd命令用于打印程序或者库文件所依赖的共享库列表。
ls ll显示文件/目录的信息

文本操作相关
more/less/head/tail/cat/grep/sed/cut/find/mkfifio/mv/
cp/scp/mount/df/du/cd/chmod/chown/
yum rpm
gdb pdb
java
make


GP:
gpstart gpstate gpstop gprecoverseg gpload(BUG FIX)/gpfdist/ gptransfer/gpcopy gpcheckcat/gpcheck/gpcheckperf gpconfig 
gpinitsystem gpactivatestandby gpinitstandby 

gpstart 
数据库启动命令


gpstate 
查看GP数据库实例状态。
-c 查看primary和mirror之间的映射关系。
-e 查看当前集群是否有异常实例
-p 列出GP数据库各实例运行的端口
-m 显示mirror信息
-f  列出standby master状态信息
-Q  对宕掉的实例做快速检查

gpstop 
数据库停止命令。
-a 不要求用户确认
-r 重启数据库
-M fast 迅速关闭（关闭连接、事务中断并回滚） immediate 立即关闭（可能造成数据库进入恢复模式） smart 智能关闭（存在活动连接则发出警告）
-u 重新加载配置

gprecoverseg
集群恢复命令
-F 强制恢复实例，删除坏掉的实例数据，重新初始化，并进行数据同步，数据多的话比较慢。不加该参数默认是增量恢复，从宕掉之前开始同步数据，相对比较块。
-r 角色互换。


gpload
数据加载工具，封装了gpfdist。实现原理：读取mapping文件里的配置项和要入库的表（目标表）进行比较，如果字段一致，则调用gpfdist
入库,否则报错。源码存在BUG，列名包含大写字母时，mapping文件列在读取时会自动转成小写，导致匹配失效，无法入库。gptransfer也存在此类问题，对于特殊字符
处理存在BUG，待定位。此类BUG是由于python封装产生的，应查看gpload.py

gpfilespace
根据指定路径，创建文件空间。之后，create tablespace 指定文件空间，可实现数据存放之不同磁盘，有利于读写分离）


SQL:
执行计划、切片
vacuum、analyze、cost计算
系统表、事务、锁
分布式死锁检测（6.0）
create index && insert into partition ==deadlock
参数：
源码：gpload、
      存储：appendonly、heap
      
psql
PG、GP数据库内置连接工具
pg_ctl 数据库内置启停、配置加载管理工具
pg_controldata

\d \d+
显示关系的粗略、详细信息
\l \l+
显示数据库对象粗略、详细信息
\c
链接数据库
\du
显示当前数据库集群内所有角色信息
\df
显示当前搜索路径下函数信息
\x
宽模式
\t
\f
\timing
打开计时器功能
\q

常用系统表：
pg_class
保存几乎所有数据库关系、表、视图、索引等信息。
pg_database
保存所有数据库对象的信息（所有数据库共享该表）
pg_index
保存索引信息
pg_namespace
保存模式信息
pg_filespace_entry（所有数据库共享该表）
获取文件空间的详细信息
pg_tablespace
保存表空间信息（所有数据库共享）
pg_catalog.pg_partition
保存分区表信息（只保留父表信息，不保存子表信息,oid字段唯一地标识一个分区表）
pg_catalog.pg_partition_rule
保存分区表子表信息，分区规则（paroid对应pg_partition的oid字段，parchildrelid pg_class 的oid字段，唯一地标识子分区）

--获取分区表（各个子表）的大小，通过pg_partition和pg_partition_rule两表通过pg_partition的oid字段
--和pg_partition_rule的paroid字段关联取得各个子表以及对应父表信息

常用系统视图：
pg_catalog.pg_tables
pg_catalog.pg_indexes
pg_catalog.pg_views
pg_catalog.pg_partitions
保存分区表信息（只保留分区表子表信息）
pg_catalog.pg_stat_activity
查看当前数据库连接会话
gp_toolkit.pg_locks_on_relation
查看当前数据库对象里面当前的关系上锁的持有情况。
pg_settings
保存参数信息，context字段代表参数是否需要重启数据库，如果是Postmaster属性，则必须重启才能生效。

...................
常用系统函数：

gp_dist_random()
可将要在segment节点执行的任务，在master节点下发并获取结果
pg_size_pretty()
获取关系、数据库大小信息
pg_relation_size()
获取表大小
pg_database_size()
获取数据库大小


................
UDF
获取分区表子表信息
获取各个实例当前执行的语句
获取数据目录


PGOPTIONS='-c gp_session_role' psql -p 40001
连接实例

random（）函数的坑：
在GP的master节点调用时同时生成多行数据并插入某张表时每行数据不发生变化，在单节点正常。
insert into test_group4
select generate_series(1,20),
(random()*200)::int,
(random()*800)::int,
(random()*1600)::int,
(random()*3200)::int,
(random()*6400)::int,
(random()*12800)::int,
(random()*40000)::int,
(random()*100000)::int,
(random()*1000000)::int,
'hello',
'welcome',
'haha',
'chen';

分区创建时不指定名字删除不了

原理：索引加速原理，
分布式死锁检测存在BUG
=======
>>>>>>> e0b909cb965544e458b427c587c288349d8a6b1f

