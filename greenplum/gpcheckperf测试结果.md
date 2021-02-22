[gpadmin@node1 ~]$ gpcheckperf -f hostfile_exkeys -d /home/gpadmin -v
[Info] sh -c 'cat /proc/meminfo | grep MemTotal'
MemTotal:         995892 kB

/usr/local/mpp1-db/bin/gpcheckperf -f hostfile_exkeys -d /home/gpadmin -v
--------------------
  SETUP 2019-05-20T19:27:57.524702
--------------------
[Info] verify python interpreter exists
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'python -c print'
[Info] making gpcheckperf directory on all hosts ...
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'rm -rf  /home/gpadmin/gpcheckperf_$USER ; mkdir -p  /home/gpadmin/gpcheckperf_$USER'
[Info] copy local /usr/local/mpp1-db/bin/lib/multidd to remote /home/gpadmin/gpcheckperf_$USER/multidd
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/multidd =:/home/gpadmin/gpcheckperf_$USER/multidd
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/multidd'

--------------------
--  DISK WRITE TEST
--------------------
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'time -p /home/gpadmin/gpcheckperf_$USER/multidd -i /dev/zero -o /home/gpadmin/gpcheckperf_$USER/ddfile -B 32768'

Message from syslogd@node1 at May 20 19:29:01 ...
 kernel:NMI watchdog: BUG: soft lockup - CPU#0 stuck for 24s! [dd:8103]

--------------------
--  DISK READ TEST
--------------------
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'time -p /home/gpadmin/gpcheckperf_$USER/multidd -o /dev/null -i /home/gpadmin/gpcheckperf_$USER/ddfile -B 32768'

--------------------
--  STREAM TEST
--------------------
[Info] copy local /usr/local/mpp1-db/bin/lib/stream to remote /home/gpadmin/gpcheckperf_$USER/stream
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/stream =:/home/gpadmin/gpcheckperf_$USER/stream
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/stream'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys /home/gpadmin/gpcheckperf_$USER/stream

-------------------
--  NETPERF TEST
-------------------
[Info] copy local /usr/local/mpp1-db/bin/lib/gpnetbenchServer to remote /home/gpadmin/gpcheckperf_$USER/gpnetbenchServer
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/gpnetbenchServer =:/home/gpadmin/gpcheckperf_$USER/gpnetbenchServer
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/gpnetbenchServer'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'F=gpnetbenchServer && (pkill $F || pkill -f $F || killall -9 $F) > /dev/null 2>&1 || true'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys '/home/gpadmin/gpcheckperf_$USER/gpnetbenchServer -p 23000 > /dev/null 2>&1'
[Info] copy local /usr/local/mpp1-db/bin/lib/gpnetbenchClient to remote /home/gpadmin/gpcheckperf_$USER/gpnetbenchClient
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/gpnetbenchClient =:/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/gpnetbenchClient'
[Info] ssh -o 'BatchMode yes' -o 'StrictHostKeyChecking no' node1 '/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient -H node2 -p 23000 -t TCP_STREAM -l 15 -fM -P 0 '
[Info] ssh -o 'BatchMode yes' -o 'StrictHostKeyChecking no' node3 '/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient -H node1 -p 23000 -t TCP_STREAM -l 15 -fM -P 0 '
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect
[Info] Connected to server
0     0        32768       14.97     295.16

[Info] node1 -> node2 : ['0', '0', '32768', '14.97', '295.16']
[Info] Connected to server
0     0        32768       14.99     189.65

[Info] node3 -> node1 : ['0', '0', '32768', '14.99', '189.65']
[Info] ssh -o 'BatchMode yes' -o 'StrictHostKeyChecking no' node2 '/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient -H node1 -p 23000 -t TCP_STREAM -l 15 -fM -P 0 '
[Info] ssh -o 'BatchMode yes' -o 'StrictHostKeyChecking no' node1 '/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient -H node3 -p 23000 -t TCP_STREAM -l 15 -fM -P 0 '
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect
[Info] Connected to server
0     0        32768       14.59     178.42

[Info] node2 -> node1 : ['0', '0', '32768', '14.59', '178.42']
[Info] Connected to server
0     0        32768       14.60     287.01

[Info] node1 -> node3 : ['0', '0', '32768', '14.60', '287.01']
--------------------
  TEARDOWN
--------------------
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'rm -rf  /home/gpadmin/gpcheckperf_$USER'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'F=gpnetbenchServer && (pkill $F || pkill -f $F || killall -9 $F) > /dev/null 2>&1 || true'

====================

==  RESULT 2019-05-20T19:34:00.126606
====================

 disk write avg time (sec): 87.79
 disk write tot bytes: 6118735872
 disk write tot bandwidth (MB/s): 91.81
 disk write min bandwidth (MB/s): 11.72 [node1]
 disk write max bandwidth (MB/s): 42.25 [node2]

 disk read avg time (sec): 106.68
 disk read tot bytes: 6118735872
 disk read tot bandwidth (MB/s): 54.82
 disk read min bandwidth (MB/s): 17.39 [node2]
 disk read max bandwidth (MB/s): 19.49 [node1]

 stream tot bandwidth (MB/s): 16005.00
 stream min bandwidth (MB/s): 4582.90 [node3]
 stream max bandwidth (MB/s): 6775.00 [node1]

Netperf bisection bandwidth test
node1 -> node2 = 295.160000
node3 -> node1 = 189.650000
node2 -> node1 = 178.420000
node1 -> node3 = 287.010000

Summary:
sum = 950.24 MB/sec
min = 178.42 MB/sec
max = 295.16 MB/sec
avg = 237.56 MB/sec
median = 287.01 MB/sec

[Warning] connection between node3 and node1 is no good
[Warning] connection between node2 and node1 is no good
[gpadmin@node1 ~]$







# 输出打印解读：

#### 1、先在对端启动服务gpnetbenServer

[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys '/home/gpadmin/gpcheckperf_$USER/gpnetbenchServer -p 23000 > /dev/null 2>&1'

#### 2、在本机启动gpnetbenchClient

[Info] ssh -o 'BatchMode yes' -o 'StrictHostKeyChecking no' node1 '/home/gpadmin/gpcheckperf_$USER/gpnetbenchClient -H node2 -p 23000 -t TCP_STREAM -l 15 -fM -P 0 '

#### 执行命令：

[gpadmin@node1 lib]$ ./gpnetbenchClient -H node3 -P 1 -p 23001 -l 15
Connected to server
               Send
               Message  Elapsed
               Size     Time     Throughput
n/a   n/a      bytes    secs.    MBytes/sec
0     0        32768       14.85     481.50

打印传输的字节，耗时，吞吐量。