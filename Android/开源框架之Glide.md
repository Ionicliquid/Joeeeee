# with：生命周期管理

根据传入的`Context`不同，针对`RequestManager`有三种不同的管理策略：

1. 非UI线程或者传入的`Context`为`Application`,`RequestManager`生命周期保持和应用一致；
2. `Fragment`或`Activity`为`androidx`中的子类；
3. `Fragment`或`Activity`为`android.app`中的子类；

``` java
public RequestManager get(@NonNull Context context) {  
  if (context == null) {  
    throw new IllegalArgumentException("You cannot start a load on a null Context");  
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {  
    if (context instanceof FragmentActivity) {  
      return get((FragmentActivity) context);  
  } else if (context instanceof ContextWrapper  
        // Only unwrap a ContextWrapper if the baseContext has a non-null application context.  
 // Context#createPackageContext may return a Context without an Application instance, // in which case a ContextWrapper may be used to attach one.  && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {  
      return get(((ContextWrapper) context).getBaseContext());  
  }  
  }  
  
  return getApplicationManager(context);  
}
```

#### Application 或者 子线程

``` java
private RequestManager getApplicationManager(@NonNull Context context) {  
  // Either an application context or we're on a background thread.  
  if (applicationManager == null) {  
    synchronized (this) {  
      if (applicationManager == null) {  
        // Normally pause/resume is taken care of by the fragment we add to the fragment or  
 // activity. However, in this case since the manager attached to the application will not // receive lifecycle events, we must force the manager to start resumed using // ApplicationLifecycle.  
 // TODO(b/27524013): Factor out this Glide.get() call.  
  Glide glide = Glide.get(context.getApplicationContext());  
  applicationManager =  
            factory.build(  
                glide,  
 new ApplicationLifecycle(),  
 new EmptyRequestManagerTreeNode(),  
  context.getApplicationContext());  
  }  
    }  
  }  
  
  return applicationManager;  
}
class ApplicationLifecycle implements Lifecycle {  
  @Override  
  public void addListener(@NonNull LifecycleListener listener) {  
    listener.onStart();  
  }  
  
  @Override  
  public void removeListener(@NonNull LifecycleListener listener) {  
    // Do nothing.  
  }  
}
```

与应用的生命周期一致，只需要在生命周期onStart即可，其他的生命周期管理交给应用完成；

#### FragmentActivity/Fragment

``` java
public RequestManager get(@NonNull FragmentActivity activity) {  
  if (Util.isOnBackgroundThread()) {  
    return get(activity.getApplicationContext());  
  }  
  assertNotDestroyed(activity);  
  frameWaiter.registerSelf(activity);  
 boolean isActivityVisible = isActivityVisible(activity);  
  Glide glide = Glide.get(activity.getApplicationContext());  
 return lifecycleRequestManagerRetriever.getOrCreate(  
      activity,  
  glide,  
  activity.getLifecycle(),  
  activity.getSupportFragmentManager(),  
  isActivityVisible);  
}
@NonNull  
public RequestManager get(@NonNull Fragment fragment) {  
  Preconditions.checkNotNull(  
      fragment.getContext(),  
  "You cannot start a load on a fragment before it is attached or after it is destroyed");  
 if (Util.isOnBackgroundThread()) {  
    return get(fragment.getContext().getApplicationContext());  
  }  
  // In some unusual cases, it's possible to have a Fragment not hosted by an activity. There's  
 // not all that much we can do here. Most apps will be started with a standard activity. If // we manage not to register the first frame waiter for a while, the consequences are not // catastrophic, we'll just use some extra memory.  if (fragment.getActivity() != null) {  
    frameWaiter.registerSelf(fragment.getActivity());  
  }  
  FragmentManager fm = fragment.getChildFragmentManager();  
  Context context = fragment.getContext();  
  Glide glide = Glide.get(context.getApplicationContext());  
 return lifecycleRequestManagerRetriever.getOrCreate(  
      context, glide, fragment.getLifecycle(), fm, fragment.isVisible());  
}
RequestManager getOrCreate(  
    Context context,  
  Glide glide,  
 final Lifecycle lifecycle,  
  FragmentManager childFragmentManager,  
 boolean isParentVisible) {  
  Util.assertMainThread();  
  RequestManager result = getOnly(lifecycle);  
 if (result == null) {  
    LifecycleLifecycle glideLifecycle = new LifecycleLifecycle(lifecycle);  
  result =  
        factory.build(  
            glide,  
  glideLifecycle,  
 new SupportRequestManagerTreeNode(childFragmentManager),  
  context);  
  lifecycleToRequestManager.put(lifecycle, result);  
  glideLifecycle.addListener(  
        new LifecycleListener() {  
          @Override  
  public void onStart() {}  
  
          @Override  
  public void onStop() {}  
  
          @Override  
  public void onDestroy() {  
            lifecycleToRequestManager.remove(lifecycle);  
  }  
        });  
  // This is a bit of hack, we're going to start the RequestManager, but not the  
 // corresponding Lifecycle. It's safe to start the RequestManager, but starting the // Lifecycle might trigger memory leaks. See b/154405040  if (isParentVisible) {  
      result.onStart();  
  }  
  }  
  return result;  
}
```

1. 将RequestManager 保存到LifecycleRequestManager中，同一个Context只创建一次，保证同一个上下文拥有相同的RequestManager对象；
2. Context 已经实现LifecycleOwner对象，直接将RequestManager加入到Context对应的lifecycle观察者集合中，如果当前activity/fragment可见，调用onStart方法；

#### android.app.Fragment/Activity

这个方法目前已经废弃，我们可以了解一下它的实现。

``` java
public RequestManager get(@NonNull android.app.Fragment fragment) {  
  if (fragment.getActivity() == null) {  
    throw new IllegalArgumentException(  
        "You cannot start a load on a fragment before it is attached");  
  }  
  if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {  
    return get(fragment.getActivity().getApplicationContext());  
  } else {  
    // In some unusual cases, it's possible to have a Fragment not hosted by an activity. There's  
 // not all that much we can do here. Most apps will be started with a standard activity. If // we manage not to register the first frame waiter for a while, the consequences are not // catastrophic, we'll just use some extra memory.  if (fragment.getActivity() != null) {  
      frameWaiter.registerSelf(fragment.getActivity());  
  }  
    android.app.FragmentManager fm = fragment.getChildFragmentManager();  
 return fragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());  
  }  
}
private RequestManagerFragment getRequestManagerFragment(  
    @NonNull final android.app.FragmentManager fm, @Nullable android.app.Fragment parentHint) {  
  // If we have a pending Fragment, we need to continue to use the pending Fragment. Otherwise  
 // there's a race where an old Fragment could be added and retrieved here before our logic to // add our pending Fragment notices. That can then result in both the pending Fragmeng and the // old Fragment having requests running for them, which is impossible to safely unwind.  RequestManagerFragment current = pendingRequestManagerFragments.get(fm);  
 if (current == null) {  
    current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);  
 if (current == null) {  
      current = new RequestManagerFragment();  
  current.setParentFragmentHint(parentHint);  
  pendingRequestManagerFragments.put(fm, current);  
  fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();  
  handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();  
  }  
  }  
  return current;  
}
private RequestManager fragmentGet(  
    @NonNull Context context,  
  @NonNull android.app.FragmentManager fm,  
  @Nullable android.app.Fragment parentHint,  
 boolean isParentVisible) {  
  RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);  
  RequestManager requestManager = current.getRequestManager();  
 if (requestManager == null) {  
    // TODO(b/27524013): Factor out this Glide.get() call.  
  Glide glide = Glide.get(context);  
  requestManager =  
        factory.build(  
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);  
  // This is a bit of hack, we're going to start the RequestManager, but not the  
 // corresponding Lifecycle. It's safe to start the RequestManager, but starting the // Lifecycle might trigger memory leaks. See b/154405040  if (isParentVisible) {  
      requestManager.onStart();  
  }  
    current.setRequestManager(requestManager);  
  }  
  return requestManager;  
}
```

1. `getRequestManagerFragment` ：获取空白`Fragment`；
   1. 先尝试从缓存`map pendingRequestManagerFragments`中获取，不为空直接返回；
   2. 为空则创建，并将其加入到缓存`map`中，同时`commit`到对应的`Activity`，添加成功则移除缓存；
2. 模仿`androidx`中的`fragment`， 在`RequestManagerFragment`中保存`ActivityFragmentLifecycle`对象，并将`RequestManager`加入`Lifecycle`的观察者列表`lifecycleListeners`中，用来监听`Activity`的生命周期变化；

为什么要缓存？添加成功之后为什么要立即移除？

1. `fm`的`commit`操作是通过`Handler`实现，并不是立即添加成功，没有缓存，多次`request fragment `会创建多个；
2. 发送移除消息一定是在`commit`之后执行，清空缓存列表，减少内存占用；

### into流程

#### ModelLoader<Model, Data>

Model 对应传入的数据类型，Data为输出的数据类型。第一次加载的网络图片 Model为String 类型;

```java
private static void initializeDefaults(Context context,  
Registry registry,  
BitmapPool bitmapPool,  
ArrayPool arrayPool,  
GlideExperiments experiments){
...
registry.append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())  
.append(String.class, InputStream.class, new StringLoader.StreamFactory())  
.append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())  
.append(  
    String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
    ...
}

//ModelLoaderRegistry
private synchronized <A> List<ModelLoader<A, ?>> getModelLoadersForClass(  
    @NonNull Class<A> modelClass) {  
  List<ModelLoader<A, ?>> loaders = cache.get(modelClass);  
 if (loaders == null) {  
    loaders = Collections.unmodifiableList(multiModelLoaderFactory.build(modelClass));  
  cache.put(modelClass, loaders);  
  }  
  return loaders;  
}
public <A> List<ModelLoader<A, ?>> getModelLoaders(@NonNull A model) {  
  List<ModelLoader<A, ?>> modelLoaders = getModelLoadersForClass(getClass(model));  
 if (modelLoaders.isEmpty()) {  
    throw new NoModelLoaderAvailableException(model);  
  }  
  int size = modelLoaders.size();  
 boolean isEmpty = true;  
  List<ModelLoader<A, ?>> filteredLoaders = Collections.emptyList();  
  //noinspection ForLoopReplaceableByForEach to improve perf  
  for (int i = 0; i < size; i++) {  
    ModelLoader<A, ?> loader = modelLoaders.get(i);  
 if (loader.handles(model)) {  
      if (isEmpty) {  
        filteredLoaders = new ArrayList<>(size - i);  
  isEmpty = false;  
  }  
      filteredLoaders.add(loader);  
  }  
  }  
  if (filteredLoaders.isEmpty()) {  
    throw new NoModelLoaderAvailableException(model, modelLoaders);  
  }  
  return filteredLoaders;  
}
```

![Loader架构](https://github.com/Ionicliquid/imageres/raw/master/Model_String.png)

1. 根据传入的Model类型，在RegistryFactory中查找对应的Loader，String类型查到到4个;
2. Loader会重写handles方法，判断是否可以处理, DataUrlLoader<Model, Data> 首先过滤掉。

```
//DataUrlLoader<Model, Data>
public boolean handles(@NonNull Model model) {  
 return model.toString().startsWith(DATA_SCHEME_IMAGE);  
}
```

3. 遍历每一个MultiLoader 找到真正的处理器 UrlUriLoader;

```
//UrlUriLoader<Data>
public boolean handles(@NonNull Uri uri) {  
  return SCHEMES.contains(uri.getScheme());  
}
```

# 缓存

#### 内存缓存 

#### ActiveResources ：活动缓存/前台缓存/弱引用缓存

由弱引用WeakReference 结合ReferenceQueue 实现

``` java
ActiveResources(boolean isActiveResourceRetentionAllowed) {  
  this(  
      isActiveResourceRetentionAllowed,  
  java.util.concurrent.Executors.newSingleThreadExecutor(  
          new ThreadFactory() {  
            @Override  
  public Thread newThread(@NonNull final Runnable r) {  
              return new Thread(  
                  new Runnable() {  
                    @Override  
  public void run() {  
                      Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);  
  r.run();  
  }  
                  },  
  "glide-active-resources");  
  }  
          }));  
}  
  
@VisibleForTesting  
ActiveResources(  
    boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {  
  this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;  
 this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;  
  
  monitorClearedResourcesExecutor.execute(  
      new Runnable() {  
        @Override  
  public void run() {  
          cleanReferenceQueue();  
  }  
      });  
}
void cleanReferenceQueue() {  
  while (!isShutdown) {  
    try {  
      ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();  
  cleanupActiveReference(ref);  
  
  // This section for testing only.  
  DequeuedResourceCallback current = cb;  
 if (current != null) {  
        current.onResourceDequeued();  
  }  
      // End for testing only.  
  } catch (InterruptedException e) {  
      Thread.currentThread().interrupt();  
  }  
  }  
}
```

1. ActiveResources 创建时会启动一个THREAD_PRIORITY_BACKGROUND 后台线程，调用cleanReferenceQueue 方法，轮询取出引用队列中将要被GC的Resource进行释放.

```java
//添加
synchronized void activate(Key key, EngineResource<?> resource) {  
  ResourceWeakReference toPut =  
      new ResourceWeakReference(  
          key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);  
  
  ResourceWeakReference removed = activeEngineResources.put(key, toPut);  
 if (removed != null) {  
    removed.reset();  
  }  
}  
// 删除
synchronized void deactivate(Key key) {  
  ResourceWeakReference removed = activeEngineResources.remove(key);  
 if (removed != null) {  
    removed.reset();  
  }  
}
```

2. ResourceWeakReference 继承自WeakReference 封装了Resource和ReferenceQueue实行，激活时会加入到本地缓存的map中，用于前台使用，当引用被回收时，会将其加入到引用队列中。

``` java
// Engine
@Override  
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {  
  activeResources.deactivate(cacheKey);  
 if (resource.isMemoryCacheable()) {  
    cache.put(cacheKey, resource);  
  } else {  
    resourceRecycler.recycle(resource, /* forceNextFrame= */ false);  
  }  
}
```

3. 前台缓存被释放时，会加入到LruResourceCache中；

#### LruResourceCache

``` java
private final Map<T, Entry<Y>> cache = new LinkedHashMap<>(100, 0.75f, true);
public synchronized Y put(@NonNull T key, @Nullable Y item) {  
  final int itemSize = getSize(item);  
 if (itemSize >= maxSize) {  
    onItemEvicted(key, item);  
 return null;  }  
  
  if (item != null) {  
    currentSize += itemSize;  
  }  
  @Nullable Entry<Y> old = cache.put(key, item == null ? null : new Entry<>(item, itemSize));  
 if (old != null) {  
    currentSize -= old.size;  
  
 if (!old.value.equals(item)) {  
      onItemEvicted(key, old.value);  
  }  
  }  
  evict();  
  
 return old != null ? old.value : null;  
}
protected synchronized void trimToSize(long size) {  
  Map.Entry<T, Entry<Y>> last;  
  Iterator<Map.Entry<T, Entry<Y>>> cacheIterator;  
 while (currentSize > size) {  
    cacheIterator = cache.entrySet().iterator();  
  last = cacheIterator.next();  
 final Entry<Y> toRemove = last.getValue();  
  currentSize -= toRemove.size;  
 final T key = last.getKey();  
  cacheIterator.remove();  
  onItemEvicted(key, toRemove.value);  
  }  
}
// 动态计算maxSize
public void trimMemory(int level) {  
  if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {  
	   clearMemory();  
  } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN  
      || level == android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {  
   trimToSize(getMaxSize() / 2);  
  }  
}
```

1. 借助LinkedHashMap 实现 LRU缓存，与根据数量来实现的LRU不同，Glide是根据动态计算的图片累加大小来决定容量；
2. put
   a. 图片大小超过maxSize，直接废弃，不缓存；
   b. 已经存在相同key的元素，currentSize减去当前oldvalue大小，并实现替换；
   c. 如果currentSize> maxSize 大小，按序遍历map，移除最老的元素，并减去其大小，重新计算currentSize后继续遍历；
3. get: 将LinkedHashMap 的accessOrder设置为true， 借助其双向链表的数据结构，每次get时将元素移动至末尾.
4. 实现trimMemory，动态计算maxSize 或者彻底清除缓存；

#### 磁盘缓存

``` java
private void runWrapped() {  
  switch (runReason) {  
    case INITIALIZE:  
      stage = getNextStage(Stage.INITIALIZE);  
  currentGenerator = getNextGenerator();  
  runGenerators();  
 break; case SWITCH_TO_SOURCE_SERVICE:  
      runGenerators();  
 break; case DECODE_DATA:  
      decodeFromRetrievedData();  
 break; default:  
      throw new IllegalStateException("Unrecognized run reason: " + runReason);  
  }  
}
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

如果内存缓存没找到对应资源，接着从磁盘缓存中查找，磁盘缓存分为`RESOURCE_CACHE `和`DATA_CACHE` 分别对应`ResourceCacheGenerator`和`DataCacheGenerator`；

1. **<font color="#ff0000">磁盘缓存的实现？DiskLruCache</font>**
2. gif 加载？

