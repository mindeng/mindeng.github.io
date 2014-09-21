---
layout: post
title:  "Android Service 详解"
date:   2014-09-19
tags:   [android, service]
---

*温馨提示：如果英文好请直接阅读官方文档：
[Services](http://developer.android.com/guide/components/services.html)
。尽管这篇文章包含了我的一些个人思考和理解，但我的所谓“详解”应该不会超出官方资料。*

基本概念
=======

Android 使用 Service 这个概念，来抽象后台服务，与 Activity 对应，意指不
提供用户界面的、可长期在后台运行的程序组件。

应用程序的其他组件（一般就是 Activity）可以 `start` 一个 Service，当
Service 启动后，即使用户按 Home 键返回主屏、或切换到其他应用，该
Service 仍会继续在后台运行。另外，也可以 `bind` 一个 Service，以便和该
Service 交互，这种交互可以是跨进程的（IPC）。

Service 的分类
-------------

Service 主要可分为两类：

* Started

  其他组件（例如 Activity）通过调用
  [startActivity()](http://developer.android.com/reference/android/content/Context.html#startService(android.content.Intent))
  方法启动的 service，即为 started service。这类 service 可以脱离启动它
  的组件独立运行。即使启动它的组件被销毁了，该 service 还是可以继续在后
  台运行，直到你调用 stopSelf 接口停掉它。举个例子，你可以启动一个 started
  service 来上传文件，上传完以后自己停掉就行，这样上传文件的操作就不会受 Activity 生
  命周期的影响。
  
* Bound
  
    通过调用
    [bindService](http://developer.android.com/reference/android/content/Context.html#bindService(android.content.Intent,
    android.content.ServiceConnection, int)) 方法启动的 service，即为
    bound service。这类 service 仅会在其他组件仍旧 bind 时生存，一旦没
    有其他组件 bind 它了，便会被系统销毁。一个 service 可以被多个其他组
    件同时 bind，当所有组件都解绑了，该 service 便会被系统销毁。bound
    service 提供 IBinder 接口供客户组件与之交互，这种交互可以
    是跨进程通讯（IPC）。
    
    **注意:** started 和 bound 并非互斥的概念，而是可以同时进行的，即一个
     service 可以同时被 start 和 bind。
    
无论是 started 还是 bound，任何其他组件（包括其他应用的组件）都可以通过
Intent 来使用你的 service，就像通过 Intent 使用 Activity 一样。当然，你
可以把你的 Service 声明为私有的，这样可以阻止其他应用的访问。关于
Service 的声明和 Android 权限机制等问题，后面可以另写一篇文章详细介
绍一下。

**注意:**
Service 默认是运行在 App 的主进程、主线程中的，即 Service 默
认不会创建自己的进程或线程。这意味着，如果你的 service 需要做一些 CPU
密集型操作或一些阻塞操作（例如播放MP3、网络I/O等），你应该在 service 里
创建新的线程来做这些操作。不然，你的应用极有可能出现 ANR 错误。其实从这
点，我们也可以看出 Service 和 Thread 的区别：

* Service 是 Android 框架层的概念，它仅意味着一个可以在后台运行的应用程
序组件，无关乎代码的运行、调度方式
* Thread 是操作系统层面的概念，是系统的调度单元，是代码真正执行的地方

当然，service 作为一个应用程序组件，则意味着它可以在不同组件间、甚至在
不同应用间进行复用，还意味着可以配置成在另一个独立的进程中运行。

如果仅仅只是需要在用户与你的应用交互时，执行一些后台操作（例如在使用你
的应用期间播放某段音乐），则大可不必使用 service，直接使用线程即可。

Service 的回调方法
-----------------

我们可以通过继承 Service 类的来创建自己的 Service。当然，也可以基于任何
现有的 Service 的子类来构建。

下面是 Service 类最重要的一些回调方法：

* [onStartCommand](http://developer.android.com/reference/android/app/Service.html#onStartCommand(android.content.Intent, int, int))
  
    对应 started service。当其他组件调用 startService() 时会被调用。一旦
    该方法被执行，service 即被启动并会一直运行下去。你需要负责在适当的时
    候，通过调用 stopService() 或 stopSelf() 来停止 service 的运行。如果你只
    想通过 bind 的方式使用你的 service，则可以不实现该方法。
    
* [onBind](http://developer.android.com/reference/android/app/Service.html#onBind(android.content.Intent))
  
    对应 bound service。当 server 第一次被绑定时会被调用。该方法需要返回
    一个 IBinder 接口，以供客户组件与你的 service 交互。该方法必须实现，如
    果你不想通过 bind 的方式访问你的 service，则应返回 null。
    
* [onCreate](http://developer.android.com/reference/android/app/Service.html#onCreate())
  
    该方法仅在 service 第一次被创建时会被系统调用，可以用来执行一些一次性的初始
    化操作。该方法会在 onStartCommand 和 onBind 之前被调用。
  
* [onDestroy](http://developer.android.com/reference/android/app/Service.html#onDestroy())
  
    当 service 不再被使用时，系统会调用该方法以销毁该 service。你应该在
    该方法中执行必要的清理工作，如停掉线程、反注册 listener、receiver 等。该
    方法是系统最后调用的方法。
  
系统只有在内存资源紧张且需要回收资源以供当前 Activity 使用时，才会强制
停止 service。

如果该 service 是 bind 到当前 Activity 的，则被杀的几率很
小，如果该 service 是在前台运行的（调用了 startForeground()方法），则几
乎不可能被杀掉。

如果 service 是普通的 started 的，则随着运行时间的增长，系统会降低其优
先级，在内存紧张时很可能会被杀掉。因此，你应该设计好你的 service，以便
可以优雅的应对重启的情况。如果系统杀掉了你的 service，一旦系统有足够的
资源，便会重新启动被杀的 service （依赖于你的onStartCommand() 的返回值，
此行为会有所不同。后面会有具体介绍）。

下面介绍 Service 的具体创建方法。

在 manifest 中声明 service
-----------------------------

和 Activity 类似，你需要在 manifest 文件中声明你的 service。如下例所示：

```
<manifest ... >
  ...
  <application ... >
      <service android:name=".MyService" />
      ...
  </application>
</manifest>
```

参考
[<service>](http://developer.android.com/guide/topics/manifest/service-element.html)
了解更多声明 service 时可使用的配置属性。

在众多属性中，`android.name` 是唯一必须的属性，它指定了 service 的类名。

**注意:** 你的应用一旦发布，就不应再修改这个属性，因为如果 service 的
类名被修改了，则会影响到其他显示调用你的 service 的客户端代码（除非你的
service 是完全私有的，且所有代码都在你自己的控制范围内）。Android 中还
有其他一些不应修改的东西，可参考 [Things That Cannot
Change](http://android-developers.blogspot.com/2011/06/things-that-cannot-change.html)
。

**安全小贴士**
* 尽量使用显式 intent 来访问 service
* 尽量不要为你的 service 定义 intent filters
* 尽量使用私有 service，即把 "android:exported" 属性定义为 false
* 如果需要使用 exported service，请使用 signature 级别的权限，或者自己做好安全检查

Started Service
===============

创建 Started Service
--------------------

一个应用组件（例如 Activity）可以通过调用 startService()并传入一个
Intent 来启动指定的 service，Intent 可以带上所需的数据，Service 会在
onStartCommand() 回调中收到该 Intent。

例如，假如一个 Activity 需要上传一些数据，则可以 start 一个service 并透
过 Intent 把需要保存的数据（或数据的路径）传入。service在
onStartCommand() 中 收到 Intent，可以提取出数据并上传。上传完毕，应调用
stopSelf() 停止服务。

你可以通过继承如下两个类来创建一个 started service：

* [Service](http://developer.android.com/reference/android/app/Service.html)

    所有 service 的基类。如果你扩展该类，你应该自己创建线程来处理阻塞任
    务，否则会阻塞 UI 线程。
  
* [IntentService](http://developer.android.com/reference/android/app/IntentService.html)

    Service 的一个子类，该类会创建一个线程来处理所有的 start 请求，一次处
    理一个。如果你不需要你的 service 同时处理多个请求，这个类是你的最佳
    选择。你只需实现 onHandleIntent() 方法即可，该方法会收到每次 start
    请求的 intent，你可以针对每个请求做处理。

IntentService 的代码非常简单，建议可以阅读一下[源码](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/app/IntentService.java)。
。

参考 IntentService 的源码，我们会注意到 onStartCommand() 方法返回了一
个整数，该整数描述了当系统杀掉你的 service 时，后续会如何处置你的 service。
该返回值的取值范围如下：

* START_NOT_STICKY

    如果你的 service 被杀掉了，将不会自动重启（除非系统中还有未处理的
    intent 请求等待投递）。

* START_STICKY

    如果系统杀掉了 service，将会在资源不再紧张时重新启动你的 service，
    并调用 onStartCommand() 方法。注意，该方法 *不会* 重新传入上次的
    intent，而是传入一个 null intent（除非系统中有待投递的请求，则会传
    入待投递请求的 Intent）。这种情况适合类似媒体播放器的应用，
    这类 service 在启动时并非执行某个指令，而是持续运行并等待任务。

* START_REDELIVER_INTENT

    如果 service 被杀掉了，将会被重启并且会传入上一次的 intent，其他未
    处理的 intent 将会依次被投递。该选项适合用来执行一些希望立即被恢复的
    任务，例如下载文件。

启动/停止一个 Started Service
------------------------

我们知道，可以在一个 activity 或其他应用组件中，通过调用
startService() 并传入一个Intent 来启动 service。startService() 方法会立
即返回，系统会负责调用 service 的 onStartCommand() 方法。如果该
service 当前没有运行，则会先调用 service 的 onCreate() 方法，再调用
onStartCommand()。

如果 service 未提供 binding 调用方式，则随 startService() 方法传入的
intent 是唯一的通讯方式。不过，如果你希望 service 能够返回结果，可以传
入一个广播的 PendingIntent （通过调用 PendingIntent.getBroadcast()），
service 可以通过该广播来投递结果。

多个 start 请求对应多次 onStartCommand() 调用，但是仅需要一次 stop 请
求便可停止 service。

我们来看一下 `onStartCommand()` 的原型：

```java
public int onStartCommand(Intent intent, int flags, int startId) {
    // ...
}
```

我们注意到有一个 `startId` 的参数。这个参数是用来干嘛的呢？文档上说，该
id 是作为本次请求的唯一标识，可以用作 stopSelf(int) 的参数。为什么stop
时需要该参数？原因是，当我们在 stopSelf 的时候，并不知道系统中是否还有
待处理的 intent，所以，如果粗暴直接的不带参数调用 stopSelf()，则会直接
停止 service，从而可能会丢掉潜在的未处理请求。而如果调用带参数的版本，
传入本次已处理完的请求的 id，则会判断该 id 是否是系统最后一次收到的请求
id，如果是则可以停止 service，否则该 service 会继续运行以执行未处理的请
求。

Bound Service
=====================

当你需要与 service 进行交互，或者需要暴露出一些功能提供给其他应用使用，
则可以使用 bound service。

要创建一个 bound service，你必须实现 onBind() 回调方法，并返回一个
IBinder 供 client 使用。其他的应用组件通过调用 bindService() 来绑定你
的 service，并获得 IBinder 接口以便和你的 service 交互。这类 service
存在的目的就是为了服务于 bind 它的其他组件，所以，一旦没有组件 bind 它
了，则会被系统销毁。

多个 clients 可以同时 bind 一个 service，当所有 clients 通过调用
unbindService() 解除绑定时，系统便会销毁该 service。

Bound service 的创建和使用相对 started service 来说会复杂一些。如有必要，
后面可以再开一篇介绍一下。这里重点介绍一下 IBinder 的几种创建方式。

要创建一个 bound service，最重要的是要先定义好与 client 交互
的接口，即 IBinder。根据应用场景的不同，Android 提供了三种创建 IBinder
的方法：

* 继承 Binder 类

    这是最简单的一种方式。如果你的 service 是私有的（即仅供本 app 使
    用），并且和 client 运行在同一个进程空间，则可以通过继承 Binder 来创建
    你的接口类，并在 onBind() 方法中返回该类的实例。client 拿到该实例，
    便可以访问任何你在该类中定义的 public 方法。

* 使用 Messenger 类

    如果你需要跨进程访问你的 service，你可以通过 Messenger 来创建
    IBinder 接口。这是一种最简单的实现 IPC 的方式，因为 Messenger 会把
    所有的请求放入一个请求队列，这样你就不用担心同时处理多个请求，也就不
    用担心线程安全问题了（你只需一次处理一个请求）。

* 使用 AIDL

    AIDL (Android Interface Definition Language) 是最复杂的创建
    IBinder 的方式。与 Messenger 相比，它允许你同时处理多个请求，也因此，
    你需要编写多线程代码并处理多线程安全问题。事实上，Messenger 在底层
    也是基于AIDL 实现的。在 Android 的官方文档中，特别提到一点，就是大
    多数应用程序不需要也不应该使用这种方式来构建 IBinder 接口，因为这会
    引入多线程代码，从而招致不必要的复杂性，除非你真的、真的确定你的
    service 需要具备同时处理多个请求的能力。

这里不得不吐槽一句，我遇到过的绝大多数开发者（包括我面试过的几乎所有候
选人），一旦提到 Service 通讯方式则言必称 AIDL，而对另外两种官方更
加推荐的方式则只字不提。我相信大多数场景下，继承 Binder 类和使用
Messenger 应该足够我们使用了，为啥就鲜有人提及呢？真是难以理解。

还有一点值得一提的是，当一个 service 被多个 clients 绑定时，onBind() 只
会在第一次绑定时被系统调用一次，用于获取 IBinder 实例，后续的 bind 操作
系统会直接返回相同的 IBinder 实例，而不再调用 onBind()。这点和
onStartCommand() 有很大的区别。

另外，如果一个 service 同时被 started 和 binding，你可以选择在
onUnbind() 回调方法中返回 `true`，这样下次 service 再被 client 绑定时，
就不会调用 onBind() 方法，会改调用 onRebind() 方法。当然，前提是
service 尚未被停掉。

向用户发送通知
============

在 Service 中可以通过 toast 或者状态来通知用户某个事件的发生。

通常情况下，当某个后台任务完成时，使用状态栏通知是比较好的选择。例如，
当一个文件在 service 中下载完成了，可以创建一个状态栏通知来通知用户。当
用户点击该通知时，便可启动相应的 activity 来查看结果。

前台 Service
============

一个前台 service 意味着用户知道该 service 正在运行，因此当系统内存紧张
时，该 service 不会被放在 kill 的候选名单中。一个前台 service 必须提供
一个 "Ongoing" 的状态栏通知，"Ongoing" 意味着该通知不能被 dismiss，除
非该 service 停止了或者取消了前台状态。

例如，一个音乐播放器可以将其播放 service 设置成前台 service，因为用户知
道该任务正在运行。可以通过通知栏通知来显示当前正在播放的音乐，同时用户
可以点击该通知进入播放界面。

通过调用 startForeground() 方法可以将当前 service 设置成前台 service。
该方法需要提供两个参数：notification ID 和 notification。举例如下：

```java
Intent notificationIntent = new Intent(this, ExampleActivity.class);
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, 0);
NotificationCompat.Builder builder =
        new NotificationCompat.Builder(this)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle("My notification")
        .setContentText("Hello World!")
	.setContentIntent(pendingIntent);
Notification notification = builder.build();
startForeground(ONGOING_NOTIFICATION_ID, notification);
```

**注意:** 你传入的通知 ID 不可以为0。

要取消前台运行状态，可以调用 stopForeground() 方法。该方法传入一个
boolean 值，表明是否同时移除通知栏通知。

要点总结
=======

* Service 默认运行在主进程、主线程中
* Started Service 的生命周期不依赖于启动它的组件，仅取决于你何时调用 stop 方法停止 service
* Bound Service 的生命周期依赖于绑定它的组件
* 每次 startService() 调用，都会产生一次 onStartCommand() 回调
* 每次 bindService() 调用，不一定会产生一次 onBind() 回调。只有 service 第一次被绑定时，才会产生 onBind() 调用
* 多次 startService() 调用，仅需一次 stop 即可停止
* 多次 bindService() 调用，需要对应多次 unbindService() 调用
* 一个 service 可以同时被多个 client 绑定，也可以被多次 start，亦可以同时被 bind 和 start
* 可以使用 startForeground() 方法使 service 在前台运行，前台 service 基本不会被系统 kill 掉
