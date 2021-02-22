Greenplum数据库之间的迁移：
gptransfer 
前提：源主机集群和目的主机集群要做密钥交换。
使用：
1、	具有较为丰富的参数可供使用，指定是否进行表分析、行数校验。
2、	能够指定在数据迁移期间对相关对象持排他锁，防止数据变更导致失败。
影响迁移效率的重要参数
--batch-size 一次批量传输中可传输表的数量
--sub-batch-size  指定传输一张表起的并行子进程数目。
并行度 –batch-size * --sub-batch-size
默认值--batch-size 5
      --sub-batch-size 2

局限性：
1、只能做GP数据库之间的数据迁移。
2、默认数据库postgres、template0、template1无法传输。
3、角色、资源队列、自定义函数、存储过程不能传输。
4、配置文件pg_hba.conf、postgresql.conf无法传输。
5、不适合小表迁移，因为迁移过程要建立、移除外部表，耗时占大部分的时间。

