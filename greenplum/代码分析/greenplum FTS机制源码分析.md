# Greenplum FTS介绍

FTS(Fault Tolerance Serve)是GreenPlum中的故障检测服务，是保证GP高可用的核心功能。GreenPlum的Segment的健康检测及HA是由GP Master实现的，GP Master上面有个专门的进程–FTS进程，它可以快速检测到Primary或者Mirror是否挂掉，并及时作出Primary/Mirror 故障切换。

### 一、FTS架构

GreenPlum的Segment的健康检测及HA是由GP Master实现的，GP Master上面有个专门的进程–FTS进程，它可以快速检测到Primary或者Mirror是否挂掉，并及时作出Primary/Mirror 故障切换。如果FTS挂掉了，Master将会重新fork出来一个FTS进程。
![Greenplum--FTS故障检测原理](https://s1.51cto.com/images/blog/201901/18/8e81c7cceccc5a8a95accf924b2e619b.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 二、FTS实现原理

GP Master上面的FTS进程每隔60s(时间可以配置)向Primary或者Mirror发送心跳包，Primary和Mirror收到心跳包后返回它们的当前状态，FTS进程根据Segment返回状态更新元信息和作出故障切换。因为Segment可能很多，为了加快检测速度，FTS是多线程的，默认16个线程。
![Greenplum--FTS故障检测原理](https://s1.51cto.com/images/blog/201901/18/69c993509b2736a61129ad179ea517f7.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 三、Segment检测及故障切换

GP Master首先会检测Primary状态，如果Primary不可连通，那么将会检测Mirror状态，Primary/Mirror状态总共有4种：
Primary活着，Mirror活着。GP Master探测Primary成功之后直接返回，进行下一个Segment检测；
Primary活着，Mirror挂了。GP Master探测Primary成功之后，通过Primary返回的状态得知Mirror挂掉了（Mirror挂掉之后，Primary将会探测到，将自己变成ChangeTracking模式），这时候更新Master元信息，进行下一个Segment检测；
Primary挂了，Mirror活着。GP Master探测Primary失败之后探测Mirror，发现Mirror是活着，这时候更新Master上面的元信息，同时使Mirror接管Primary（故障切换），进行下一个Segment检测；
Primary挂了，Mirror挂了。GP Master探测Primary失败之后探测Mirror，Mirror也是挂了，直到重试最大值（默认配置为5次），结束这个Segment的探测，也不更新Master元信息了，进行下一个Segment检测。
![Greenplum--FTS故障检测原理](https://s1.51cto.com/images/blog/201901/18/3dd7cc698eda0258632fd8b5dac8bc24.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 四、参数配置

#### 4.1、gp_fts_probe_threadcount

用来故障检测的线程数量，默认为16。

#### 4.2、gp_fts_probe_interval

两次检测的时间间隔，默认为60s。如果一次检测时间使用10s，那么剩余50s将会sleep;如果超过60s，将会直接进入下一次检测。

#### 4.3、gp_fts_probe_timeout  

检测Segment超时时间，默认值: 20。

#### 4.4、gp_fts_probe_retries

检测Segment失败重试次数，如果超过这个次数，将会认为当前节点挂掉，默认值: 5。

#### 4.5、gp_segment_connect_timeout

Prmary和Mirror文件同步允许连接Mirror最大超时时间，如果达到这个超时时间，Primary将会认为Mirror挂掉了，默认值: 180s。





## 源码分析：

```
[gpadmin5@node1 ~]$ ps -ef|grep postgres
gpadmin5   7391      1  0 Jun08 ?        00:00:09 /usr/local/gp5/bin/postgres -D /opt/gp5/disk/master/gpseg-1 -p 5400 --gp_dbid=1 --gp_num_contents_in_cluster=2 --silent-mode=true -i -M master --gp_contentid=-1 -x 0 -E
gpadmin5   7392   7391  0 Jun08 ?        00:00:03 postgres:  5400, master logger process 
gpadmin5   7395   7391  0 Jun08 ?        00:00:01 postgres:  5400, stats collector process   
gpadmin5   7396   7391  0 Jun08 ?        00:00:09 postgres:  5400, writer process   
gpadmin5   7397   7391  0 Jun08 ?        00:00:02 postgres:  5400, checkpointer process   
gpadmin5   7398   7391  0 Jun08 ?        00:00:00 postgres:  5400, seqserver process   
gpadmin5   7399   7391  0 Jun08 ?        00:00:01 postgres:  5400, ftsprobe process   
gpadmin5   7401   7391  0 Jun08 ?        00:00:01 postgres:  5400, sweeper process   
gpadmin5   7402   7391  0 Jun08 ?        00:00:03 postgres:  5400, stats sender process   
gpadmin5   7404   7391  0 Jun08 ?        00:00:08 postgres:  5400, wal writer process   
gpadmin5  43516  43111  0 07:39 pts/0    00:00:00 grep --color=auto postgres
[gpadmin5@node1 ~]$ pstack 7399   
#0  0x00007fa19c65af73 in select () from /lib64/libc.so.6
#1  0x0000000000af894e in pg_usleep (microsec=<optimized out>) at pgsleep.c:43
#2  0x00000000006dd78d in FtsLoop () at fts.c:562
#3  ftsMain (argv=0x0, argc=0) at fts.c:379
#4  0x00000000006de235 in ftsprobe_start () at fts.c:164
#5  0x00000000007c2d1a in CommenceNormalOperations () at postmaster.c:4304
#6  0x00000000007c5465 in do_reaper () at postmaster.c:4678
#7  0x00000000007cb9bd in ServerLoop () at postmaster.c:2409
#8  0x00000000007cd2a2 in PostmasterMain (argc=argc@entry=15, argv=argv@entry=0x315e0e0) at postmaster.c:1519
#9  0x00000000004c7427 in main (argc=15, argv=0x315e0e0) at main.c:206
[gpadmin5@node1 ~]$ 
```

调用堆栈大致如上所示，由ftsprobe process进程负责故障检测，主备（Primary/Mirror）切换。

由ftsprobe_start开始跟踪，主要调用的是ftsMain(),ftsMain()调用FtsLoop();

FtsLoop()函数内部执行一个死循环，主要执行探测，标记，读取并更新元数据操作。



![1560095206220](C:\Users\cc\AppData\Roaming\Typora\typora-user-images\1560095206220.png)







```
​```static`
`void FtsLoop()`
`{`
	`bool	updated_bitmap, processing_fullscan;`
	`MemoryContext probeContext = NULL, oldContext = NULL;`
	`time_t elapsed,	probe_start_time;``
```

	probeContext = AllocSetContextCreate(TopMemoryContext,
										 "FtsProbeMemCtxt",
										 ALLOCSET_DEFAULT_INITSIZE,	/* always have some memory */
										 ALLOCSET_DEFAULT_INITSIZE,
										 ALLOCSET_DEFAULT_MAXSIZE);
	//读取元数据，并更新segment状态									 
	readCdbComponentInfoAndUpdateStatus(probeContext);
	for (;;)
	{
		if (shutdown_requested)
			break;
		/* no need to live on if postmaster has died */
		if (!PostmasterIsAlive(true))
			exit(1);
		if (got_SIGHUP)
		{
			got_SIGHUP = false;
			ProcessConfigFile(PGC_SIGHUP);
		}
		probe_start_time = time(NULL);
		ftsLock();
		/* atomically clear cancel flag and check pause flag */
		bool pauseProbes = ftsProbeInfo->fts_pauseProbes;
		ftsProbeInfo->fts_discardResults = false;
		ftsUnlock();
		if (pauseProbes)
		{
			if (gp_log_fts >= GPVARS_VERBOSITY_VERBOSE)
				elog(LOG, "skipping probe, we're paused.");
			goto prober_sleep;
		}
		if (cdb_component_dbs != NULL)
		{
			freeCdbComponentDatabases(cdb_component_dbs);
			cdb_component_dbs = NULL;
		}
		if (ftsProbeInfo->fts_probeScanRequested == ftsProbeInfo->fts_statusVersion)
			processing_fullscan = true;
		else
			processing_fullscan = false;
		readCdbComponentInfoAndUpdateStatus(probeContext);
		fault_strategy = get_gp_fault_strategy();
		if (fault_strategy == GpFaultStrategyMirrorLess)
		{
			/* The dispatcher could have requested a scan so just ignore it and unblock the dispatcher */
			if (processing_fullscan)
			{
				ftsProbeInfo->fts_statusVersion = ftsProbeInfo->fts_statusVersion + 1;
				rescan_requested = false;
			}
			goto prober_sleep;
		}
		elog(DEBUG3, "FTS: starting %s scan with %d segments and %d contents",
			 (processing_fullscan ? "full " : ""),
			 cdb_component_dbs->total_segment_dbs,
			 cdb_component_dbs->total_segments);
		/*
		 * We probe in a special context, some of the heap access
		 * stuff palloc()s internally
		 */
		oldContext = MemoryContextSwitchTo(probeContext);
		/* probe segments */
		FtsProbeSegments(cdb_component_dbs, scan_status);
		/*
		 * Now we've completed the scan, update shared-memory. if we
		 * change anything, we return true.
		 更新元数据
		 */
		updated_bitmap = probePublishUpdate(scan_status);


		MemoryContextSwitchTo(oldContext);
		/* free any pallocs we made inside probeSegments() */
		MemoryContextReset(probeContext);
		cdb_component_dbs = NULL;
		if (!FtsIsActive())
		{
			if (gp_log_fts >= GPVARS_VERBOSITY_VERBOSE)
				elog(LOG, "FTS: skipping probe, FTS is paused or shutting down.");
			goto prober_sleep;
		}
		/*
		 * If we're not processing a full-scan, but one has been requested; we start over.
		 */
		if (!processing_fullscan &&
			ftsProbeInfo->fts_probeScanRequested == ftsProbeInfo->fts_statusVersion)
			continue;
		/*
		 * bump the version (this also serves as an acknowledgement to
		 * a probe-request).
		 */
		if (updated_bitmap || processing_fullscan)
		{
			ftsProbeInfo->fts_statusVersion = ftsProbeInfo->fts_statusVersion + 1;
			rescan_requested = false;
		}
		/* if no full-scan has been requested, we can sleep. */
		if (ftsProbeInfo->fts_probeScanRequested >= ftsProbeInfo->fts_statusVersion)
		{
			/* we need to do a probe immediately */
			elog(LOG, "FTS: skipping sleep, requested version: %d, current version: %d.",
				 (int)ftsProbeInfo->fts_probeScanRequested, (int)ftsProbeInfo->fts_statusVersion);
			continue;
		}
	prober_sleep:
		{
			/* check if we need to sleep before starting next iteration */
			//每次循环睡眠，如果从开始探测到现在耗时不超过60s（默认值），则睡眠剩下的时间；否则，不睡眠
			elapsed = time(NULL) - probe_start_time;
			if (elapsed < gp_fts_probe_interval && !shutdown_requested)
			{
				pg_usleep((gp_fts_probe_interval - elapsed) * USECS_PER_SEC);
			}
		}
	} /* end server loop */
	return;
`}`



扫描一遍实例列表（读取gp_segment_configuration系统表），跳过标记为‘d’的实例，封装到scan_status数组中，之后开启16个线程，探测实例。

```
`void`
`FtsProbeSegments(CdbComponentDatabases *dbs, uint8 *probeRes)`
`{`
	`int i;`
```

	threadWorkerInfo worker_info;
	int workers = gp_fts_probe_threadcount;
	pthread_t *threads = NULL;
	cdb_component_dbs = dbs;
	scan_status = probeRes;
	if (cdb_component_dbs == NULL || scan_status == NULL)
	{
		elog(ERROR, "FTS: segment configuration has not been loaded to shared memory");
	}
	/* reset probe results */
	memset(scan_status, 0, cdb_component_dbs->total_segment_dbs * sizeof(scan_status[0]));
	/* figure out which segments to include in the scan. */
	进行标记，获取要探测的实例列表*/
	for (i=0; i < cdb_component_dbs->total_segment_dbs; i++)
	{
		CdbComponentDatabaseInfo *segInfo = &cdb_component_dbs->segment_db_info[i];
		if (FtsIsSegmentAlive(segInfo))
		{
			/* mark segment for probing */
			scan_status[segInfo->dbid] = PROBE_SEGMENT;
		}
		else
		{
			/* consider segment dead */
			scan_status[segInfo->dbid] = PROBE_DEAD;
		}
	}
	worker_info.scan_status = scan_status;
	threads = (pthread_t *)palloc(workers * sizeof(pthread_t));
		//默认开启16个线程去探测
	for (i = 0; i < workers; i++)
	{
#ifdef USE_ASSERT_CHECKING`
		`int ret =`
`#endif /* USE_ASSERT_CHECKING */`
		`gp_pthread_create(&threads[i], probeSegmentFromThread, &worker_info, "probeSegments");

		Assert(ret == 0 && "FTS: failed to create probing thread");
	}
	/* we have nothing left to do but wait */
	for (i = 0; i < workers; i++)
	{
```
#ifdef USE_ASSERT_CHECKING`
		`int ret =`
`#endif /* USE_ASSERT_CHECKING */`
		`pthread_join(threads[i], NULL);`
```

		Assert(ret == 0 && "FTS: failed to join probing thread");
	}
	pfree(threads);
	threads = NULL;
	/* if we're shutting down, just exit. */
	if (!FtsIsActive())
		return;
	if (gp_log_fts >= GPVARS_VERBOSITY_DEBUG)
	{
		elog(LOG, "FTS: probe results for all segments:");
		for (i=0; i < cdb_component_dbs->total_segment_dbs; i++)
		{
			CdbComponentDatabaseInfo *segInfo = NULL;
			segInfo = &cdb_component_dbs->segment_db_info[i];
			elog(LOG, "segment dbid %d status 0x%x.", segInfo->dbid, scan_status[segInfo->dbid]);
		}
	}
`}`



## //线程调用方法，交互式地选取每个Primary/Mirror对。探测Primary，Primary宕掉探测Mirror

可分为以下几种情况：

只有当Primary挂掉或者网络断掉才会检测Mirror

/* probe mirror only if primary is dead or has a crash/network fault */

处理逻辑，Primary 正常，跳过Mirror检测；

Primary异常、检测Mirror



`static void *`
`probeSegmentFromThread(void *arg)`
`{`
	`threadWorkerInfo *worker_info;`
	`int i;`

	worker_info = (threadWorkerInfo *) arg;
	i = 0;
	for (;;)
	{
		CdbComponentDatabaseInfo *primary = NULL;
		CdbComponentDatabaseInfo *mirror = NULL;
		char probe_result_primary = PROBE_DEAD;
		char probe_result_mirror = PROBE_DEAD;
		/*
		 * find untested primary, mark primary and mirror "tested" and unlock.
		 */
		pthread_mutex_lock(&worker_thread_mutex);
		for (; i < cdb_component_dbs->total_segment_dbs; i++)
		{
			primary = &cdb_component_dbs->segment_db_info[i];
			/* check segments in pairs of primary-mirror */
			if (!SEGMENT_IS_ACTIVE_PRIMARY(primary))
			{
				continue;
			}
			if (PROBE_CHECK_FLAG(worker_info->scan_status[primary->dbid], PROBE_SEGMENT))
			{
				/* prevent re-checking this pair */
				worker_info->scan_status[primary->dbid] &= ~PROBE_SEGMENT;
				mirror = FtsGetPeerSegment(primary->segindex, primary->dbid);
				/* check if mirror is marked for probing */
				if (mirror != NULL &&
					PROBE_CHECK_FLAG(worker_info->scan_status[mirror->dbid], PROBE_SEGMENT))
				{
					worker_info->scan_status[mirror->dbid] &= ~PROBE_SEGMENT;
				}
				else
				{
					mirror = NULL;
				}
				break;
			}
		}
		pthread_mutex_unlock(&worker_thread_mutex);
		/* check if all segments were probed */
		if (i == cdb_component_dbs->total_segment_dbs || primary == NULL)
		{
			break;
		}
		/* if we've gotten a pause or shutdown request, we ignore probe results. */
		if (!FtsIsActive())
		{
			break;
		}
		/* probe primary */
		probe_result_primary = probeSegment(primary);
		Assert(!PROBE_CHECK_FLAG(probe_result_primary, PROBE_SEGMENT));
		if ((probe_result_primary & PROBE_ALIVE) == 0 && gp_log_fts >= GPVARS_VERBOSITY_VERBOSE)
		{
			write_log("FTS: primary (dbid=%d, content=%d, status 0x%x) didn't respond to probe.",
			          primary->dbid, primary->segindex, probe_result_primary);
		}
		if (mirror != NULL)
		{
			/* assume mirror is alive */
			probe_result_mirror = PROBE_ALIVE;
			/* probe mirror only if primary is dead or has a crash/network fault */
			if (!PROBE_CHECK_FLAG(probe_result_primary, PROBE_ALIVE) ||
				PROBE_CHECK_FLAG(probe_result_primary, PROBE_FAULT_CRASH) ||
				PROBE_CHECK_FLAG(probe_result_primary, PROBE_FAULT_NET))
			{
				/* probe mirror */
				probe_result_mirror = probeSegment(mirror);
				Assert(!PROBE_CHECK_FLAG(probe_result_mirror, PROBE_SEGMENT));
				if ((probe_result_mirror & PROBE_ALIVE) == 0 && gp_log_fts >= GPVARS_VERBOSITY_VERBOSE)
				{
					write_log("FTS: mirror (dbid=%d, content=%d, status 0x%x) didn't respond to probe.",
					          mirror->dbid, mirror->segindex, probe_result_mirror);
				}
			}
		}
		/* update results */
		pthread_mutex_lock(&worker_thread_mutex);
		worker_info->scan_status[primary->dbid] = probe_result_primary;
		if (mirror != NULL)
		{
			worker_info->scan_status[mirror->dbid] = probe_result_mirror;
		}
		pthread_mutex_unlock(&worker_thread_mutex);
	}
	return NULL;
`}`

### 实际的探测函数（逻辑如上所述，这里对primary建立套接字、发送探测消息、接收返回结果）

/*
 * This is called from several different threads: ONLY USE THREADSAFE FUNCTIONS INSIDE.
 */
static char
probeSegment(CdbComponentDatabaseInfo *dbInfo)
{
	Assert(dbInfo != NULL);

	/* setup probe descriptor */
	ProbeConnectionInfo probeInfo;
	memset(&probeInfo, 0, sizeof(ProbeConnectionInfo));
	probeInfo.segmentId = dbInfo->segindex;
	probeInfo.dbId = dbInfo->dbid;
	probeInfo.role = dbInfo->role;
	probeInfo.mode = dbInfo->mode;
	probeInfo.hostIp = dbInfo->hostip;
	probeInfo.port = dbInfo->port;
	probeInfo.segmentStatus = PROBE_DEAD;

	/* set probe start timestamp */
	gp_set_monotonic_begin_time(&probeInfo.startTime);
	int retryCnt = 1;

	/*
	 * probe segment: open socket -> connect -> send probe msg -> receive response;
	 * on any error the connection is shut down, the socket is closed
	 * and the probe process is restarted;
	 * this is repeated until no error occurs (response is received) or timeout expires;
	 */
	while (!probeGetIpAddr(&probeInfo) ||
	       !probeOpenSocket(&probeInfo) ||
	       !probeMarkSocketNonBlocking(&probeInfo) ||
	       !probeConnect(&probeInfo) ||
	       !probeSend(&probeInfo) ||
	       !probeReceive(&probeInfo) ||
	       !probeProcessResponse(&probeInfo))
	{
		probeClose(&probeInfo);

		Assert(probeInfo.segmentStatus == PROBE_DEAD);

		/* check if FTS is active */
		if (!FtsIsActive())
		{
			/* the returned value will be ignored */
			probeInfo.segmentStatus = PROBE_ALIVE;
			break;
		}

		/*
		 * if maximum number of retries was reached,
		 * report segment as non-responsive (dead)
		 */
		if (retryCnt == gp_fts_probe_retries)
		{
			write_log("FTS: failed to probe segment (content=%d, dbid=%d) after trying %d time(s), "
					  "maximum number of retries reached.",
					  probeInfo.segmentId,
					  probeInfo.dbId,
					  retryCnt);
			break;
		}

		/* sleep for 1 second to avoid tight loops */
		pg_usleep(USECS_PER_SEC);
		retryCnt++;

		write_log("FTS: retry %d to probe segment (content=%d, dbid=%d).",
				  retryCnt - 1, probeInfo.segmentId, probeInfo.dbId);

		/* reset timer */
		gp_set_monotonic_begin_time(&probeInfo.startTime);
	}

	/* segment response to probe was received, close connection  */
	probeClose(&probeInfo);

	return probeInfo.segmentStatus;
}



### 状态标记、元数据更新函数：

### ![1560182132448](C:\Users\cc\AppData\Roaming\Typora\typora-user-images\1560182132448.png)



更新gp_segment_configuration、gp_configuration_history

`/*`

 * `update segment configuration in catalog and shared memory`
 `*/`
`static bool`
`probeUpdateConfig(FtsSegmentStatusChange *changes, int changeCount)`
`{`
	`Relation configrel;`
	`Relation histrel;`
	`SysScanDesc sscan;`
	`ScanKeyData scankey;`
	`HeapTuple configtuple;`
	`HeapTuple newtuple;`
	`HeapTuple histtuple;`
	`Datum configvals[Natts_gp_segment_configuration];`
	`bool confignulls[Natts_gp_segment_configuration] = { false };`
	`bool repls[Natts_gp_segment_configuration] = { false };`
	`Datum histvals[Natts_gp_configuration_history];`
	`bool histnulls[Natts_gp_configuration_history] = { false };`
	`bool valid;`
	`bool primary;`
	`bool changelogging;`
	`int i;`
	`char desc[SQL_CMD_BUF_SIZE];`

`#ifdef USE_SEGWALREP`
	`Assert(fault_strategy == GpFaultStrategyWalRepMirrored);`
`#else`
	`Assert(fault_strategy == GpFaultStrategyFileRepMirrored);`
`#endif`

	/*
	 * Commit/abort transaction below will destroy
	 * CurrentResourceOwner.  We need it for catalog reads.
	 */
	ResourceOwner save = CurrentResourceOwner;
	StartTransactionCommand();
	GetTransactionSnapshot();
	elog(LOG, "probeUpdateConfig called for %d changes", changeCount);
	histrel = heap_open(GpConfigHistoryRelationId,
						RowExclusiveLock);
	configrel = heap_open(GpSegmentConfigRelationId,
						  RowExclusiveLock);
	for (i = 0; i < changeCount; i++)
	{
		FtsSegmentStatusChange *change = &changes[i];
		valid   = (changes[i].newStatus & FTS_STATUS_ALIVE);
		primary = (changes[i].newStatus & FTS_STATUS_PRIMARY);
		changelogging = (changes[i].newStatus & FTS_STATUS_CHANGELOGGING);
		if (changelogging)
		{
			Assert(primary && valid);
		}
		Assert((valid || !primary) && "Primary cannot be down");
		/*
		 * Insert new tuple into gp_configuration_history catalog.
		 */
		histvals[Anum_gp_configuration_history_time-1] =
				TimestampTzGetDatum(GetCurrentTimestamp());
		histvals[Anum_gp_configuration_history_dbid-1] =
				Int16GetDatum(changes[i].dbid);
		snprintf(desc, sizeof(desc),
				 "FTS: content %d fault marking status %s%s role %c",
				 change->segindex, valid ? "UP" : "DOWN",
				 (changelogging) ? " mode: change-tracking" : "",
				 primary ? 'p' : 'm');
		histvals[Anum_gp_configuration_history_desc-1] =
					CStringGetTextDatum(desc);
		histtuple = heap_form_tuple(RelationGetDescr(histrel), histvals, histnulls);
		simple_heap_insert(histrel, histtuple);
		CatalogUpdateIndexes(histrel, histtuple);
		/*
		 * Find and update gp_segment_configuration tuple.
		 */
		ScanKeyInit(&scankey,
					Anum_gp_segment_configuration_dbid,
					BTEqualStrategyNumber, F_INT2EQ,
					Int16GetDatum(changes[i].dbid));
		sscan = systable_beginscan(configrel, GpSegmentConfigDbidIndexId,
								   true, SnapshotNow, 1, &scankey);
		configtuple = systable_getnext(sscan);
		if (!HeapTupleIsValid(configtuple))
		{
			elog(ERROR, "FTS cannot find dbid=%d in %s", changes[i].dbid,
				 RelationGetRelationName(configrel));
		}
		configvals[Anum_gp_segment_configuration_role-1] =
				CharGetDatum(primary ? 'p' : 'm');
		repls[Anum_gp_segment_configuration_role-1] = true;
		configvals[Anum_gp_segment_configuration_status-1] =
				CharGetDatum(valid ? 'u' : 'd');
		repls[Anum_gp_segment_configuration_status-1] = true;
		if (changelogging)
		{
			configvals[Anum_gp_segment_configuration_mode-1] =
					CharGetDatum('c');
		}
		repls[Anum_gp_segment_configuration_mode-1] = changelogging;
	
		newtuple = heap_modify_tuple(configtuple, RelationGetDescr(configrel),
									 configvals, confignulls, repls);
		simple_heap_update(configrel, &configtuple->t_self, newtuple);
		CatalogUpdateIndexes(configrel, newtuple);
	
		systable_endscan(sscan);
		pfree(newtuple);
		/*
		 * Update shared memory
		 */
		ftsProbeInfo->fts_status[changes[i].dbid] = changes[i].newStatus;
	}
	heap_close(histrel, RowExclusiveLock);
	heap_close(configrel, RowExclusiveLock);
	
	SIMPLE_FAULT_INJECTOR(FtsWaitForShutdown);
	/*
	 * Do not block shutdown.  We will always get a change to update
	 * gp_segment_configuration in subsequent probes upon database
	 * restart.
	 */
	if (shutdown_requested)
	{
		elog(LOG, "Shutdown in progress, ignoring FTS prober updates.");
		return false;
	}
	CommitTransactionCommand();
	CurrentResourceOwner = save;
	return true;
`}`



参考链接：

<https://blog.51cto.com/13126942/2344330>