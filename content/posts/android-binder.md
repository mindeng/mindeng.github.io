+++
title = "Android 的 Binder 机制"
date = 2023-07-01T23:41:00+08:00
lastmod = 2023-07-11T14:02:01+08:00
tags = ["android"]
draft = false
+++

## 同步调用 {#同步调用}

`IBinder` 接口相关的关键 API 主要有两个：

**`IBinder.transact()`**
: 用来向一个 `IBinder` 对象发起调用请求。

**`Binder.onTransact()`**
: 用于处理 `Binder` 对象收到的调用。

请注意，这套 transaction API 是一套 **同步** API, 即一个 `transact()` 调用会一直阻塞，直到对面的 `Binder.onTransact()` 方法返回之后， `transact()` 调用才会返回。

之所以这么设计，是因为 `IBinder` 同时支持两种调用方式：

**本地进程调用**
: 和调用本地方法类似，用户期待的行为就是同步调用。

**远程 IPC 调用**
: 跨进程情况下，Android 底层 IPC 机制保证和本地调用有同样的语意。

本地进程调用的情况下，用户期待的行为就是同步调用。因此，IPC 也采用同样的策略，这样可以保证在业务进行重构（例如拆分多进程）时不受影响。


## 线程池机制 {#线程池机制}

系统在每个进程中都维护了一个“事务线程池” (a pool of transaction threads), 这些线程用于分发所有的来自其他进程的 IPC 调用。

正因如此，​**AIDL 接口的实现必须是线程安全的**​。类似地，​`ContentProvider` 方法（query()、
insert()、delete()、update() 和 getType() 方法）也必须实现为线程安全的方法。

参考示意图如下：

{{< figure src="/ox-hugo/android-binder-thread-pool.svg" >}}

调用时序图：

{{< figure src="/ox-hugo/android-binder-transact-sequence.svg" >}}

[如前所述](#同步调用)，对于 process A 来说，整个调用是一个同步等待的过程。


## 进程间递归调用 {#进程间递归调用}

_Binder_ 系统还支持进程之间的递归调用。例如：

1.  假设 process A 在主线程向 process B 发起一个 `transact()` 请求，我们称为
    _transact 1_ 吧；
2.  process B 在自己的线程池中处理 binder 请求时（即 `onTransact()` 方法），又向
    process A 发起了一个 `transact()` 请求，我们称为 _transact 2_ ；
3.  此时 process A 的主线程还在等待自己发出的 `transact()` 的响应，仍处于阻塞状态（参考[同步调用](#同步调用)）。不过由于 Android 会自动在 process A 中维护一个线程池，用于这类处理 IPC 请求，因此 _transact 2_ 可以得到正常处理，处理完之后，正常返回到进程 B 中；
4.  返回进程 B 后，继续处理 _transact 1_, 然后返回 process A, 结束 _transact 1_ 的调用。

试想一下，如果系统不维护一个线程池来处理这类 IPC 请求，那在 process B 递归的向
process A 发起 IPC 请求时，这两个进程可能就被卡死了。

上述过程是不是有点像在模拟一个本地的递归调用？事实上，和[同步调用](#同步调用)类似，之所以这么设计，也是为了尽量让上层应用对“IPC 调用”这件事情保持透明，底层可以任意切换组件的物理布局，而不影响上层的业务调用。

上述过程可以参考如下时序图：

{{< figure src="/ox-hugo/android-binder-recursion.svg" >}}


## 远程对象生命周期检测 {#远程对象生命周期检测}

和 `IBinder` 这类远程对象协同工作时，很重要的一点是对其生命周期的感知。一旦远程对象所在的进程被销毁了，该远程对象也会成为一个无效对象。

系统提供了如下几种方式用于 `IBinder` 对象的生命周期检测：

**`RemoteException`**
: 当 `IBinder` 对象所在的进程已经被销毁时，调用
    `IBinder.transact()` 方法会抛出该异常。

**`IBinder.pingBinder()`**
: 该方法返回 `false` 表示其所在进程已销毁。

**`IBinder.linkToDeath()`**
: 可用于注册一个 `DeathRecipient` 对象，用来接收进程销毁时的回调。


## 参考资料 {#参考资料}

-   _IBinder.java_ 源码
-   _Binder.java_ 源码
