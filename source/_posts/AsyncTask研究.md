---
title: AsyncTask研究
date: 2018-01-03 22:32:22
categories: Android
tags: Android学习
---

### 1. AsyncTask概述
在Android平台上，要执行异步工作时，我们常常会用到AsyncTask。这个类可以算是历史悠久，早在Android 1.5版时，它就存在了。

AsyncTask的使用方法比较简单，无非是创建一个AsyncTask派生类对象，重写其doInBackground()函数，然后在合适世纪调用这个对象的execute()或executeOnExecutor()函数即可。
``` java
private static class MyTask extends AsyncTask<Void, Void, Void> ｛
    // . . . . . .
    @Override
    public Void doInBackground(Void... param) {
        //. . . . . .
        return null;
    }
｝
private class TestClickListener implements View.OnClickListener {
    public void onClick(View v) {
        switch (v.getId()) {
        case R.id.scan_btn:
            testTask();
            break;
        . . . . . .
    }     
    private void testTask() {
        for (int i = 0; i < 5; i++) {
            MyTask t = new MyTask(i+100);
            t.execute();    // 调用execute()即可
        }
    }
```
一般情况下，我们会像上面代码中这样调用AsyncTask的execute()函数，这样，投入执行的task会串行执行。不过，有时候我们也希望task们可以并行执行，此时只需把execute()换成executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR)即可。

### 2. AsyncTask的内部机制
AsyncTask本身是个抽象的泛型基类，正如前面所说，在实际使用时，我们必须定义它的派生类，并在实现AsyncTask派生类时，重写其doInBackground()成员函数。

AysncTask的声明如下：
【frameworks/base/core/java/android/os/AsyncTask.java】

public abstract class AsyncTask<Params, Progress, Result>
作为一种异步执行的任务，AsyncTask是依靠内部的线程池来完成任务调度的。大体上说，AsyncTask内部搞了两个静态的执行器，分别表示成AsyncTask.THREAD_POOL_EXECUTOR 和 AsyncTask.SERIAL_EXECUTOR，前者是可并行执行的执行器（线程池），后者是串行执行的执行器（线程池）。

AsyncTask的构造函数如下：
``` java
/**
 * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
 */
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
            return postResult(result);
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()", e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```
构造函数的注释中说的很明确，必须在UI线程里构造AsyncTask对象。而且构造函数里为两个重要的成员：mWorker和mFuture赋了值，这个我们后文再细说。

#### 2.1 AsyncTask的execute()
我们先回过头看前文曾经提到的AsyncTask的execute()函数，其代码如下：
【frameworks/base/core/java/android/os/AsyncTask.java】
``` java
MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```
因为params参数是可变长参数，所以execute()可以接受0到n个参数。注意，execute()和executeOnExecutor()都必须在UI线程里调用。

execute()只是简单地调用executeOnExecutor()而已，它传递的静态变量sDefaultExecutor引用的就是串行执行器AsyncTask.SERIAL_EXECUTOR：
<code>
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
</code>
executeOnExecutor()的代码截选如下：
``` java
MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    . . . . . .
    . . . . . .
    mStatus = Status.RUNNING;
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);  // 注意，mFuture本身实现了Runnable接口
    return this;
}
```
也就是说，最终还是在调用执行器的execute()函数，只不过会把一个mFuture委托给执行器去回调。

默认情况下使用的串行执行器类是SerialExecutor，它的代码如下：
``` java
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) { // 参数r一般就是mFuture引用的对象
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```
从代码里可以看到，所谓的串行执行器内部，其实也是在复用THREAD_POOL_EXECUTOR，只不过利用对mActive的判断，把调用的流程改成串行的了。

SerialExecutor内部使用的是java.util.ArrayDeque队列，它的poll()函数可以检索并移除此队列的头部，如果返回null，则表示此队列已经取空了。每次摘取一个列头，并记录在mActive变量里，然后交给THREAD_POOL_EXECUTOR来处理。

ThreadPoolExecutor是java提供的线程池实现。总之，线程池会在后续的某个时刻，回调上面插入的Runnable对象的run()。在executeOnExecutor()函数里，我们已经看到向执行器添加了AsynctTask的mFuture成员，而mFuture本身实现了Runnable接口，以后回调就是回调mFuture的run()函数。

#### 2.2 AsyncTask和线程池的协作
##### 2.2.1 AsyncTask里的mFuture
AsyncTask的mFuture非常重要，它的定义如下：

private final FutureTask<Result> mFuture;
类型为FutureTask，其实现可以参考JDK里的代码：
【java/util/concurrent/FutureTask.java】
<code>
public class FutureTask<V> implements RunnableFuture<V> 
</code>
【java/util/concurrent/FunnableFuture.java】
<code>
public interface RunnableFuture<V> extends Runnable, Future<V>
</code>
在前文列出AsyncTask构造函数时，我们已经看到mFuture的创建代码了，注意，在创建FutureTask对象时，传入了mWorker，它会被记入mFuture内部（如果分析JDK的代码，可以看到大体上就是记入mFuture.sync.callable了）。后续在被线程池执行时，这个mWorker才是最核心的对象。

欲了解详情，我们先得看看AsyncTask机制运用的线程池。在AsyncTask类里这样定义线程池成员的：
``` java
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);
public static final Executor THREAD_POOL_EXECUTOR;
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```
注意，线程池都是记在静态变量里的，它的生命期和进程的生命期基本一致。

细心的同学还记得，前文在定义AsyncTask派生类时，我们写的是private static class，大家不要忘记加static，否则就是写了一个普通内嵌类，而普通内嵌类对象内部会隐式地引用其外部类，这样当我们的task对象记入线程池后，就有可能导致task的外部类（很有可能是个Activity或Service）对象在较长时间内都不能被垃圾回收机制回收，从而导致内存泄漏。

本文的重点并不想太深入线程池的内部机理，我们只做必要的探讨即可。我们大体上只需知道线程池里的线程会执行FutureTask的run()函数即可。而FutureTask的run()代码如下：
【java/util/concurrent/FutureTask.java】
``` java
public void run() {
    sync.innerRun();
}
```
而FutureTask.Sync的innerRun()代码如下：
``` java
void innerRun() {
    if (!compareAndSetState(READY, RUNNING))
        return;

    runner = Thread.currentThread();
    if (getState() == RUNNING) { // recheck after setting thread
        V result;
        try {
            result = callable.call();  // 这一步间接调用到AsyncTask的doInBackground()。
        } catch (Throwable ex) {
            setException(ex);
            return;
        }
        set(result);  // 如果不出异常的话，会对call返回的结果执行set()操作。
    } else {
        releaseShared(0); // cancel
    }
}
```
其中会调用callable.call()，这一步就会间接调用到AsyncTask的doInBackground()。再接下来，如果不出异常的话，会对call()返回的结果执行set()操作。大家还记得前文WorkerRunnable实现的call()函数吗？它最后返回语句为：return postResult(result);现在设置的就是这个postResult对象。

FutureTask的set()函数的代码如下：
【java/util/concurrent/FutureTask.java】
``` java
protected void set(V v) {
    sync.innerSet(v);
}
    void innerSet(V v) {
        for (;;) {
            int s = getState();
            if (s == RAN)
                return;
            if (s == CANCELLED) {
                releaseShared(0);
                return;
            }
            if (compareAndSetState(s, RAN)) {
                result = v;
                releaseShared(0);
                done();
                return;
            }
        }
    }
``` 
结果记录进Sync类的result成员，然后回调FutureTask的done()函数，这也就回调到前文我们看到的AysncTask的mFuture的done()函数了。我们再列一下mFuture的代码：
【frameworks/base/core/java/android/os/AsyncTask.java】
``` java
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()", e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
``` 
done()里面做的无法一些善后处理。
``` java
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}
``` 
##### 2.2.2 AsyncTask里的mWorker
AsyncTask的另一个重要成员是mWorker，

private final WorkerRunnable<Params, Result> mWorker;
除了在executeOnExecutor()里会为mWorker的mParams成员赋值外，AsyncTask一般不会直接操作mWorker。mWorker会间接记录进mFuture。当mFuture被回调时，系统会间接回调mWorker的call()成员函数，而这个call()函数是整个AsyncTask的核心行为。

现在我们可以画一张AsyncTask的示意图：



其实，当一个AsyncTask被安插进线程池时，线程池主要关心的是其mFuture成员引用的FutureTask。所以我们可以画出如下示意图：



当回调发生时，最终间接执行到mWorker成员的call()函数，在介绍AsyncTask的构造函数时，我们已经见过该函数的代码，现在再列一遍：
``` java
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND); // 设为后台线程
            //noinspection unchecked
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();   // ???
            return postResult(result);
        }
    };
``` 
看到了吗，当线程池里的某个线程回调到上面的call()函数时，会先把线程优先级设置为“后台线程”，然后会调用doInBackground()函数。大家还记得吧，前文说过我们在实现一个AsyncTask派生类时，主要重写的就是这个doInBackground()函数，现在终于派上用场了。

上面代码中还调用了一个不常见的函数：Binder.flushPendingCommands()。这个函数对应的注释是这样说的：（本函数）会将所有在当前线程里挂起的“Binder命令”扔回内核驱动。一般可以在执行那些有可能阻塞较长时间的操作之前调用一下该函数，这样可以确保挂起的对象引用被及时释放，避免“持有执行对象的进程”占据比“实际需要持有的时间”更长的时间。所以，我判断此处的调用多少有点儿问题，也许更合理的调用地方是在doInBackground()一句之前。

##### 2.2.3 UI线程和AsyncTask工作线程之间的协作
回调的call()函数最终还会通过postResult()，发回一条MESSAGE_POST_RESULT消息。postResult()的代码如下：
``` java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
``` 
此处的getHandler()得到的实际是一个可向UI线程发送消息的handler（即AsyncTask的静态成员sHandler）。getHandler()的代码如下：
``` java
private static Handler getHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler();
        }
        return sHandler;
    }
}
``` 
这里搞了个类似单例的sHandler，类型为InternalHandler：
``` java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());    // 用于向UI线程发送消息！
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
从InternalHandler的构造函数可以看到，postResult()最终就是向UI线程发回MESSAGE_POST_RESULT消息的。

当UI线程最终处理MESSAGE_POST_RESUTL消息时，会调用AsyncTask的finish()。
``` java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
另一方面，用户在编写doInBackground()时，还可以在合适时机调用publishProgress()，向UI线程发出MESSAGE_POST_PROGRESS消息。publishProgress()的代码如下：
``` java
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```
这个消息同样被刚刚说到的InternalHandler处理，处理时会回调AsyncTask的onProgressUpdate()。

关于UI线程和执行AsyncTask的线程之间的交互，我们可以画一张示意图如下：



这张图反映了一个AsyncTask对象在运作时，大体上是如何被UI线程和工作线程调用执行的。

##### 2.2.4 AsyncTask的内部状态
细心的读者还会发现，AsyncTask在finish()时会把自己的状态置为Status.FINISHED。简单说来，AsyncTask可以处于3种状态，分别是PENDING、RUNNING、FINISHED。这3种状态的切换很简单，示意图如下：




##### 2.2.5 cancel动作
当然，用户还可以随时中途放弃执行当前任务。不管是在主线程处理MESSAGE_POST_PROGRESS时，还是在工作线程处理doInBackground()时，用户都可以调用cancel()函数。该函数的代码如下：
【frameworks/base/core/java/android/os/AsyncTask.java】
``` java
public final boolean cancel(boolean mayInterruptIfRunning) {
    mCancelled.set(true);
    return mFuture.cancel(mayInterruptIfRunning);
}
``` 
【java/util/concurrent/FutureTask.java】
``` java
public boolean cancel(boolean mayInterruptIfRunning) {
    return sync.innerCancel(mayInterruptIfRunning);
}
```
【java/util/concurrent/FutureTask.java】
``` java
boolean innerCancel(boolean mayInterruptIfRunning) {
    for (;;) {
        int s = getState();
        if (ranOrCancelled(s))
            return false;
        if (compareAndSetState(s, CANCELLED))
            break;
    }
    if (mayInterruptIfRunning) {
        Thread r = runner;
        if (r != null)
            r.interrupt();
    }
    releaseShared(0);
    done();
    return true;
}
```
简单地说，cancel()动作会将mCancelled设为true，这样以后再调用isCancelled()时，就会返回true。前文我们已经看过AsyncTask的finish()的代码，现在再列一下：
``` java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
可以看到，如果该任务是被用户cancel的，那么finish时执行的会是onCancelled()，而不是onPostExecute()。另外，为了确保在用户cancel任务之后，该任务能真的快速退出，我们应该在doInBackground()里周期性地检查一下isCancelled()的返回值，一旦发现，就立即退出。

### 3 小结
关于AsyncTask的知识，我们就先说这么多。现在大体总结一下：
1）使用AsyncTask时，主要是重写其派生类的doInBackground()，而且该函数会在线程池的某个工作线程里被回调的；
2）必须在UI线程调用AsyncTask的execute()或executeOnExecutor()；
3）可以在doInBackground()里的合适时机调用publishProgress()，向UI线程通知工作进展；
4）可以随时调用cancel()，放弃执行任务。