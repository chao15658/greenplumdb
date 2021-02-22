# gpcheckperf检测网络原理：

执行网络性能测试的代码：

奇数个节点：取出最后一个节点，在相邻2个节点间进行测试，最后在第一个节点和之前去除的那个节点间进行测试

偶数个节点：相邻2个节点间进行测试

```
#############
def runNetPerfTest():
    (netperf, hostlist, netserver_port) = setupNetPerfTest()
    if not netperf:
        return None

​```
# make sure we have even number of hosts
odd = None
h = hostlist[:]
if len(h) & 1 == 1:
    odd = h.pop()  # take one out

# run netperf test between host[0] & host[1], host[2] & host[3], ...
result = []
for i in xrange(len(h)):
    if i & 1 == 0: continue
    x = h[i - 1]
    y = h[i]
    res = runNetperfTestBetween(x, y, netperf, netserver_port)
    if res and len(res) >= 6: result.append(res)
    res = runNetperfTestBetween(y, x, netperf, netserver_port)
    if res and len(res) >= 6: result.append(res)

# run netperf test between the odd host and host 0
if odd:
    x = odd
    y = h[0]
    res = runNetperfTestBetween(x, y, netperf, netserver_port)
    if res and len(res) >= 6: result.append(res)
    res = runNetperfTestBetween(y, x, netperf, netserver_port)
    if res and len(res) >= 6: result.append(res)

return result
​```
```

当其中某两个节点的网络测试速度<0.9 * max时，这两个节点就被认为网络状况不好

```
#############
def printNetResult(result):
    print 'Netperf bisection bandwidth test'
    for h in result:
        print '%s -> %s = %f' % (h[0], h[1], h[6])

​```
sum = min = max = avg = 0
n = map(lambda x: float(x[6]), result)
sum = reduce(lambda x, y: x + y, n)
min = reduce(lambda x, y: x > y and y or x, n, 9999999999)
max = reduce(lambda x, y: x > y and x or y, n)
avg = float(sum) / len(result)

copy = n[:]
copy.sort()
median = copy[len(copy) / 2]

print ''
print 'Summary:'
print 'sum = %.2f MB/sec' % sum
print 'min = %.2f MB/sec' % min
print 'max = %.2f MB/sec' % max
print 'avg = %.2f MB/sec' % avg
print 'median = %.2f MB/sec' % median
print ''

limit = max * 0.9
for r in result:
    if r[6] < limit:
        print '[Warning] connection between %s and %s is no good' % (r[0], r[1])
​```


```



gpcheckperf输出打印：

[gpadmin@ETL ~]$ gpcheckperf -f seg_host -d /export/gpdata –v

/apps/greenplum/bin/gpcheckperf -f seg_host -d /export/gpdata –v

--------------------
#### --  DISK WRITE TEST

--------------------
#### --  DISK READ TEST

#### --  STREAM TEST

#### --  NETPERF TEST
XXXXXXXXXXXXX

====================

#### ==  RESULT

 disk write avg time (sec): 1045.10
 disk write tot bytes: 1622627450880
 disk write tot bandwidth (MB/s): 2224.48
 disk write min bandwidth (MB/s): 179.06 [gsdw2]
 disk write max bandwidth (MB/s): 928.53 [gsdw4]



 disk read avg time (sec): 968.36
 disk read tot bytes: 1622627450880
 disk read tot bandwidth (MB/s): 3126.63
 disk read min bandwidth (MB/s): 182.02 [gsdw2]
 disk read max bandwidth (MB/s): 1776.85 [gsdw4]



 stream tot bandwidth (MB/s): 49777.98
 stream min bandwidth (MB/s): 6416.68 [gsdw5]
 stream max bandwidth (MB/s): 10046.99 [ gmdw]

Netperf bisection bandwidth test（Netperf二分带宽测试）
gmdw -> gsdw1 = 112.290000
gsdw2 -> gsdw3 = 112.330000
gsdw4 -> gsdw5 = 112.270000
gsdw1 -> gmdw = 112.310000
gsdw3 -> gsdw2 = 112.270000
gsdw5 -> gsdw4 = 112.290000

Summary:
sum = 673.76 MB/sec
min = 112.27 MB/sec
max = 112.33 MB/sec
avg = 112.29 MB/sec

median = 112.29 MB/sec



# 测网络原理

该工具运行一种网络基准测试程序，该程序当前主机发送5秒钟的数据流到测试中包含的每台远程主机。数据被并行传输到每台远程主机，并以兆字节（MB）每秒报告最小、最大、平均和中位网络传输速率。

相关参数：

--netperf

指定应该用netperf二进制文件来执行网络测试，而不是Greenplum网络测试。用户可以指定可选的--netperf选项来使用netperf二进制文件而不是默认的gpnetbench*工具。

--duration time

以秒（s）、分钟（m）、小时（h）或天数（d）指定网络测试的持续时间。默认值是15秒。



-r ds{n|N|M}

d磁盘IO

s流复制测试

n|N|M网络串行、并行、全矩阵测试

指定测试指标，不指定，默认为dsN。



# gpnetbenchServer/gpnetbenchClient测网络：

[gpadmin@node1 ~]$ gpcheckperf -f hostfile_exkeys -d /home/gpadmin -v
[Info] sh -c 'cat /proc/meminfo | grep MemTotal'
MemTotal:         995892 kB

## /usr/local/mpp1-db/bin/gpcheckperf -f hostfile_exkeys -d /home/gpadmin -v

  SETUP 2019-05-20T19:27:57.524702

[Info] verify python interpreter exists
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'python -c print'
[Info] making gpcheckperf directory on all hosts ...
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'rm -rf  /home/gpadmin/gpcheckperf_$USER ; mkdir -p  /home/gpadmin/gpcheckperf_$USER'
[Info] copy local /usr/local/mpp1-db/bin/lib/multidd to remote /home/gpadmin/gpcheckperf_$USER/multidd
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/multidd =:/home/gpadmin/gpcheckperf_$USER/multidd
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/multidd'

------

## --  DISK WRITE TEST

[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'time -p /home/gpadmin/gpcheckperf_$USER/multidd -i /dev/zero -o /home/gpadmin/gpcheckperf_$USER/ddfile -B 32768'

Message from syslogd@node1 at May 20 19:29:01 ...
 kernel:NMI watchdog: BUG: soft lockup - CPU#0 stuck for 24s! [dd:8103]

------

## --  DISK READ TEST

[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'time -p /home/gpadmin/gpcheckperf_$USER/multidd -o /dev/null -i /home/gpadmin/gpcheckperf_$USER/ddfile -B 32768'

------

## --  STREAM TEST

[Info] copy local /usr/local/mpp1-db/bin/lib/stream to remote /home/gpadmin/gpcheckperf_$USER/stream
[Info] /usr/local/mpp1-db/bin/gpscp -f hostfile_exkeys /usr/local/mpp1-db/bin/lib/stream =:/home/gpadmin/gpcheckperf_$USER/stream
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'chmod a+rx /home/gpadmin/gpcheckperf_$USER/stream'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys /home/gpadmin/gpcheckperf_$USER/stream

------

## --  NETPERF TEST

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

## [Info] node1 -> node3 : ['0', '0', '32768', '14.60', '287.01']

##   TEARDOWN

[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'rm -rf  /home/gpadmin/gpcheckperf_$USER'
[Info] /usr/local/mpp1-db/bin/gpssh -f hostfile_exkeys 'F=gpnetbenchServer && (pkill $F || pkill -f $F || killall -9 $F) > /dev/null 2>&1 || true'

====================

# ==  RESULT 2019-05-20T19:34:00.126606

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

# netperf(第三方插件)测网络性能

要求2个部分。netperf、netserver

netserver 服务端。server端是netserver，用来侦听来自client端的连接，client端是netperf，用来向server发起网络测试。

#### netperf命令参数

-H 指定连接的机器IP/hostname

-p 指定netperf的端口

-t 指定要执行的测试  TCP_STREAM/UDP_STREAM

-l 指定测试时长（sec=4）

-f 指定输出格式

-P 指定是否显示测试HEADER

​    cmd = ('%s -H %s -p %d -t TCP_STREAM -l %s -f M -P 0 '

​           % (netperf_path, y, netserver_port, sec))



#### 测试步骤：

1、先在对端开启服务进程

netserver(可不接参数)

2、在本地发起请求

netperf -H +对端IP/hostname



#### 默认输出如下：

MIGRATED TCP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to node2 () port 0 AF_INET
Recv   Send    Send
Socket Socket  Message  Elapsed
Size   Size    Size     Time     Throughput（吞吐量）
bytes  bytes   bytes    secs.    10^6bits/sec

 87380  16384  16384    10.04    2519.97

可看到吞吐量为2519.97Mbit/s，历时10.04s。

其中Revc代表远端系统（即server）使用大小为87380字节的socket接收缓冲；

Send代表本地系统（即client）使用大小为16384字节的socket发送缓冲；

```
向远端系统发送的测试分组大小为16384字节；
```

#### 参考：

<https://blog.csdn.net/chdhust/article/details/50490721>

<https://www.jianshu.com/p/42e0fa6bf79c>

# 测磁盘I/O

#### 磁盘写测试：

使用multidd从/dev/zero读取gpcheckperf指定总体大小（-S）、块大小（-B）的数据到-d指定的目录下	(dir/gpcheckperf_$USER/ddfile文件中)。

#### 磁盘读测试：

使用multidd从磁盘写的ddfile文件里按照gpcheckperf启动时指定的参数总体大小（-S）、块大小（-B）的数据读取到/dev/null。在命令之前加time记录执行时间。

如果不指定-S、-B 时，默认按照块大小为32KB、 2 * 物理内存的文件，拷贝到指定位置，测试磁盘读写速度。

#### 相关参数：

-B size 在测磁盘I/O时指定每个块的大小，默认32KB。

-S size 在测磁盘I/O时指定复制文件的大小，默认为物理内存的2倍大小。

-d segdir 指定测磁盘性能时数据存放目录。通常是Primary/Mirror数据目录，因此可以指定多个位置。



#### gpcheckperf获取当前机器物理内存方式：

sh -c 'cat /proc/meminfo | grep MemTotal'



#### multidd简介：

通过sh -c 'cat /proc/meminfo | grep MemTotal'获取当前机器物理内存大小

 内部实现是对dd命令的封装：

cmd.append('dd if=%s of=%s count=%d bs=%d' % (ifile, ofile, cnt, blocksz))





# 测流复制：

stream程序测试。