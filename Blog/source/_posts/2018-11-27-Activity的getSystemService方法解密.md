---
layout: post
title: Activity的getSystemService方法解密
date: 2018-11-27 18:01:47
categories: 
- Andriod源码

tags:
- Android
- 源码解析
- 高级UI
---


## 写在前面

开发过程中，我们经常需要一些系统服务，比如 LayoutInflater、ActivityManager 等等，每次获取的时候我们都直接调用 Activity,Service 的 getSystemService(String name) 这个方法，然而这个方法是怎么工作的？每次获取的对象是不是一个？我们都不太清楚，今天，我们就来揭开 getSystemService 方法的面纱，剖析他的工作过程。

<!-- more -->

## 概要 ##

### 系统启动

系统启动时会调用 SystemServer.main()，其中会调用 SystemThread.run() 方法，run 方法中调用 SystemServiceManager 的 startService 方法使用反射实例化  AMS、PMS... 等系统对象并调用其 onStart 方法启动服务

IServiceManager 通过 Binder 机制向外提供 AMS、PMS... 等系统服务


### 应用进程  

通过 Binder 在 Native 层获取 IServiceManager 在应用进程的代理

ServiceManager 类代理 ServiceManagerProxy(是 IServiceManager 在客户端的代理类)

通过 ServiceManager 获取 AMS、PMS 等系统服务在客户端的代理

**IServiceManager** 是 Binder 进程间通信机制的守护进程，其目的非常简单，就是管理 Android 系统里的 Binder 对象


**注意：以下说的 service 是指系统服务的对象名，并不是我们的四大组件中的 Service ，不要混淆，这里不涉及四大组件中的 Service**


## 一、开始


平时我们获取系统服务都是直接通过 Context 对象的 getService(String name) 方法来获取，这个方法是在 ContextImpl 中的实现的

```Java
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

调用了 SystemServiceRegistry.getSystemService(this, name) 方法，我们接着往下看

```Java
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

getSystemService 方法中会在 SYSTEM_SERVICE_FETCHERS 中根据 name 来取一个 ServiceFetcher 对象，有必要先看看这个对象是什么了

```Java  
private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
    new HashMap<Class<?>, String>();

private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
        new HashMap<String, ServiceFetcher<?>>();

static abstract interface ServiceFetcher<T> {
    T getService(ContextImpl ctx);
}  
```

**SYSTEM_SERVICE_NAMES** 是 SystemServiceRegistry 类中的一个常量，类型为一个 HashMap，key 为想要获取的系统服务的对应 Class，value 为 Context 中定义的对应该 Service 的 String 常量。

**SYSTEM_SERVICE_FETCHERS** 是 SystemServiceRegistry 类中的一个常量，类型为一个 HashMap，key 在 Context 中定义的对应具体某个 Service 的 String 常量，value 为 ServiceFetcher 对象。

**ServiceFetcher** 为一个接口，需要一个泛型，并且定义了 getService 方法，这个方法返回的就是泛型 T 的对象

我们看到 getSystemService 方法在 ServiceFetcher 不为 null 时最终返回的是 ServiceFetcher 的 getService 方法的返回值，并没有通过别的方式来获取 Service，说明 SYSTEM_SERVICE_FETCHERS 中已经存在了我们要的东西，我们就看看 SYSTEM_SERVICE_FETCHERS 什么时候添加了内容

```Java
final class SystemServiceRegistry {    

    static {
        registerService(Context.ACCESSIBILITY_SERVICE, AccessibilityManager.class,
                new CachedServiceFetcher<AccessibilityManager>() {
            @Override
            public AccessibilityManager createService(ContextImpl ctx) {
                return AccessibilityManager.getInstance(ctx);
            }});

        registerService(Context.CAPTIONING_SERVICE, CaptioningManager.class,
                new CachedServiceFetcher<CaptioningManager>() {
            @Override
            public CaptioningManager createService(ContextImpl ctx) {
                return new CaptioningManager(ctx);
            }});

        registerService(Context.ACCOUNT_SERVICE, AccountManager.class,
                new CachedServiceFetcher<AccountManager>() {
            @Override
            public AccountManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ACCOUNT_SERVICE);
                IAccountManager service = IAccountManager.Stub.asInterface(b);
                return new AccountManager(ctx, service);
            }});

        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});
            
        ......// 省略更多的 registerService 方法调用
    }
    
    
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }        
}
```

可以看到，SYSTEM_SERVICE_NAMES 和 SYSTEM_SERVICE_FETCHERS 添加内容是在 registerService 方法，中，那么哪里调用了这个方法呢，原来在 SystemServiceRegistry 类的静态代码块中调用了多次  registerService 方法。我们来具体看一个，就挑我们最熟悉的 ActivityManagerService 的方法

```Java
registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
        new CachedServiceFetcher<ActivityManager>() {
    @Override
    public ActivityManager createService(ContextImpl ctx) {
        return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
    }
});
```

可以看到，注册方法中，创建了一个 CachedServiceFetcher 的匿名内部类，看一下这个类 

```Java
static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
    private final int mCacheIndex;

    public CachedServiceFetcher() {
        mCacheIndex = sServiceCacheSize++;
    }

    @Override
    @SuppressWarnings("unchecked")
    public final T getService(ContextImpl ctx) {
        final Object[] cache = ctx.mServiceCache;
        synchronized (cache) {
            // Fetch or create the service.
            Object service = cache[mCacheIndex];
            if (service == null) {
                service = createService(ctx);
                cache[mCacheIndex] = service;
            }
            return (T)service;
        }
    }

    public abstract T createService(ContextImpl ctx);
}
```

CachedServiceFetcher 类继承了 ServiceFetcher ，所以说添加到 SYSTEM_SERVICE_FETCHERS 中没有问题，CachedServiceFetcher 中有一个 final 的 mCacheIndex 属性，还有一个 final 的 getService 方法， 还有一个抽象的 createService 方法，下面一一介绍

- **mCacheIndex** 这个属性在创建对象的时候赋值，赋值为  mCacheIndex = sServiceCacheSize++;  sServiceCacheSize 是 SystemServiceRegistry 中的一个静态的值，初始值为 0
    
        private static int sServiceCacheSize;

也就是说静态代码块中每调用一次 registerService 方法，这个值就会自加一，并且每个 CachedServiceFetcher 在 HashMap 中的索引都由 mCacheIndex 属性来记录。也就是说 sServiceCacheSize 是的值就是下一个放入 HashMap 中的 CachedServiceFetcher 对象的索引。而每一个 
CachedServiceFetcher 在 HashMap 中的索引也就是自己的 mCacheIndex 属性的值。

剩下的两个方法后面讲，先看上面的代码，到静态代码块执行结束，也就是 Context 的 getService 能得到的系统 Service 对应的 CachedServiceFetcher 都已经初始化，SYSTEM_SERVICE_NAMES 和 SYSTEM_SERVICE_FETCHERS 都被赋值。

返回去看 getSystemService 方法，由于 SYSTEM_SERVICE_FETCHERS 中的 key 为 String 类型的 name，我们根据 name 在 SYSTEM_SERVICE_FETCHERS 中取出对应的 ServiceFetcher ，这里是 CachedServiceFetcher ，再调用其 getService 方法，并将 ContextImpl 参数传入，这里就详细说一下 getService 方法

- **getService** 由上面 CachedServiceFetcher 类的代码中可以看到，getService 方法中首选创建了一个 Object 数组，其值为 ctx.mServiceCache ，而 mServiceCache 的值是根据 sServiceCacheSize 的长度创建的数组，源码如下：
- 
```Java
// ContextImpl 
final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();

public static Object[] createServiceCache() { // 该方法在 ContextImpl 实例化时调用，所以 sServiceCacheSize 已经为注册的系统服务的数量值
    return new Object[sServiceCacheSize];
}
```

得到数组 cache 之后，使用 synchronized 锁防止多个线程同时访问，从 cache 数组中根据 mCacheIndex 索引得到 Object 类型的 service 对象，如果 service 为 null ，则调用 creatService 方法来创建 service，创建之后为 service[mCacheIndex] 赋值。如果 service 不为 null ，则直接返回。createService 为抽象方法，具体的实现在匿名内部类中，以 ActivityManager 为例：

```Java
registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
        new CachedServiceFetcher<ActivityManager>() {
    @Override
    public ActivityManager createService(ContextImpl ctx) {
        return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
    }
});
```

createService 方法中，直接 new 了一个 ActivityManager ，然后返回，这里我们不深究 ActivityManager 的创建过程。

由 creatService 方法就得到了 ActivityManager 实例，然后，cache[mCacheIndex] = service; 将该实例放入 cache 数组，cache 数组为每一个 ContextImpl 对象自有的，并不是静态的。

由 getService 方法可以看出，ContextImpl 中的 ActivityManager 在第一次获取时才会通过 createService 方法赋值，并且第一次初始化之后，同一个 ContextImpl 再次获取的时候并不会重新创建新的 ActivityManager 实例，也就是说 ActivityManager 是同一 ContextImpl 单例，不同的 ContextImpl 会创建不同的 ActivityManager。

**LayoutInflater 并不是单例的，其构造模式同 ActivityManager**

**注意：** 在学习了 Binder 之后，会发现 ActivityManager 其实只是系统的 AMS 在客户端(上层应用)的代理 ActivityManagerProxy 的代理(这里包含两层代理，ActivityManagerProxy 对 AMS 远程代理，ActivityManager 对 ActivityManagerProxy 代理)，ActivityManagerProxy 并不是整个系统单例的，但是 AMS 确实整个系统单例的，有关 Binder 和代理模式的知识点请看这里

[Binder](http://www.jianshu.com/p/5e408875b615)  
[代理模式](http://www.jianshu.com/p/7ac9e3b8600a)


## 二、有没有单例的系统服务？

当然有了，其实 ServiceFetcher 的实现了还有两个，StaticServiceFetcher，StaticApplicationContextServiceFetcher ，我们看一个就行了，他们的 create 方法创建的服务会赋值给他们的一个内部属性，这个属性就是对应的系统服务对象在客户端的代理，其实例化之后，再获取时都会返回这个对象。

```Java
static abstract class StaticServiceFetcher<T> implements ServiceFetcher<T> {
    private T mCachedInstance;

    @Override
    public final T getService(ContextImpl unused) {
        synchronized (StaticServiceFetcher.this) {
            if (mCachedInstance == null) {
                // 第一次创建之后赋值给自己的内部属性 mCachedInstance
                mCachedInstance = createService();
            }
            return mCachedInstance;
        }
    }

    public abstract T createService();
}

registerService(Context.HDMI_CONTROL_SERVICE, HdmiControlManager.class,
        new StaticServiceFetcher<HdmiControlManager>() {
    @Override
    public HdmiControlManager createService() {
        IBinder b = ServiceManager.getService(Context.HDMI_CONTROL_SERVICE);
        return new HdmiControlManager(IHdmiControlService.Stub.asInterface(b));
    }
});
```

## 三、延伸

刚才分析的是 ActivityManager ，还有向 IWifiPasspointManager 等系统服务在客户端的代理也可以通过 SystemServiceRegistry 来获取，但是这个过程同样需要使用 IPC 远程获取。

IWifiPasspointManager 等作为系统中的服务，在开机时就会被启动，而且一直存在，注意运行在系统进程，IWifiPasspointManager，随着系统进程退出而消亡，可以说，IWifiPasspointManager 等系统服务与系统进程共存亡。

在应用开发中使用 IWifiPasspointManager 等系统服务时都是通过 Binder 机制获取 IWifiPasspointManager 等服务的代理，通过代理调用 IWifiPasspointManager 等服务的方法。此工作在 Android 中由 ServiceManager 完成，ServiceManager 其实是 IManagerService 在应用进程的代理，其获取是通过 Binder 由 Native 层获取的。



## 四、总结

SystemServiceRegistry 类第一次加载时就会把获取一些系统服务或者应用进程通用服务的 ServiceFetcher 放进自己内部的一个 HashMap，其中 ServiceFetcher 有两种，第一种是 LayoutInflater ActivityManager 等对应的 CacheServiceFetcher；第二种是 StaticServiceFetcher 跟 StaticApplicationContextServiceFetcher。

ServiceFetcher 中定义了创建以及获取对应服务的方法。其实在每次获取服务时都是通过 SystemServiceRegistry 找到内部系统服务对应的 ServiceFetcher ，再由 ServiceFetcher 来获取自己对应的系统服务。

其中有一个需要注意的点，CacheServiceFetcher 是每个 Context 都会创建一个服务对象。而 StaticServiceFetcher 和 StaticApplicationContextServiceFetcher 这种创建的系统服务是单例的服务对象。

终于，撕开了 getSystemService 方法的面纱。
