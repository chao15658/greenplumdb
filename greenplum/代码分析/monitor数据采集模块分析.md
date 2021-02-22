# gpperfmon流程分析



### gpmmon进程调用栈：

[root@node1 ~]# pstack 7415
Thread 5 (Thread 0x7f864ce49700 (LWP 7447)):
#0  0x00007f865153af73 in select () from /lib64/libc.so.6
#1  0x00007f8651c885a5 in apr_sleep () from /lib64/libapr-1.so.0
#2  0x0000000000405002 in conm_main (thread_=<optimized out>, arg_=<optimized out>) at gpmmon.c:551
#3  0x00007f8651a4fdd5 in start_thread () from /lib64/libpthread.so.0
#4  0x00007f8651543ead in clone () from /lib64/libc.so.6
Thread 4 (Thread 0x7f864c648700 (LWP 7448)):
#0  0x00007f8651544483 in epoll_wait () from /lib64/libc.so.6
#1  0x00007f86520e0803 in epoll_dispatch () from /lib64/libevent-2.0.so.5
#2  0x00007f86520cc3ea in event_base_loop () from /lib64/libevent-2.0.so.5
#3  0x000000000040640a in event_main (thread_=<optimized out>, arg_=<optimized out>) at gpmmon.c:398
#4  0x00007f8651a4fdd5 in start_thread () from /lib64/libpthread.so.0
#5  0x00007f8651543ead in clone () from /lib64/libc.so.6
Thread 3 (Thread 0x7f864be47700 (LWP 7449)):
#0  0x00007f865153af73 in select () from /lib64/libc.so.6
#1  0x00007f8651c885a5 in apr_sleep () from /lib64/libapr-1.so.0
#2  0x0000000000405d31 in harvest_main (thread_=<optimized out>, arg_=<optimized out>) at gpmmon.c:749
#3  0x00007f8651a4fdd5 in start_thread () from /lib64/libpthread.so.0
#4  0x00007f8651543ead in clone () from /lib64/libc.so.6
Thread 2 (Thread 0x7f864b646700 (LWP 7450)):
#0  0x00007f8651a53965 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x00007f8651eaccc1 in apr_queue_pop () from /lib64/libaprutil-1.so.0
#2  0x0000000000405bbc in message_main (thread_=<optimized out>, arg_=0x1840fa0) at gpmmon.c:852
#3  0x00007f8651a4fdd5 in start_thread () from /lib64/libpthread.so.0
#4  0x00007f8651543ead in clone () from /lib64/libc.so.6
Thread 1 (Thread 0x7f865280d880 (LWP 7415)):
#0  0x00007f865153af73 in select () from /lib64/libc.so.6
#1  0x00007f8651c885a5 in apr_sleep () from /lib64/libapr-1.so.0
#2  0x0000000000403c4b in gpmmon_main () at gpmmon.c:1036
#3  main (argc=<optimized out>, argv=<optimized out>) at gpmmon.c:1566



### gpsmon进程调用栈：

#0  0x00007f521a81e463 in __epoll_wait_nocancel () from /lib64/libc.so.6
#1  0x00007f521af5c803 in epoll_dispatch () from /lib64/libevent-2.0.so.5
#2  0x00007f521af483ea in event_base_loop () from /lib64/libevent-2.0.so.5
#3  0x0000000000407392 in gx_main (port=port@entry=8888, signature=<optimized out>) at gpsmon.c:1594
#4  0x00000000004037ae in main (argc=10, argv=<optimized out>) at gpsmon.c:1764



### gpperfmon原理：

在GP master节点启动一个gpmmon进程，然后通过gpmmon进程使用ssh方式在集群内每个节点启动一个gpsmon进程去采集其所在主机的监控指标，然后再通过socket通信方式发送到gpmmon进程，gpmmon进程将数据存储到gpperfmon数据库。



#### 具体实现细节：

gpmmon使用libevent框架监听套接字上读事件，（150s没收到数据则返回，由参数配置）。首先发送一个'D'字符，请求数据。当有数据到达时，调用设置的回调函数，实现数据写入文件系统，然后入库。



#### gpmmon各个线程介绍

##### main thread：主线程

每一个配置的时间间隔发送dump命令给gpsmon

存储来自gpsmon的_now和_tail文件

调用harvest每N个间隔将_tail文件数据插入_history表



##### Connection Manager Thread 连接管理线程



##### Event Handling Thread 事件处理线程

gpmmon利用libevent监听套接字读事件，收到数据，调用函数recv_from_gx。

event_set(h->event, h->sock, EV_READ | EV_PERSIST | EV_TIMEOUT,
								recv_from_gx, h);

超时时间150s。

##### Message Thread 消息线程



##### Harvest Thread 收割线程



gpmmon写入流程：

PROCESS:
				1) WITH TAIL MUTEX: rename tail files to stage files  将tail文件重命名为stage文件
				2) WITH TAIL MUTEX: create new tail files  创建新的tail文件
				3) Append data from stage files into _tail files  将stage文件追加到__tail文件
				4) load data from _tail files into system history 将___tail_文件数据加载如system_history
				5) delete _tail files after successful data load  加载成功后删除__tail文件

	NOTES:
				1) mutex is held only over rename and creation of new tail files
				   The reason is this is a fast operation and both the main thread doing dumps and the harvest thread uses the tail files
				2) tail files are renamed/moved because this operation is safe to be done while the clients are reading the files.
				3) The stage files are written over every harvest cycle, so the idea is no client
				   will still be reading the tail files for an entire harvest cycle.  (this is not perfect logic but ok)









#### gpmmon进程介绍

gpsmon也使用libevent监听套接字读事件，收到'D'字符时，将数据发送到gpmmon。

gpsmon调用逻辑如下：

main (argc=10, argv=<optimized out>) at gpsmon.c:1764

gx_main (port=port@entry=8888, signature=<optimized out>) at gpsmon.c:1594

	    setup_gx(port, signature);
	    setup_sigar();
	    setup_udp();
	    setup_tcp();
	    
	    其中setup_tcp()
	    函数内部调用：
	    event_set(&gx.listen_event, sock, EV_READ | EV_PERSIST | EV_TIMEOUT, gx_accept, 0);
	    
	    gx_accept内部调用：
	    	event_set(&gx.tcp_event, nsock, EV_READ | EV_PERSIST | EV_TIMEOUT, gx_gettcpcmd, 0);
	
	gx_gettcpcmd内部调用：
		send_machine_metrics(sock);


​	send_machine_metrics主要负责获取节点的内存、CPU、磁盘、负载信息，主体如下：
​	sigar_mem_get(gx.sigar, &mem);
​		TR2(("mem ram: %" FMT64 " total: %" FMT64 " used: %" FMT64 " free: %" FMT64 "\n",
​			 mem.ram, mem.total, mem.used, mem.free));
​	
​		memset(&swap, 0, sizeof(swap));
​		sigar_swap_get(gx.sigar, &swap);
​		TR2(("swap total: %" FMT64 " used: %" FMT64 "page_in: %" FMT64 " page_out: %" FMT64 "\n",
​			 swap.total, swap.used, swap.page_in, swap.page_out));
​	
​		memset(&cpu, 0, sizeof(cpu));
​		sigar_cpu_get(gx.sigar, &cpu);
​		TR2(("cpu user: %" FMT64 " sys: %" FMT64 " idle: %" FMT64 " wait: %" FMT64 " nice: %" FMT64 " total: %" FMT64 "\n",
​				cpu.user, cpu.sys, cpu.idle, cpu.wait, cpu.nice, cpu.total));
​	
​		memset(&loadavg, 0, sizeof(loadavg));
​		sigar_loadavg_get(gx.sigar, &loadavg);


系统指标采集框架:sigar。

sigar提供的主要API如下：

sigar_mem_get    内存
sigar_swap_get    交换
sigar_cpu_get      CPU  
sigar_loadavg_get    负载
sigar_file_system_usage_get      I/O
sigar_net_interface_stat_get       网络









华为提出的问题：

1、gpmmon怎么识别gpsmon，信息保存在哪里？

调用情况如下：

gpmmon_main

  -> gethostlist

​      -> gpdb_get_hostlist

函数gpdb_get_hostlist 位于文件  gpmmondb.c ，执行下面的SQL获取主机信息，主机信息主要取自gp_segment_configuration。

	// 0 -- hostname, 1 -- address, 2 -- datadir, 3 -- is_master,
	const char *QUERY = "SELECT distinct hostname, address, case when content < 0 then 1 else 0 end as is_master, MAX(fselocation) as datadir FROM pg_filespace_entry "
			    "JOIN gp_segment_configuration on (dbid = fsedbid) WHERE fsefsoid = (select oid from pg_filespace where fsname='pg_system') "
		  	    "GROUP BY (hostname, address, is_master) order by hostname";
2、gpmmon进程挂了怎么办，

它是由postgre主进程拉起的，过一会儿postgres会再拉起一个gpmmon进程。

gpmmon进程会遍历主机列表，每个主机拉起一个gpsmon进程。gpsmon进程负责采集并发送数据到gpmmon端，gpmmon进程接收数据并存储。



蚂蚁金服提出的问题（每个问题问的）

0、监控系统整体介绍

答得还行。深层次的原理了解太少。

1监控系统的IO模型

不了解。libevent事件驱动模型

2 Google 的RPC工具

不会。

3 epoll触发方式

不清楚。边缘触发、条件触发

边缘触发：同一个事件只触发一次。

条件触发：同一个事件触发后不处理的话会不断报告。

4 数据保存策略及删除触发方式

比较模糊。

5 监控对性能的影响

没测过。

6 监控系统的客户端 服务端模型

C/S模型

7、数据采集间隔参数、数据删除参数 响应流程

记不清了。

8、连接管理

不会。

#### 参考链接

<https://blog.csdn.net/wangrbj/article/details/83206147>

https://github.com/greenplum-db/gpdb/wiki/Gpperfmon-Overview