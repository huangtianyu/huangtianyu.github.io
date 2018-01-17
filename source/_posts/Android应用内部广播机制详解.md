---
title: Android应用内部广播机制详解
date: 2018-01-17 18:07:47
categories: Android
tags: 应用内部广播,LocalBroadcastManager
---
### 1. 简介
通常我们在使用Android广播的时候都会直接将广播注册到系统的AMS当中，由于AMS任务繁忙，一般可能不会立即能处理到我们发出的广播，如果我们使用广播是在应用内的单个进程中使用，则完全可以采用LocalBroadcastManager来处理。LocalBroadcastManager采用的是Handler的消息机制来处理的广播，而注册到系统中的是通过Binder机制实现的，速度是应用内广播要快很多。不过由于Handler的消息机制是为了同一个进程的多线程间进行通信的，因而跨进程时无法使用应用内广播。
#### 1.1 使用
在使用上和普通的Broadcast类似，主要分5步。具体如下：
``` java
//1. 自定义广播接收者
public class LocalReceiver extends BroadcastReceiver {
    public void onReceive(Context context, Intent intent) {
        ...
    }
}
LocalReceiver localReceiver = new LocalReceiver();

//2. 注册广播
LocalBroadcastManager.getInstance(context)
             .registerReceiver(localReceiver, new IntentFilter(“test”));
//4. 发送广播
LocalBroadcastManager.getInstance(context).sendBroadcast(new Intent("test"));
//5. 取消注册广播
LocalBroadcastManager.getInstance(context).unregisterReceiver(localReceiver);
```
自定义广播和普通的广播一样，在注册广播的时候将该广播接受者注册到LocalBroadcatManager中。当发生时也是调用LocalBroadcastManager的sendBroadcast进行发生。同样在不使用时记得取消广播注册。
### 2. LocalBroadcastManager
#### 2.1 初始化
LocalBroadcastManager采用的是单例模式，其构造函数是私有的，获取该类实例的方法是getInstance，具体代码如下：
``` java
  private final Handler mHandler;

    private static final Object mLock = new Object();
    private static LocalBroadcastManager mInstance;

    public static LocalBroadcastManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                mInstance = new LocalBroadcastManager(context.getApplicationContext());
            }
            return mInstance;
        }
    }

    private LocalBroadcastManager(Context context) {
        mAppContext = context;
        //mHandler是主线程的
        mHandler = new Handler(context.getMainLooper()) {

            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_EXEC_PENDING_BROADCASTS:
                        executePendingBroadcasts();//这里去执行广播分发
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
```
在构造函数中创建了一个mHandler，该mHandler关联的是主线程的Looper。即消息处理时都在主线程中处理。
#### 2.2 registerReceiver
``` java
public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        //在注册，取消注册，发送广播的时候都需要先获取mReceivers的锁
        synchronized (mReceivers) {
            //新建一个ReceiverRecord实体表示该receiver及对应的filter
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            //获取receiver对应的filters
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
                //如果该receiver没有对应的filters则，新建一个。
                filters = new ArrayList<IntentFilter>(1);
                mReceivers.put(receiver, filters);
            }
            //将filter放入该receiver对应的filters中
            filters.add(filter);
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    //将action放入mActions中
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
```
注册的时候也就是将receiver自己和对应的filter及action放入到mReceivers和mActions当中。代码比较简单。

#### 2.3 发送广播sendBroadcast
``` java
public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();

            final boolean debug = DEBUG ||
                    ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
            if (debug) Log.v(
                    TAG, "Resolving type " + type + " scheme " + scheme
                    + " of intent " + intent);

            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            if (entries != null) {
                if (debug) Log.v(TAG, "Action list: " + entries);

                ArrayList<ReceiverRecord> receivers = null;
                for (int i=0; i<entries.size(); i++) {
                    ReceiverRecord receiver = entries.get(i);
                    if (debug) Log.v(TAG, "Matching against filter " + receiver.filter);

                    if (receiver.broadcasting) {
                        if (debug) {
                            Log.v(TAG, "  Filter's target already added");
                        }
                        continue;
                    }

                    int match = receiver.filter.match(action, type, scheme, data,
                            categories, "LocalBroadcastManager");
                    if (match >= 0) {
                        if (debug) Log.v(TAG, "  Filter matched!  match=0x" +
                                Integer.toHexString(match));
                        if (receivers == null) {
                            receivers = new ArrayList<ReceiverRecord>();
                        }
                        receivers.add(receiver);
                        receiver.broadcasting = true;
                    } else {
                        if (debug) {
                            String reason;
                            switch (match) {
                                case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                                case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                                case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                                case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                                default: reason = "unknown reason"; break;
                            }
                            Log.v(TAG, "  Filter did not match: " + reason);
                        }
                    }
                }

                if (receivers != null) {
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    return true;
                }
            }
        }
        return false;
    }
```
主要步骤：1.根据Intent的action来查询相应的广播接收者列表；
2.创建相应广播，添加到mPendingBroadcasts队列；
3.发送MSG_EXEC_PENDING_BROADCASTS消息。将消息传给主线程进行处理。
4.主线程mHandler接受到后就由该类的handlerMessage进行处理。在该方法中调用executePendingBroadcasts()进行处理
``` java

    private void executePendingBroadcasts() {
        while (true) {
            BroadcastRecord[] brs = null;
            synchronized (mReceivers) {//注意多线程下的同步
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return;
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);//把待处理的广播转成数组形式
                mPendingBroadcasts.clear();//然后就可以把mPendingBroadcasts清空
            }
            //for循环变量每个接受者，然后调用对应的onReceive
            for (int i=0; i<brs.length; i++) {
                BroadcastRecord br = brs[i];
                for (int j=0; j<br.receivers.size(); j++) {
                    br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
```
处理也很简单，查询相应的变量，找到有多少个接受者，然后调用接受者的onReceive，该调用在主线程中，因而不要做耗时操作。在LocalBroadcastManager中还提供了同步发送广播处理的方法：
``` java
    //使用该方法会立即去让接受者处理广播。
    public void sendBroadcastSync(Intent intent) {
        if (sendBroadcast(intent)) {
            executePendingBroadcasts();
        }
    }
```

#### 2.4 广播的注销
``` java
 public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
            ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            for (int i=0; i<filters.size(); i++) {
                IntentFilter filter = filters.get(i);
                for (int j=0; j<filter.countActions(); j++) {
                    String action = filter.getAction(j);
                    ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    if (receivers != null) {
                        for (int k=0; k<receivers.size(); k++) {
                            if (receivers.get(k).receiver == receiver) {
                                receivers.remove(k);
                                k--;
                            }
                        }
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
```
注销广播也很简单，找到注册时候添加到List中的变量，然后remove掉。注意要讲mReceivers,mActions里面保存的都remove了。

### 3.总结
和普通广播比，应用内广播安全，速度快。缺点是只能在应用的一个进程中使用，不能跨进程使用。