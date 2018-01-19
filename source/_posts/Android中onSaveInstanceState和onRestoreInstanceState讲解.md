---
title: Android中onSaveInstanceState和onRestoreInstanceState讲解
date: 2018-01-19 18:21:47
categories: Android
tags: onSaveInstanceState
---

在Android系统中，有时系统可能因为系统资源不够而杀死(kill)某些Activity，在kill Activity之前会调用 onSaveInstanceState来保存一些状态信息(当然也可以保存其他信息)，当再次回到该Activity时，系统会调用onRestoreInstanceState来恢复数据。
     下面先讲一下onSaveInstance的调用时机，也就是会在什么情况下被调用。onSaveInstance不是Activity正常生命周期里面的函数。在Google API文档上是这样介绍的： Android calls onSaveInstanceState() before the activity becomesvulnerable to being destroyed by the system, but does not bother calling itwhen the instance is actually being destroyed by a user action (such aspressing the BACK key)。也就是当某个activity变得“容易”被系统销毁时，该activity的onSaveInstanceState就会被执行，除非该activity是被用户主动销毁的该方法不会调用，例如当用户按BACK键的时候。“容易”包含以下几种情况：
1、当用户按下HOME键时。当按下HOME键后，由于系统不知道用户会新开多少程序，因而这个Activity可能会由于系统资源不足而被系统kill，因为在onPause之后会调用onSaveInstance来保存数据。
2、长按HOME键，选择运行其他的程序时。分析同上。
3、按下电源按键（关闭屏幕显示）时。分析同上。
4、从activity A中启动一个新的activity时。分析同上。
5、屏幕方向切换时，例如从竖屏切换到横屏时。在屏幕切换之前，系统会销毁activity A，在屏幕切换之后系统又会自动地创建activity A，所以onSaveInstanceState一定会被执行。

一般调用方式如下：
```java
@Override  
    protected void onSaveInstanceState(Bundle outState) {  
        // TODO Auto-generated method stub  
        //这里保存需要保存的数据  
        String string = edit.getText().toString();  
        outState.putString("test", string);  
        super.onSaveInstanceState(outState);  
    } 
```

系统kill进程是有个先后顺序的：在Google上是这样介绍的[ProcessLifecycle](https://developer.android.com/reference/android/app/Activity.html#ProcessLifecycle),其解释如下：
一般系统杀死进程的顺序是：

　　Android系统会尽力保持应用的进程，但是有时为了给新的进程和更重要的进程回收一些内存空间，它会移除一些旧的进程。

　　为了决定哪些进程留下，哪些进程被杀死，系统根据在进程中在运行的组件及组件的状态，为每一个进程分配了一个优先级等级。

　　优先级最低的进程首先被杀死。

　　这个进程重要性的层次结构有**五个等级**，下面就列出这五种进程，按照重要性来排列，最重要的放在最前。 

1. 前台进程 Foreground process
**前台进程**是用户当前做的事所必须的进程，如果满足下面各种情况中的一种，一个进程被认为是在前台：
1.1 进程持有一个正在与用户交互的Activity（Activity正处于onResume()的状态）。
1.2 进程持有一个Service，这个Service和用户正在交互的Activity绑定。
1.3 进程持有一个Service，这个Service是在前台运行的，即它调用了`**[startForeground()](http://developer.android.com/reference/android/app/Service.html#startForeground(int, android.app.Notification))**`。
1.4 进程持有一个Service，这个Service正在执行它的生命周期回调函数（`[onCreate()](http://developer.android.com/reference/android/app/Service.html#onCreate())`, `[onStart()](http://developer.android.com/reference/android/app/Service.html#onStart(android.content.Intent, int))`, or `[onDestroy()](http://developer.android.com/reference/android/app/Service.html#onDestroy())`）。
1.5 进程持有一个BroadcastReceiver，这个BroadcastReceiver正在执行它的 `[onReceive()](http://developer.android.com/reference/android/content/BroadcastReceiver.html#onReceive(android.content.Context, android.content.Intent))` 方法。
杀死前台进程需要用户交互，因为前台进程的优先级是最高的。

2.可见进程 Visible process
　　如果一个进程不含有任何前台的组件，但是仍然影响着用户在屏幕上可以看到的内容，就是**可见进程**。
　　可见进程满足下列情况之一：
　　1.进程持有一个Activity，这个Activity不在前台，但是仍然被用户可见（处于**onPause()**调用后又没有调用**onStop()**的状态）。
　　这种情况发生在，比如，前台的activity打开了一个对话框，这样activity就会在其后可见。
　　2.进程持有一个Service，这个Service和一个可见的（或者前台的）Activity绑定。
　　可见的进程也被认为是很重要的，一般不会被销毁，除非是为了保证所有前台进程的运行而不得不杀死可见进程的时候。 

3. 服务进程 Service process
　　如果一个进程中运行着一个service，这个service是通过 `[startService()](http://developer.android.com/reference/android/content/Context.html#startService(android.content.Intent))` 开启的，并且不属于上面两种较高优先级的情况，这个进程就是一个**服务进程**。
　　尽管服务进程没有和用户可以看到的东西绑定，但是它们一般在做的事情是用户关心的，比如后台播放音乐，后台下载数据等。 

4. 后台进程 Background process
　　如果进程不属于上面三种情况，但是进程持有一个用户不可见的activity（activity的**onStop()**被调用，但是**onDestroy()**没有调用的状态），就认为进程是一个**后台进程**。
　　后台进程不直接影响用户体验，系统会为了前台进程、可见进程、服务进程而任意杀死后台进程。
　　通常会有很多个后台进程存在，它们会被保存在一个**LRU (least recently used)**列表中，这样就可以确保用户最近使用的activity最后被销毁，即最先销毁时间最远的activity。 

5. 空进程
　　如果一个进程不包含任何活跃的应用组件，则认为是**空进程**。
　　保存这种进程的唯一理由是为了缓存的需要，为了加快下次要启动这个进程中的组件时的启动时间。
　　系统为了平衡进程缓存和底层内核缓存的资源，经常会杀死空进程。 

6. 相关说明
　　1.Android会尽可能地把进程放在高的优先级。
　　比如，一个进程拥有一个可见状态的activity和一个service，这个进程会被认为是可见进程，而不是服务进程。
　　2.一个进程的等级有可能会因为其他进程的依赖而提高，一个进程服务于另一个进程，则它的优先级不会比它服务的进程优先级低。
　　比如，A进程中的一个content provider向B进程中的一个客户提供服务，或A进程中的一个service被绑定在B进程中的一个组件上，则A进程的优先级**至少**和B进程的优先级一样高。
　　3.因为服务进程的优先级比后台进程的优先级高，所以对于一个需要启动一个长时间操作的activity来说，开启一个service比创建一个工作线程的方法更好，尤其是对于操作将很可能超出activity的持续时间时。
　　比如要上传一个图片文件，应该开启一个service来进行上传工作，这样在用户离开activity时工作仍在进行。使用service将会保证操作至少有服务进程的优先级。

总而言之，onSaveInstanceState的调用遵循一个重要原则，即当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，这是系统的责任，因为它必须要提供一个机会让你保存你的数据。
onSaveInstanceState的调用是在onPause（）之后执行的，即：onPause（）—>onSaveInstanceState()–>onStop( );
至于onRestoreInstanceState方法，需要注意的是，onSaveInstanceState方法和onRestoreInstanceState方法“不一定”是成对的被调用的，onRestoreInstanceState被调用的前提是，activity A“确实”被系统销毁了，而如果仅仅是停留在有这种可能性的情况下，则该方法不会被调用，例如，当正在显示activity A的时候，用户按下HOME键回到主界面，然后用户紧接着又返回到activity A，这种情况下activity A一般不会因为内存的原因被系统销毁，故activity A的onRestoreInstanceState方法不会被执行。onRestoreInstanceState的bundle参数会传递到onCreate方法中，你也可以选择在onCreate方法中做数据还原。
onRestoreInstanceState()在onStart() 和 onPostCreate(Bundle)之间调用。
一般调用方式如下：
```
@Override
	protected void onRestoreInstanceState(Bundle savedInstanceState) {
		// TODO Auto-generated method stub
		//在Activity因为系统额崩溃后，会调用该函数进行数据恢复，恢复的数据就是通过onSaveInstanceState保存的数据
		super.onRestoreInstanceState(savedInstanceState);
		if (savedInstanceState!=null) {
			String test = savedInstanceState.getString("test");
			if (test!=null) {
				edit.setText(test);
			}
		}
	}
```
在新版的SDK中，也就是API大于21的版本中新增了如下两个函数：
```
/**
	 * 系统重启后，数据恢复能力。需要api>21
	 */
	@Override
	public void onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState) {
		// TODO Auto-generated method stub
		//这里保存持久化数据，该函数会调用上面一个函数。因而第一个outState会通过上一个函数保存
		String string = edit.getText().toString();
		outPersistentState.putString("test", string);
		super.onSaveInstanceState(outState, outPersistentState);
	}

/** 
     * 系统重启后，数据恢复能力。需要api>21就 
     * 这个方法会调用onRestoreInstanceState(Bundle savedInstanceState)方法。 
     */  
    @Override  
    public void onRestoreInstanceState(Bundle savedInstanceState, PersistableBundle persistentState) {  
        // TODO Auto-generated method stub  
        //系统因而意外重启后会调用该方法进行数据恢复，这个需要在manifest.xml里面注册： android:persistableMode="persistAcrossReboots"  
        super.onRestoreInstanceState(savedInstanceState, persistentState);  
        if (persistentState!=null) {  
            String test = persistentState.getString("test");  
            if (test!=null) {  
                edit.setText(test);  
            }  
        }  
    }  
```
这两个函数主要是为了系统重启后的数据恢复，使用时需要在AndroidManifest.xml里面的activity中添加android:persistableMode="persistAcrossReboots"属性。PersistableBundle和Bundle差不多，是以key-value的形式使用的。具体代码见如下工程。

工程源码：[onSaveInstance测试工程](http://download.csdn.net/detail/hty1053240123/9585928)
