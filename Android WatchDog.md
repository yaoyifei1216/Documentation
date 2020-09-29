---
title: Android WatchDog
tags: 
 - android
 - system stability
categories:
 - 笔记
---
## Android WatchDog

#### 1.1问题背景

**测试机点击设置中的重置应用偏好后界面卡死**

【预置条件】无
【操作步骤】设置->系统->重置选项->点击重置应用偏好
【实际结果】界面卡死点击无反应
【预期结果】重置后界面可点击
【发生概率】10/10

#### 1.2问题原因:

```
"Binder:6102_1C" prio=5 tid=128 Blocked
waiting to lock <0x053efd20> (a com.android.server.am.ActivityManagerService) held by thread 32
"PackageManager" prio=5 tid=32 Blocked
waiting to lock <0x081d6b54> (a android.util.ArrayMap) held by thread 128
```

**循环等待导致死锁,触发system server watchdog**

#### 2.1WatchDog简介

​	Android中的watchdog功能，用于监视系统的运行，watchdog 定时器，会监测特定进程的运行情况，如果监测的进程在一定时间内没有响应，那么watchdog会输出一定的信息表明当前系统所处的状态，严重的情况下会导致watchdog重启。这种机制保证Android系统正常稳定运行。

上文中出现的问题原因就是进程死锁导致watchdog重启

#### 2.2WatchDog类

[xref](http://aospxref.com/android-10.0.0_r2/xref/): /[frameworks](http://aospxref.com/android-10.0.0_r2/xref/frameworks/)/[base](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/)/[services](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/)/[core](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/)/[java](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/java/)/[com](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/java/com/)/[android](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/java/com/android/)/[server](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/)/[Watchdog.java](http://aospxref.com/android-10.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/Watchdog.java)

类结构

```java
public class Watchdog extends Thread {......}
```

Watchdog之所以是个线程类,是因为系统启动，必定不能在system_server的当前线程中监测特定进程，一定要新开一个常驻的线程，只要system_server在，那么这个线程就会一直监测需要监测的进程。

构造方法

```java
private Watchdog() {
300          super("watchdog");
301          // Initialize handler checkers for each common thread we want to check.  Note
302          // that we are not currently checking the background thread, since it can
303          // potentially hold longer running operations with no guarantees about the timeliness
304          // of operations there.
305  
306          // The shared foreground thread is the main checker.  It is where we
307          // will also dispatch monitor checks and do other work.
308          mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
309                  "foreground thread", DEFAULT_TIMEOUT);
310          mHandlerCheckers.add(mMonitorChecker);
311          // Add checker for main thread.  We only do a quick check since there
312          // can be UI running on the thread.
313          mHandlerCheckers.add(new HandlerChecker(new Handler(Looper.getMainLooper()),
314                  "main thread", DEFAULT_TIMEOUT));
315          // Add checker for shared UI thread.
316          mHandlerCheckers.add(new HandlerChecker(UiThread.getHandler(),
317                  "ui thread", DEFAULT_TIMEOUT));
318          // And also check IO thread.
319          mHandlerCheckers.add(new HandlerChecker(IoThread.getHandler(),
320                  "i/o thread", DEFAULT_TIMEOUT));
321          // And the display thread.
322          mHandlerCheckers.add(new HandlerChecker(DisplayThread.getHandler(),
323                  "display thread", DEFAULT_TIMEOUT));
324          // And the animation thread.
325          mHandlerCheckers.add(new HandlerChecker(AnimationThread.getHandler(),
326                  "animation thread", DEFAULT_TIMEOUT));
327          // And the surface animation thread.
328          mHandlerCheckers.add(new HandlerChecker(SurfaceAnimationThread.getHandler(),
329                  "surface animation thread", DEFAULT_TIMEOUT));
330  
331          // Initialize monitor for Binder threads.
332          addMonitor(new BinderThreadMonitor());
333  
334          mOpenFdMonitor = OpenFdMonitor.create();
335  
336          // See the notes on DEFAULT_TIMEOUT.
337          assert DB ||
338                  DEFAULT_TIMEOUT > ZygoteConnectionConstants.WRAPPED_PID_TIMEOUT_MILLIS;
      }
```

私有构造方法