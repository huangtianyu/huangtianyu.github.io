---
title: 应用内存优化之OnLowMemory&OnTrimMemory
date: 2018-01-03 23:39:09
categories: Android
tags: 内存优化
---
### 1.应用内存onLowMemory& onTrimMemory优化


onLowMemory& onTrimMemory简介：
OnLowMemory是Android提供的API，在系统内存不足，所有后台程序（优先级为background的进程，不是指后台运行的进程）都被杀死时，系统会调用OnLowMemory。
OnTrimMemory是Android 4.0之后提供的API，系统会根据不同的内存状态来回调。根据不同的内存状态，来响应不同的内存释放策略。


#### 1.1 onLowMemory& onTrimMemory优化，需要释放什么资源？


在内存紧张的时候，会回调OnLowMemory/OnTrimMemory，需要在回调方法中编写释放资源的代码。
可以在资源紧张的时候，释放UI 使用的资源资：Bitmap、数组、控件资源。

#### 1.2 OnLowMemory


OnLowMemory是Android提供的API，在系统内存不足，所有后台程序（优先级为background的进程，不是指后台运行的进程）都被杀死时，系统会调用OnLowMemory。系统提供的回调有：
Application.onLowMemory()
Activity.OnLowMemory()
Fragement.OnLowMemory()
Service.OnLowMemory()
ContentProvider.OnLowMemory()

除了上述系统提供的API，还可以自己实现ComponentCallbacks，通过API注册，这样也能得到OnLowMemory回调。例如：

``` java
 public static class MyCallback implements ComponentCallbacks {
   @Override
   public void onConfigurationChanged(Configuration arg) {
   }
   @Override
   public void onLowMemory() {
   //do release operation
   }
```
然后，通过Context.registerComponentCallbacks ()在合适的时候注册回调就可以了。通过这种自定义的方法，可以在很多地方注册回调，而不需要局限于系统提供的组件。
onLowMemory 当后台程序已经终止资源还匮乏时会调用这个方法。好的应用程序一般会在这个方法里面释放一些不必要的资源来应付当后台程序已经终止，前台应用程序内存还不够时的情况。

#### 1.3 OnTrimMemory


OnTrimMemory是Android 4.0之后提供的API，系统会根据不同的内存状态来回调。系统提供的回调有：
Application.onTrimMemory()
Activity.onTrimMemory()
Fragement.OnTrimMemory()
Service.onTrimMemory()
ContentProvider.OnTrimMemory()
OnTrimMemory的参数是一个int数值，代表不同的内存状态：
TRIM_MEMORY_COMPLETE：内存不足，并且该进程在后台进程列表最后一个，马上就要被清理
TRIM_MEMORY_MODERATE：内存不足，并且该进程在后台进程列表的中部。
TRIM_MEMORY_BACKGROUND：内存不足，并且该进程是后台进程。
TRIM_MEMORY_UI_HIDDEN：内存不足，并且该进程的UI已经不可见了。 
以上4个是4.0增加
TRIM_MEMORY_RUNNING_CRITICAL：内存不足(后台进程不足3个)，并且该进程优先级比较高，需要清理内存
TRIM_MEMORY_RUNNING_LOW：内存不足(后台进程不足5个)，并且该进程优先级比较高，需要清理内存
TRIM_MEMORY_RUNNING_MODERATE：内存不足(后台进程超过5个)，并且该进程优先级比较高，需要清理内存 
以上3个是4.1增加
系统也提供了一个ComponentCallbacks2，通过Context.registerComponentCallbacks()注册后，就会被系统回调到。

#### 1.4 OnLowMemory和OnTrimMemory的比较

1，OnLowMemory被回调时，已经没有后台进程；而onTrimMemory被回调时，还有后台进程。
2，OnLowMemory是在最后一个后台进程被杀时调用，一般情况是low memory killer 杀进程后触发；而OnTrimMemory的触发更频繁，每次计算进程优先级时，只要满足条件，都会触发。
3，通过一键清理后，OnLowMemory不会被触发，而OnTrimMemory会被触发一次。

使用举例：
``` java
 @Override
 public void onTrimMemory(int level) {
 Log.e(TAG, " onTrimMemory ... level:" + level); 
 }
 @Override
 public void onLowMemory() { 
 Log.e(TAG, " onLowMemory ... "); 
 }
```
### 2.系统回调优化
#### 2.1 回调原理：
在Application、 Activity、Fragement、Service、ContentProvider中都可以重写回调方法，对OnLowMemory/OnTrimMemory进行回调，在回调方法中实现资源释放的实现。
以Activity为例，在Activity源码中能够看到对于onTrimMemory的定义，因此在回调的时候重写方法即可。
``` java
 public void onTrimMemory(int level) {
 if (DEBUG_LIFECYCLE) Slog.v(TAG, "onTrimMemory " + this + ": " + level);
 　　mCalled = true;
 　　mFragments.dispatchTrimMemory(level);
}
``` 
2.2 释放资源：
在onTrimMemory释放资源，释放图片、数组、缓存等资源。

``` java
 @Override
 public void onTrimMemory(int level) {
 // TODO Auto-generated method stub
 DLog.d(" onTrimMemory ... level:" + level);
 
 switch(level)
 
  {
  case TRIM_MEMORY_UI_HIDDEN: 
 //释放资源
 /*编写释放资源代码*/
 }
 
 break;
 }
 super.onTrimMemory(level);
 }
``` 
下面是释放Bitmap的示例代码片段：
``` java
1 // 先判断是否已经回收
2 if(bitmap != null && !bitmap.isRecycled()){ 
3 // 回收并且置为null
4 bitmap.recycle(); 
5 bitmap = null; 
6 } 
7 System.gc();
复制代码
list占用方法：
list.clear();然后在置空。
```
 

### 3.实现ComponentCallbacks


OnLowMemory除了上述系统提供的API，还可以自己实现ComponentCallbacks，通过API注册，这样也能得到OnLowMemory回调。例如：

``` java
 public static class ViewComponentCallbacks implements ComponentCallbacks {
 @Override
  public void onConfigurationChanged(Configuration arg) {
  }
  
  @Override
  public void onLowMemory() {
  //do release operation
  }
 }
```
注册自定义的回调类：

 ViewComponentCallbacks callBacks =new ViewComponentCallbacks();
 this.registerComponentCallbacks( callBacks );
回调之后，即可进行重写：

``` java
 @Override
 public void onLowMemory() {
 // TODO Auto-generated method stub
 //释放资源的方法
 super.onLowMemory();
 }
```
 

Android onTrimMemory方法的一些疑惑？
当你的app进程正在被cached时，你可能会接受到从onTrimMemory()中返回的下面的值之一:

TRIM_MEMORY_BACKGROUND: 系统正运行于低内存状态并且你的进程正处于LRU缓存名单中最不容易杀掉的位置。尽管你的app进程并不是处于被杀掉的高危险状态，系统可能已经开始杀掉LRU缓存中的其他进程了。你应该释放那些容易恢复的资源，以便于你的进程可以保留下来，这样当用户回退到你的app的时候才能够迅速恢复。
TRIM_MEMORY_MODERATE: 系统正运行于低内存状态并且你的进程已经已经接近LRU名单的中部位置。如果系统开始变得更加内存紧张，你的进程是有可能被杀死的。
TRIM_MEMORY_COMPLETE: 系统正运行与低内存的状态并且你的进程正处于LRU名单中最容易被杀掉的位置。你应该释放任何不影响你的app恢复状态的资源。
这个地方讲的进程正在被cache 是什么意思呢？是指我的进程 还没有结束 正在内存中 但是因为没有执行任何操作 不占有cpu的意思吗？
此外当我们监听onTrimMemory回调方法的时候，假设我的缓存里最大的部分就是bitmap，请问就直接把bitmap从内存里抹去就可以了么？ 那如果这么做，在进程重新被加载到前台来的时候，我怎么迅速恢复这些bitmap资源？

 
### 答疑
#### 答疑1：

android进程回收主要涉及两个组件：ActivityManagerService（Ams）和lowmemorykiller。当手机内存不足时，lowmemorykiller就要开始杀进程了。但是lowmemorykiller呢只知道进程占用的内存大小，不知道进程对用户的重要性。Ams则负责管理android四大组件，当然知道进程的重要性了，所以呢还需要与Ams充分交换意见。



当app状态发生改变时，比如退到后台时，ams会对app的进程计算出一个值，即Oomadj（ams#computeOomAdjLocked），然后把这个值传给linux内核，lowmemorykiller呢就可以拿到这个值了，lowmemorykiller则就有了所有app进程的Oomadj值，即进程对用户的重要程度。当手机内存不足时，lowmemorykiller就有了足够的信息决定干掉哪个进程了。那，lowmemorykiller决定干掉哪个进程呢？这个要根据手机还有多少空闲内存，比如还有16MB空闲内存，如下lowmemorykiller.c

``` java
static short lowmem_adj[6] = {
0,
1,
6,
12,
};
static int lowmem_adj_size = 4;
static int lowmem_minfree[6] = {
3 * 512,    /* 6MB */
2 * 1024,    /* 8MB */
4 * 1024,    /* 16MB */
16 * 1024,    /* 64MB */
};
``` 
16MB在lowmem_minfree第三个位置，取lowmem_adj第三个位置即6，ok所有Oomadj大于6的进程就被选中了。而lowmemorykiller不是把这些选中的进程都干掉，而是先干掉oomAdj最大而且占用内存最大的进程。如下lowmemorykiller.c#lowmem_scan：

``` java
for_each_process(tsk) {  
...
oom_score_adj = p->signal->oom_score_adj;
if (oom_score_adj < min_score_adj) {
task_unlock(p);
continue;
}
tasksize = get_mm_rss(p->mm);
task_unlock(p);
if (tasksize <= 0)
continue;
if (selected) {
if (oom_score_adj < selected_oom_score_adj)
continue;
if (oom_score_adj == selected_oom_score_adj &&
tasksize <= selected_tasksize)
continue;
}
selected = p;
selected_tasksize = tasksize;
selected_oom_score_adj = oom_score_adj;
}
```
总之一句话：进程对用户越不重要（Oomadj值就越大），占用内存越大，进程就越容易被干掉。

应对策略：

1 ams计算出一个危险的Oomadj值会调用onTrimMemory通知app，此时app应该把不重要的内存释放掉，只要比友商app占用的内存小被lowmemorykiller干掉的概率就小。

2 当然我们也可以先把忧伤的app干掉。。。帮用户释放内存（开玩笑啦）

3 如果我们的app能预置到手机中，并且manifest设置为persistence（Oomadj=0），或者coreserver则不必担心被lowmemorykiller干掉了。不过还有可能被linux层的oomkiller干掉的。当然这也是某些手机预置一堆app导致越用越慢的一个原因，所以我们尽量把这些预置app卸载掉。

 

#### 答疑2：

理解进程和组件之间的关系

一般来说，你的App在启动的时候，系统都会为你的APP创建一个进程。

比如我有一个App，其包名为com.performance.liferecord

当这个App启动的时候，我们可以通过ps命令来看其对应的进程 信息：

结果如下：

USER PID PPID VSIZE RSS WCHAN PC NAME



可以看到系统为这个进程分配的PD是32452，那么 PPID是什么呐？

使用PS命令来看：adb shell ps | grep 379



可以看到PPID的NAME是zygote64，也就是说32452这个进程是从Android的zygote64这个进程fork过来的。

那么 组件又是怎么回事呐？我们常说四大组件，Activity、Service、Broadcast、ContentProvider。这些个组件都是依附于一个进程来运行的，一个进程可以有多个Activity、Service等，也可以一个组件都没有。

 

Cache状态

当你的应用程序到了后台之后，会进入Cached状态，这时候进程还存在，但是组件是否存在就不一定了。

不过一般我们认为，当你按Back键回到桌面之后，如果Activity没有被释放，那么我们认为了内容泄漏，这个有兴趣可以自己看看。

以上面我提到的com.performance.liferecord这个应用为例子，当我启动这个应用之后，我们可以使用命令adb shell dumpsys meminfo来查看整个系统的状态，此时com.performance.liferecord这个进程状态为Foreground:



Foreground意味着此时你的进程处于前台进程，你的程序的界面可以被用户看到。此时我们使用adb shell dumpsys meminfo com.preformance.liferecord来看这个进程的状态：



可以看到Activity的数量为1（因为我只起了一个Activity）

当我们点击Back键的时候，我们可以使用命令adb shell dumpsys meminfo来查看 整个系统的状态，此时com.performance.liferecord这个进程状态为Cached:



Cached状态标识你的应用进程变成了后台进程，位于LRU list里面。

此时我们使用adb shell dumpsys meminfo com.performance.liferecord来看这个进程的状态：



Activity的数量妥妥变成0了，说明没有内存泄漏。此时你的进程里面，是没有任何组件的。当然也不占用cpu的资源，但是会点内存（例子中我们可以看到占用了20706kb）的内存。

将后台进程放到Cached列表里而不是直接杀掉，其好处就是下一次这个进程的某个组件（最常见的就是Activity）启动的时候，不需要再创建新的进程 ，速度杠杠的（比如就应用的热启动）。

onTrimMemory

此外当我们监听onTrimMemory回调方法的时候，假设我的缓存里最大的部分就是bitmap，请问就直接把bitmap从内存里抹去就可以了么？ 那如果这么做，在进程重新被加载到前台来的时候，我怎么迅速恢复这些bitmap资源？

如答疑1所述：

对于 图片，一般采用三层缓存机制，即内存、文件、网络、释放内存后从文件中读取就好

一般的图片加载库都会提供内存/文件的缓存，即使内存中的Bitmap清除掉了，下一次从文件中读取也会很快的。

至少onTrimMemory的优点，比如你的应用Cache了20MB的Bitmap没有清除，当手机内存不足的时候，如果你没有及时释放 这些内存，那么 很可能你的进程 就被系统给杀掉了，这会你的Bitmap当然也就没了。用户下次启动你的应用的时候，我曹这货又被杀了，这手机系统有毛病吧？

是吧。。。做系统的还得背这个锅！

关于onTrimMemory的使用，可以参考博客http://androidperformance.com/2015/07/20/Android-Performance-Memory-onTrimMemory.html