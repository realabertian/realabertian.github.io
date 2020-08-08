---
layout: post
title:  "Glide研究2: Engine状态流程"
date:   2020-08-08 18:02:53 +0800
categories: Glide
---
### DecodeJob的基本状态

DecodeJob基于两个状态值来确定其当前应该执行的动作

- runReason  表明该DecodeJob因何而被执行
    - INITIALIZE 该标志意味着DecodeJob是第一次执行
    - SWITCH_TO_SOURCE_SERVICE 表示从DiskCache切换到Source
    - DECODEC_DATA 处理时发现需要的某些数据目前还没有,需要切换来获取这些数据
- Stage 表明decode数据的进度
    - INITIALIZE 初始化状态
    - RESOURCE_CACHE 从一个缓存的Resource中decode
    - DATA_CACHE 从缓存的原数据decode
    - SOURCE 从数据源提取
    - ENCODE 完成一次成功的加载后将Resource编码缓存
    - FINISHED 没有更多的Stage需要执行了

这两个状态变量的变化图是如何的呢?接下来我们开始分析源码来重构这个状态机
从上一片文章我们知道DecodeJob是一个Runnable, 它的执行开始于runWrapped方法,而
runReason的初始值有DecodecJob的init方法早构造DecodecJob时进行赋值其初始值为INITIALIZE
```java
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```
所以首次执行时,上述代码进入第一个case,这里获得Stage.INITIALIZE的下一个Stage
```java
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```
默认情况下diskCacheStrategy是DiskCacheStrategy.AUTOMATIC, 而decodeCachedResource返回true
所以这里getNextStage返回Stage.Resource_CACHE,接着执行getNextGenerator获取生成器
```java
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```
所以此处获得的生成器是ResourceCacheGenerator,然后调用runGenerators运行这个生成器
```java
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }

    // Otherwise a generator started a new load and we expect to be called back in
    // onDataFetcherReady.
  }
```
首先执行currentGenerator.startNext(),此方法内部将会具体执行加载资源,执行完成后会通过回调通知外部加成是否成功
这里完成后会将stage设置为下一个阶段, 本阶段Stage是RESOURCE_CACHE 下一个阶段是DATA_CACHE,注意这里因为我们是第一次加载
所以startNext方法一定加载失败,也就是返回false, 那么循环就会进入,执行DATA_CACHE阶段,这个逻辑就是在资源缓存中没有找到
那么就需要从DATA_CACHE阶段去找, 然后循环再次进入这次执行DataCacheGenerator, 而这个生成器的逻辑也是类似的,在缓存中查找DataKey是否存在如果不存在就返回false
这里肯定返回false,所以继续执行下一个生成器也就是SourceGenerator, 这里DATA_CACHE阶段失败了,进入下一个阶段SOURCE,SOURCE阶段就是实际从数据源获取数据了,获取完数据还要进行必要的缓存操作
```java
@Override
  public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
    return started;
  }
```
首次运行startNext, dataToCache和sourceCacheGenrator都没有值
就是直接进入while循环尝试进行startNextLoad方法加载数据
```java
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
  }
```
加载成功后调用这里的内部回调
```java
void onDataReadyInternal(LoadData<?> loadData, Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(
          loadData.sourceKey,
          data,
          loadData.fetcher,
          loadData.fetcher.getDataSource(),
          originalKey);
    }
  }
```
这里如果缓存策略货是期望连未修改的数据源也要缓存再次执行sourceGenertor的startNext,此时dataToCache就有值了.
如果不需要则直接通知外部开始更新流程了, 两个流程不管哪个最终就走onDataFetcherReady回调,进入这个回调后就是开始
进行解码饿阶段,也就是runReason的值开始变更为RunReason.DECODE_DATA
```java
public void onDataFetcherReady(
      Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    this.isLoadingFromAlternateCacheKey = sourceKey != decodeHelper.getCacheKeys().get(0);

    if (Thread.currentThread() != currentThread) {
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
    } else {
      GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        decodeFromRetrievedData();
      } finally {
        GlideTrace.endSection();
      }
    }
  }
```
这里不管是哪个条件语句执行最后都会进入decodeFromRetrievedData区别是从runWrapped再次调度进入
还是直接进入
```java
private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey(
          "Retrieved data",
          startFetchTime,
          "data: "
              + currentData
              + ", cache key: "
              + currentSourceKey
              + ", fetcher: "
              + currentFetcher);
    }
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource, isLoadingFromAlternateCacheKey);
    } else {
      runGenerators();
    }
  }
```
可以看看到, 数据由数据源开始编码成resource,完成后通知进行encode和相关资源的释放, 这里decodeFraomData具体的处理逻辑
我们在未来有机会在深入查看它的设计, 我们直接跳过来看看notifyEncodeAndRelease干了什么
```java
private void notifyEncodeAndRelease(
      Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
    if (resource instanceof Initializable) {
      ((Initializable) resource).initialize();
    }

    Resource<R> result = resource;
    LockedResource<R> lockedResource = null;
    if (deferredEncodeManager.hasResourceToEncode()) {
      lockedResource = LockedResource.obtain(resource);
      result = lockedResource;
    }

    notifyComplete(result, dataSource, isLoadedFromAlternateCacheKey);

    stage = Stage.ENCODE;
    try {
      if (deferredEncodeManager.hasResourceToEncode()) {
        deferredEncodeManager.encode(diskCacheProvider, options);
      }
    } finally {
      if (lockedResource != null) {
        lockedResource.unlock();
      }
    }
    // Call onEncodeComplete outside the finally block so that it's not called if the encode process
    // throws.
    onEncodeComplete();
  }
```
这里进过几个阶段,首先如果resource是可初始化的资源,先进行初始化,然后接着判断是否有资源在编码中,如果没有执行通知完成
然后使用encodeManager对资源进行编码并缓存,最后通知编码完成
notifyCompelte实际回调EngineJob的onResourceReady方法, 接着回调Engine的onEngineJobComplete方法将resource放入激活缓存中
然后使用UI线程执行期执行ResourceReady回调通知Target可以设置资源了,这样资源就显示在视图上了, 当UI该激活资源被移除不实用后,会触发
onResourceRelease回调将它移动到内存缓存中, 这里利用了WeakReference的回收机制来通知onResourceRelease进行Resource的资源回收
到内存缓存中, 这里就不展开了