---
title: Flink原理-Container启动命令内存参数分析
date: 2019-06-08 19:22:29
tags:
toc: true
categories: Flink
---
如下是flink on yarn启动的一个示例:
```.
/bin/flink run -m yarn-cluster -yd -yjm 1024 -ys 2 -ytm 2048 clink-with-dependencies.jar  --topic_name topic1 --log_name --source_num 50  --sink_num 50
```
其中我们定义了JobManager的大小为1024M, TaskManager大小为2048M, 但是通过实际观察发现JobManager heap大小为424M, TaskManager heap memory大小为1304M, direct memory大小为744M , 参数值为啥是这个呢?
#  JobManager内存参数设置
可以看到, JM内存设置与实际严重不符, 我们直接上代码找下原因吧。
```
121390 YarnJobClusterEntrypoint -Xmx424m -Dlog.file=/opt/meituan/hadoop-flink/hadoop-2.7.1/logs/userlogs/application_1554212859057_0525/container_1554212859057_0525_01_000001/jobmanager.log -Dlogback.configurationFile=file:logback.xml -Dlog4j.configuration=file:log4j.properties
```
JM启动命令的产生是在AbstractYarnClusterDescriptor.startAppMaster()中, 最终进入了setupApplicationMasterContainer()中:
```
	protected ContainerLaunchContext setupApplicationMasterContainer(
			String yarnClusterEntrypoint, //获取的是org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint
			boolean hasLogback,
			boolean hasLog4j,
			boolean hasKrb5,
			int jobManagerMemoryMb) {
		// ------------------ Prepare Application Master Container  ------------------------------
		// respect custom JVM options in the YAML file
		String javaOpts = flinkConfiguration.getString(CoreOptions.FLINK_JVM_OPTIONS);
		if (flinkConfiguration.getString(CoreOptions.FLINK_JM_JVM_OPTIONS).length() > 0) {
			javaOpts += " " + flinkConfiguration.getString(CoreOptions.FLINK_JM_JVM_OPTIONS);
		}
		// Set up the container launch context for the application master
		ContainerLaunchContext amContainer = Records.newRecord(ContainerLaunchContext.class);
		final  Map<String, String> startCommandValues = new HashMap<>();
		startCommandValues.put("java", "$JAVA_HOME/bin/java");
		startCommandValues.put("jvmmem", "-Xmx" +
			Utils.calculateHeapSize(jobManagerMemoryMb, flinkConfiguration) +
			"m");
		startCommandValues.put("jvmopts", javaOpts);
		String logging = "";

       //构建启动命令，class指定了申请的container启动哪个主函数，也就是jm  applicationMaster的地址
		startCommandValues.put("logging", logging);
		startCommandValues.put("class", yarnClusterEntrypoint);   //获取的是org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint
		startCommandValues.put("redirects",
			"1> " + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/jobmanager.out " +
			"2> " + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/jobmanager.err");
		startCommandValues.put("args", "");

		final String commandTemplate = flinkConfiguration  // jm 启动默认命令
			.getString(ConfigConstants.YARN_CONTAINER_START_COMMAND_TEMPLATE,
				ConfigConstants.DEFAULT_YARN_CONTAINER_START_COMMAND_TEMPLATE);
		final String amCommand =
			BootstrapTools.getStartCommand(commandTemplate, startCommandValues); //获取真正的 jm  启动命令

		// 真正去通过yarn调度启动JM
		amContainer.setCommands(Collections.singletonList(amCommand));
		return amContainer;
	}
```
可以看到, JM启动默认只设置了-Xmx, 在Utils.calculateHeapSize(jobManagerMemoryMb)中设置了,
```
	public static int calculateHeapSize(int memory, org.apache.flink.configuration.Configuration conf) {
        // container占用比率,containerized.heap-cutoff-ratio, 默认25%
		float memoryCutoffRatio = conf.getFloat(ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_RATIO);
		// container占用最小内存大小: containerized.heap-cutoff-min, 最少600mb
		int minCutoff = conf.getInteger(ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_MIN);
		if (memoryCutoffRatio > 1 || memoryCutoffRatio < 0) {
			throw new IllegalArgumentException("The configuration value '"
				+ ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_RATIO.key()
				+ "' must be between 0 and 1. Value given=" + memoryCutoffRatio);
		}
		if (minCutoff > memory) {
			throw new IllegalArgumentException("The configuration value '"
				+ ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_MIN.key()
				+ "' is higher (" + minCutoff + ") than the requested amount of memory " + memory);
		}
		int heapLimit = (int) ((float) memory * memoryCutoffRatio);
		if (heapLimit < minCutoff) {
			heapLimit = minCutoff;
		}
		return memory - heapLimit;
	}
```
可以看到, JM启动时, 去掉yarn container占用内存大小, 即为JM大小。 container大小为20%*heap, 但是不得小于600MB,  这里当然只能600M, 所以剩余JM heap大小为(1024-600)MB=424MB。

#  TaskManager内存参数设置
如下是TM启动后的参数, 可以发现, 堆内内存+对外内存=申请的内存, 那么这个内存如何确定比例的?
```
183863 YarnTaskExecutorRunner -Xms1304m -Xmx1304m -XX:MaxDirectMemorySize=744m -Dlog.file=/opt/meituan/hadoop-flink/hadoop-2.7.1/logs/userlogs/application_1554212859057_0525/container_1554212859057_0525_01_000003/taskmanager.log -Dlogback.configurationFile=file:./logback.xml -Dlog4j.configuration=file:./log4j.properties
```
当JM为TM申请container成功后, 在<a href="https://kkewwei.github.io/elasticsearch_learning/2019/03/12/Flink%E5%8E%9F%E7%90%86-slot%E5%88%86%E9%85%8D/#%E7%94%B3%E8%AF%B7container%E6%88%90%E5%8A%9F">createTaskExecutorLaunchContext</a>中确定JM启动参数, 最终进入到了ContaineredTaskManagerParameters.create中:
```
	public static ContaineredTaskManagerParameters create(
			Configuration config,
			long containerMemoryMB,
			int numSlots) {
		// (1) try to compute how much memory used by container
		// 首先确定yarn Container使用多少内存(算是堆外内存)
		final long cutoffMB = calculateCutoffMB(config, containerMemoryMB);

		// (2) split the remaining Java memory between heap and off-heap
		//确定堆内内存大小
		final long heapSizeMB = TaskManagerServices.calculateHeapSizeMB(containerMemoryMB - cutoffMB, config);
		// use the cut-off memory for off-heap (that was its intention)
		// 堆内内存以外的全是堆内内存
		final long offHeapSizeMB = containerMemoryMB - heapSizeMB;

		// (3) obtain the additional environment variables from the configuration
		final HashMap<String, String> envVars = new HashMap<>();
		final String prefix = ResourceManagerOptions.CONTAINERIZED_TASK_MANAGER_ENV_PREFIX;

		for (String key : config.keySet()) {
			if (key.startsWith(prefix) && key.length() > prefix.length()) {
				// remove prefix
				String envVarKey = key.substring(prefix.length());
				envVars.put(envVarKey, config.getString(key, null));
			}
		}

		// done
		return new ContaineredTaskManagerParameters(
			containerMemoryMB, heapSizeMB, offHeapSizeMB, numSlots, envVars);
	}
```
该函数主要做了如下逻辑:
1. 确定yarn container本身需要的内存(算是堆外内存)。
2. 确定堆内的内存大小。
3. 确定堆外内存大小, 不是堆内的内存就全部算是堆外内存。
首先我们看下如何确定yarn container大小的:
```
	public static long calculateCutoffMB(Configuration config, long containerMemoryMB) {
		Preconditions.checkArgument(containerMemoryMB > 0);
		// (1) check cutoff ratio  // 默认值0.25f
		final float memoryCutoffRatio = config.getFloat(
			ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_RATIO);
		// (2) check min cutoff value   最少预留大小默认600MB
		final int minCutoff = config.getInteger(
			ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_MIN);
		// (3) check between heap and off-heap
		long cutoff = (long) (containerMemoryMB * memoryCutoffRatio);
		//取最大值
		if (cutoff < minCutoff) {
			cutoff = minCutoff;
		}
		return cutoff;  //至少预留600MB，
	}
```
yarn container大小取值原则: 在申请内存的containerized.heap-cutoff-ratio(默认25%)与containerized.heap-cutoff-min(默认600M)之前取最大值。本示例中, container占用600M
接下来我们看下如何确定堆内内存的:
```
	public static long calculateHeapSizeMB(long totalJavaMemorySizeMB, Configuration config) {
		// subtract the Java memory used for network buffers (always off-heap)
		//  网络buffer, 最小64M，最大1GB, 是totalJavaMemorySizeMB*0.1
		final long networkBufMB =
			calculateNetworkBufferMemory(
				totalJavaMemorySizeMB << 20, // megabytes to bytes
				config) >> 20; // bytes to megabytes
		final long remainingJavaMemorySizeMB = totalJavaMemorySizeMB - networkBufMB;
		// split the available Java memory between heap and off-heap
		final boolean useOffHeap = config.getBoolean(TaskManagerOptions.MEMORY_OFF_HEAP);  // 默认为false

		final long heapSizeMB;
		if (useOffHeap) {// 默认线上是关闭的
			long offHeapSize;
			String managedMemorySizeDefaultVal = TaskManagerOptions.MANAGED_MEMORY_SIZE.defaultValue();
			if (!config.getString(TaskManagerOptions.MANAGED_MEMORY_SIZE).equals(managedMemorySizeDefaultVal)) {
				try {// 将划去networkBuffer大小*一个堆外的系数（默认是0.7）得到其他的堆外内存
					offHeapSize = MemorySize.parse(config.getString(TaskManagerOptions.MANAGED_MEMORY_SIZE), MEGA_BYTES).getMebiBytes();
				}
			} else {
				offHeapSize = Long.valueOf(managedMemorySizeDefaultVal);
			}

			if (offHeapSize <= 0) {
				// calculate off-heap section via fraction
				double fraction = config.getFloat(TaskManagerOptions.MANAGED_MEMORY_FRACTION);
				offHeapSize = (long) (fraction * remainingJavaMemorySizeMB);
			}

			TaskManagerServicesConfiguration
				.checkConfigParameter(offHeapSize < remainingJavaMemorySizeMB, offHeapSize,
					TaskManagerOptions.MANAGED_MEMORY_SIZE.key(),
					"Managed memory size too large for " + networkBufMB +
						" MB network buffer memory and a total of " + totalJavaMemorySizeMB +
						" MB JVM memory");
			heapSizeMB = remainingJavaMemorySizeMB - offHeapSize;
		} else {
			heapSizeMB = remainingJavaMemorySizeMB;
		}

		return heapSizeMB;
	}
```
内存大小确定过程:
1. 首先确定网络buffer大小: 是totalJavaMemorySizeMB*0.1, 而且最小64M，最大1GB。 这里是(2048-600)*0.1MB=144.8M。
2. 剩余内存大小为(2048-600)*0.9MB=1304MB。
3. 在我们代码中, useOffHeap是false, 由taskmanager.memory.off-heap设置, 默认为false。
那么, TM启动的参数就很明显了, HEAP为1304MB, OFF-HEAD为2048MB-1304MB=744MB(包括Container占用空闲+网络buffer)。

