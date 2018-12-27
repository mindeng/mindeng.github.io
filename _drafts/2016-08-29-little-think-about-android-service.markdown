---
layout: post
title:  "关于 Android Service 的一点思考"
date:   2016-08-29
tags:   [android, service]
---

[Android Service - Official document](https://developer.android.com/reference/android/app/Service.html)

```
As of GINGERBREAD, when using Context.startService(Intent), you can also set
Intent.FLAG_GRANT_READ_URI_PERMISSION and/or
Intent.FLAG_GRANT_WRITE_URI_PERMISSION on the Intent. This will grant the
Service temporary access to the specific URIs in the Intent. Access will remain
until the Service has called stopSelf(int) for that start command or a later
one, or until the Service has been completely stopped. This works for granting
access to the other apps that have not requested the permission protecting the
Service, or even when the Service is not exported at all.

In addition, a service can protect individual IPC calls into it with
permissions, by calling the checkCallingPermission(String) method before
executing the implementation of that call.
```

从 GINGERBREAD (November 2010, Android 2.3, API Level 9) 开始，调用
Context.startService(Intent) 时可以为参数 Intent 设置
Intent.FLAG_GRANT_READ_URI_PERMISSION 和/或
Intent.FLAG_GRANT_WRITE_URI_PERMISSION 标志位。这些标志位会授予 service
临时访问指定 URIs 的权限。访问权限会持续生效直到该次 command 或后续 command 被
stopSelf(int) 方法停掉，或者该 service
被彻底停止。这个功能可以用来授予访问权限给其他应用，以便执行相关服务请求,  or
even when the Service is not exported at all (甚至当该 Service 是非 exported
的也可以使用该功能? 后面这句没太懂）。

```
Process Lifecycle
The Android system will attempt to keep the process hosting a service around as
long as the service has been started or has clients bound to it. When running
low on memory and needing to kill existing processes, the priority of a process
hosting the service will be the higher of the following possibilities:

* If the service is currently executing code in its onCreate(), onStartCommand(),
or onDestroy() methods, then the hosting process will be a foreground process
to ensure this code can execute without being killed.

* If the service has been started, then its hosting process is considered to be
less important than any processes that are currently visible to the user
on-screen, but more important than any process not visible. Because only a few
processes are generally visible to the user, this means that the service should
not be killed except in low memory conditions. However, since the user is not
directly aware of a background service, in that state it is considered a valid
candidate to kill, and you should be prepared for this to happen. In
particular, long-running services will be increasingly likely to kill and are
guaranteed to be killed (and restarted if appropriate) if they remain started
long enough.

* If there are clients bound to the service, then the service's hosting process
is never less important than the most important client. That is, if one of its
clients is visible to the user, then the service itself is considered to be
visible. The way a client's importance impacts the service's importance can be
adjusted through BIND_ABOVE_CLIENT, BIND_ALLOW_OOM_MANAGEMENT,
BIND_WAIVE_PRIORITY, BIND_IMPORTANT, and BIND_ADJUST_WITH_ACTIVITY.

* A started service can use the startForeground(int, Notification) API to put the
service in a foreground state, where the system considers it to be something
the user is actively aware of and thus not a candidate for killing when low on
memory. (It is still theoretically possible for the service to be killed under
extreme memory pressure from the current foreground application, but in
practice this should not be a concern.)

Note this means that most of the time your service is running, it may be killed
by the system if it is under heavy memory pressure. If this happens, the system
will later try to restart the service. An important consequence of this is that
if you implement onStartCommand() to schedule work to be done asynchronously or
in another thread, then you may want to use START_FLAG_REDELIVERY to have the
system re-deliver an Intent for you so that it does not get lost if your service
is killed while processing it.

Other application components running in the same process as the service (such as
an Activity) can, of course, increase the importance of the overall process
beyond just the importance of the service itself.
```

进程生命周期
Android 系统将尝试保持 Service 进程不被干掉, 只要该 Service 是 started
状态或有客户端绑定状态。 当系统内存不足需要杀掉现有进程时，Service
进程优先级有下面几种可能：

* 如果 Service 正在执行 onCreate(), onStartCommand(), onDestroy()
  方法时，宿主进程会作为一个前台进程，以保证代码会被执行而不会被杀掉。

* 如果 Service 是 started
  状态，宿主进程的优先级会比任何对当前用户可见的进程的优先级要低，
  但是比不可见进程优先级要高。 因为通常只有少数进程对用户可见， 所以 Service
  应该不会被杀掉，除非在内存不足情况下。然而， 因为用户并不直接知道后台进程，
  所以在这种状态下， Service 是 可能被杀掉的。 你应该为此做好准备。 特别地，
  长时间运行的 Service 被杀掉的可能性会逐渐增高， 如果运行时间够长，
  最终肯定会被杀掉（在适当情况下会被重启）。

* 如果有客户端绑定了 Service，则该 Service
  的宿主进程的优先级永远不会比这些绑定了的客户端中最重要的客户端的优先级更低。
  换言之，只要其中一个客户端是对用户可见的， 则 Service 也是可见的。 客户端影响
  Service 重要性的方式可以通过 BIND_ABOVE_CLIENT, BIND_ALLOW_OOM_MANAGEMENT,
  BIND_WAIVE_PRIORITY, BIND_IMPORTANT, BIND_ADJUST_WITH_ACTIVITY 来调整。

* 一个 started service 可以通过调用 startForeground(int, Notification) API
  来变成前台状态， 系统会认为该服务是用户显示知道的,
  因此在低内存情况不会被杀掉。 （理论上来说，当处于极端内存压力情况下， Service
  仍有可能被从前台用用中杀掉，但在实践中可以不用考虑）

注意，这意味着你的 Service 运行时间越长，
在系统内存压力大的情况下被杀掉的可能性越大。 如果被杀掉了，
系统在晚些时候会尝试重启该 Service。 由此得出一个重要的推论， 如果你实现了
onStartCommand() 方法， 并使任务被异步处理或在另一个线程处理， 则你或许会想用
START_FLAG_REDELIVERY 标记，使得系统会为你重新投递 Intent,  这样即使当你的
Service 正在处理任务时 被杀了, 也不至于丢失该任务。

在 Service 进程中运行的其他应用组件 (例如 Activity） 会增加整体进程的重要性。

思考
====

**如何保证设备上安装的 Service 不是伪造的呢？**

临时想到的解决方案：在调用方也启动一个简单的 Service (使用 signature
权限），返回一个随机字符串。 如果目标 Service
能获取并返回该随机字符串，则表示具有相同的 signature 。

当然，也可以换一个思路，不向 Service 传递敏感信息，只发请求，数据可以以 ID 或
URI 的形式传递。Service 如果需要数据，可以通过 Provider 的形式提供，并使用
signature 权限。

**关于生命周期**

* Activity 销毁了，Activity 实例的弱引用仍可能继续存在一段时间，之后可能会为空
* Service 销毁了，Service 内创建的线程会继续运行，除非进程被杀掉
* Handler 是跟线程挂钩的，Activity 销毁了，只要进程没被杀掉，handler 仍然可以收到消息

