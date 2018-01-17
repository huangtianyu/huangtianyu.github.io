---
title: SharedPreferences源码详解
date: 2018-01-17 11:59:33
categories: Android
tags: SharedPreferences Editor
---
### 1.简介
写这篇博客目的在于巩固自己对SharedPreferences的理解。SharePreferences是Android系统提供的轻量级数据存储方案，主要基于键值对方式保存数据，真实的数据是保存在/data/data/packageName/shared_pref/目录下面的。可以保存多种数据到该文件中，以下是一个简单的Sharepreference文件。
``` xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="btest" value="true" />
    <string name="stest">string</string>
    <int name="itest" value="999" />
    <long name="ltest" value="1516358782" />
    <int name="itest_1" value="2" />
</map>
```
从文件中可以看出就是采用简单xml方式进行保存的。
#### 1.1使用实例
```
SharedPreferences preferences = context.getSharedPreferences("share", Context.MODE_PRIVATE);
SharedPreferences.Editor editor = preferences.edit();
editor.putBoolean("btest", true);
editor.putString("stest", "string test");
//editor.apply();//异步保存
editor.commit();//同步保存
```
#### 1.2基本结构
这里借用Gityuan博客中的类继承图
![Sharepreference架构图](https://s1.ax1x.com/2018/01/17/pDZT58.jpg)
在Sharepreference中，Sharepreference和Editor只是两个接口，在这两个接口中定义了一个普通的键值对存储的数据一些常用的操作。然后具体你想把这键值对存哪，你可以自己定义相应的文件或数据库，甚至你可以写个保存到网络中去。在Android系统中给出的是采用xml方式存在xml的文件中，具体实现类是SharepreferenceImpl和SharepreferenceImpl.EditorImpl。同时在ContextImpl中有Sharepreference的对应内存中的数据。
### 2.Sharepreference源码分析
#### 2.1获取Sharepreference
Activity.java
```
    public SharedPreferences getPreferences(int mode) {
        return getSharedPreferences(getLocalClassName(), mode);
    }
    
        @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        return mBase.getSharedPreferences(name, mode);
    }
```
Context采用的是装饰模式，其中正在干活的是ContextImpl，mBase即为ContextImpl，具体代码如下：
ContextImpl.java
``` java
//ContextImpl类中的静态Map声明，全局的一个sSharedPrefs
private static ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>> sSharedPrefs;

    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        SharedPreferencesImpl sp; 
        synchronized (ContextImpl.class) {
            if (sSharedPrefs == null) { //静态变量，全局唯一
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }

            final String packageName = getPackageName();//通过包名找到对应的prefs集合
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }

            // At least one application in the world actually passes in a null
            // name.  This happened to work because when we generated the file name
            // we would stringify it to "null.xml".  Nice.
            if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                    Build.VERSION_CODES.KITKAT) {
                if (name == null) {
                    name = "null";
                }
            }

            sp = packagePrefs.get(name);//这里获取sp
            if (sp == null) {//如果为空，则构建一个sp，并将它放入packagePrefs里面
                File prefsFile = getSharedPrefsFile(name);//正在获取文件的地方
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
        }
        //下面是为了跨进程使用Sharepreference的，跨进程使用也就是重新装载一次sharepreference
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }

```
正在保存的文件获取是getSharedPrefsFile(name)，代码如下：
``` java
    @Override
    public File getSharedPrefsFile(String name) { //文件以.xml结尾
        return makeFilename(getPreferencesDir(), name + ".xml");
    }
    private File makeFilename(File base, String name) {
        if (name.indexOf(File.separatorChar) < 0) { //name中不能存在文件路径分隔符
            return new File(base, name);
        }
        throw new IllegalArgumentException(
                "File " + name + " contains a path separator");
    }
    private File getPreferencesDir() {//sharepreference文件的目录/data/data/{包名}/shared_prefs
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDirFile(), "shared_prefs");
            }
            return mPreferencesDir;
        }
    }
```
在ContextImpl中存在一个静态的sSharedPrefs，通过它来获取对应应用的prefs，在通过prefs找到对应名称的Sharepreference的引用。在系统中共用一个sSharedPrefs，每个应该在获取sp的时候都会将创建后sp加入到sSharedPrefs中以便后续进行访问。
#### 2.2 SharepreferenceImpl
    从上面我们可以看到我们要获取的是Sharepreference，但是返回的是SharepreferenceImpl，这就赤裸裸的告诉我们SharepreferenceImpl是Sharepreference接口的实现类，具体代码如下：
``` java

final class SharedPreferencesImpl implements SharedPreferences {
    private static final String TAG = "SharedPreferencesImpl";
    private static final boolean DEBUG = false;

    // Lock ordering rules:
    //  - acquire SharedPreferencesImpl.this before EditorImpl.this
    //  - acquire mWritingToDiskLock before EditorImpl.this

    private final File mFile;
    private final File mBackupFile;
    private final int mMode;

    private Map<String, Object> mMap;     // guarded by 'this'
    private int mDiskWritesInFlight = 0;  // guarded by 'this'
    private boolean mLoaded = false;      // guarded by 'this'
    private long mStatTimestamp;          // guarded by 'this'
    private long mStatSize;               // guarded by 'this'

    private final Object mWritingToDiskLock = new Object();
    private static final Object mContent = new Object();
    private final WeakHashMap<OnSharedPreferenceChangeListener, Object> mListeners =
            new WeakHashMap<OnSharedPreferenceChangeListener, Object>();

    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file); //创建临时备份文件，这样写入失败的时候就用这个备份的还原
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();//异步加载文件内容到内存
    }

    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                //由于是多线程加载的时候注意同步处理
                synchronized (SharedPreferencesImpl.this) {
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
...
  private static File makeBackupFile(File prefsFile) {
        return new File(prefsFile.getPath() + ".bak");
    }
...
}
```
在获取sp的时候，如果通过sSharedPrefs获取为空就会先创建一个sp，在new SharepreferenceImpl的时候，在构造函数中最后就会异步加载文件到内存，异步开启一个线程后就调用loadFromDiskLocked()函数进行加载：
SharepreferenceImpl.java
``` java

    private void loadFromDiskLocked() {
        if (mLoaded) {//加载过了就返回
            return;
        }
        if (mBackupFile.exists()) {//如果存在备份就直接使用备份
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);//利用XmlUtils进行解析
                } catch (XmlPullParserException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (FileNotFoundException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
        }
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<String, Object>();
        }
        notifyAll();//没加载完前，所有操作（get等）都会等待加载完成，加载完成后通知其他操作可以进行操作了。
    }
```
一旦加载完成后，就会notifyAll()，我们先看下get的操作
``` java

    public Map<String, ?> getAll() {
        synchronized (this) {
            awaitLoadedLocked();
            //noinspection unchecked
            return new HashMap<String, Object>(mMap);
        }
    }

    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (this) {
            awaitLoadedLocked();//没有加载就阻塞等待
            String v = (String)mMap.get(key);//加载完成了就直接去内存中的值，记住是从内存中取。不会再次读取文件总内容。
            return v != null ? v : defValue;
        }
    }
...
    private void awaitLoadedLocked() {//这里判断是否已经加载文件到内存了，没有加载就会阻塞等待
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```
在get数据时，首先判断文件是否加载到内存，然后就直接读取内存中的值，这里可以看出一旦装载了，那么读取的速度就很快。
#### 2.3 EditorImpl
上面SharepreferenceImpl是实现了get操作，真正的写入是Editor接口来完成的，而EditorImpl是具体的实现类。其代码如下：
``` java
    public final class EditorImpl implements Editor {
        private final Map<String, Object> mModified = Maps.newHashMap();
        private boolean mClear = false;
        //从这里可以看出每次存的时候都是先存入内存中的mModified变量中
        public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
        public Editor putStringSet(String key, @Nullable Set<String> values) {
            synchronized (this) {
                mModified.put(key,
                        (values == null) ? null : new HashSet<String>(values));
                return this;
            }
        }
        public Editor putInt(String key, int value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
...
        public Editor remove(String key) {
            synchronized (this) {
                mModified.put(key, this);
                return this;
            }
        }

        public Editor clear() {
            synchronized (this) {
                mClear = true;
                return this;
            }
        }

        public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }

    }
```
首先查看下存入的代码：
```
        //从这里可以看出每次存的时候都是先存入内存中的mModified变量中
        public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
```
存入的时候首先获取同步锁，然后将存入的数据放入EditorImpl中的一个mModified变量中，也就是存入的时候并没有放入Sharepreference中，只有在使用了apply或者commit后才真正存入。
下面来看看commit操作：
``` java
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();//步骤1
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);//步骤2
            try {
                mcr.writtenToDiskLatch.await();//等待写入完成
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);//步骤3
            return mcr.writeToDiskResult;步骤4
        }
```
步骤1：
```
        // Returns true if any changes were made 真正存入文件中
        private MemoryCommitResult commitToMemory() {
            MemoryCommitResult mcr = new MemoryCommitResult();
            synchronized (SharedPreferencesImpl.this) {
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mcr.mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;

                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    mcr.keysModified = new ArrayList<String>();
                    mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }

                synchronized (this) {
                    if (mClear) {
                        if (!mMap.isEmpty()) {
                            mcr.changesMade = true;
                            mMap.clear();
                        }
                        mClear = false;
                    }

                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        //删除一些需要删除的数据
                        if (v == this || v == null) {
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            //将变化的数据放入SharepreferenceImpl的mMap中
                            mMap.put(k, v);
                        }

                        mcr.changesMade = true;
                        if (hasListeners) {
                            mcr.keysModified.add(k);
                        }
                    }
                    //变化的数据都加入了mMap后就可以清除mModified内容了。
                    mModified.clear();
                }
            }
            //返回封装mMap的MemoryCommitResult数据
            return mcr;
        }
```
步骤2：
```
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

      final boolean isFromSyncCommit = (postWriteRunnable == null);//如果postWriteRunnable为null就是同步

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {//同步就直接运行写入数据writeToDiskRunnable
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
    
// 写入数据
    private void writeToFile(MemoryCommitResult mcr) {
        // Rename the current file so it may be used as a backup during the next read
        if (mFile.exists()) {
            if (!mcr.changesMade) {
                // If the file already exists, but no changes were
                // made to the underlying map, it's wasteful to
                // re-write the file.  Return as if we wrote it
                // out.
                mcr.setDiskWriteResult(true);
                return;
            }
            if (!mBackupFile.exists()) {
                if (!mFile.renameTo(mBackupFile)) {
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    mcr.setDiskWriteResult(false);
                    return;
                }
            } else {
                mFile.delete();
            }
        }

        // Attempt to write the file, delete the backup and return true as atomically as
        // possible.  If any exception occurs, delete the new file; next time we will restore
        // from the backup.
        try {
            FileOutputStream str = createFileOutputStream(mFile);
            if (str == null) {
                mcr.setDiskWriteResult(false);
                return;
            }
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
            FileUtils.sync(str);
            str.close();
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (this) {
                    mStatTimestamp = stat.st_mtime;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }
            // Writing was successful, delete the backup file if there is one.
            mBackupFile.delete();
            mcr.setDiskWriteResult(true);
            return;
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }
        // Clean up an unsuccessfully written file
        if (mFile.exists()) {
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        mcr.setDiskWriteResult(false);
    }
```
步骤3通知写入数据发生变化
```

        private void notifyListeners(final MemoryCommitResult mcr) {
            if (mcr.listeners == null || mcr.keysModified == null ||
                mcr.keysModified.size() == 0) {
                return;
            }
            if (Looper.myLooper() == Looper.getMainLooper()) {
                for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
                    final String key = mcr.keysModified.get(i);
                    for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                        if (listener != null) {
                            listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                        }
                    }
                }
            } else {
                // Run this function on the main thread.
                ActivityThread.sMainThreadHandler.post(new Runnable() {
                        public void run() {
                            notifyListeners(mcr);
                        }
                    });
            }
        }
```
步骤4返回写入的结果。
##### 2.3.2 Editor.apply()
代码如下：
```
public void apply() {
    //写数据到内存，返回数据结构
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
        public void run() {
            try {
                //等待写文件结束
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException ignored) {
            }
        }
    };

    QueuedWork.add(awaitCommit);
    //一个收尾的Runnable
    Runnable postWriteRunnable = new Runnable() {
        public void run() {
            awaitCommit.run();
            QueuedWork.remove(awaitCommit);
        }
    };
    //这里postWriteRunnable不为null，所以会在一个新的线程池调运postWriteRunnable的run方法
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    //通知变化
    notifyListeners(mcr);
}
```
apply会将写入放入到一个线程池中操作，这不会阻塞调用的线程。其他的都和commit类似。
QueuedWork.java
```
public class QueuedWork {

    private static final ConcurrentLinkedQueue<Runnable> sPendingWorkFinishers =
       new ConcurrentLinkedQueue<Runnable>();

    public static void add(Runnable finisher) {
        sPendingWorkFinishers.add(finisher);
    }

    public static void remove(Runnable finisher) {
        sPendingWorkFinishers.remove(finisher);
    }

    public static void waitToFinish() {
        Runnable toFinish;
        while ((toFinish = sPendingWorkFinishers.poll()) != null) {
            toFinish.run();
        }
    }

    public static boolean hasPendingWork() {
        return !sPendingWorkFinishers.isEmpty();
    }
}
```
### 总结
apply 与commit的对比

apply没有返回值, commit有返回值能知道修改是否提交成功
apply是将修改提交到内存，再异步提交到磁盘文件; commit是同步的提交到磁盘文件;
多并发的提交commit时，需等待正在处理的commit数据更新到磁盘文件后才会继续往下执行，从而降低效率; 而apply只是原子更新到内存，后调用apply函数会直接覆盖前面内存数据，从一定程度上提高很多效率。
获取SP与Editor:

getSharedPreferences()是从ContextImpl.sSharedPrefsCache唯一的SPI对象;
edit()每次都是创建新的EditorImpl对象.
优化建议:

强烈建议不要在sp里面存储特别大的key/value, 有助于减少卡顿/anr
请不要高频地使用apply, 尽可能地批量提交;commit直接在主线程操作, 更要注意了
不要使用MODE_MULTI_PROCESS;
高频写操作的key与高频读操作的key可以适当地拆分文件, 由于减少同步锁竞争;
不要一上来就执行getSharedPreferences().edit(), 应该分成两大步骤来做, 中间可以执行其他代码.
不要连续多次edit(), 应该获取一次获取edit(),然后多次执行putxxx(), 减少内存波动; 经常看到大家喜欢封装方法, 结果就导致这种情况的出现.
每次commit时会把全部的数据更新的文件, 所以整个文件是不应该过大的, 影响整体性能;

### 参考
[Gityuan博客](http://gityuan.com/2017/06/18/SharedPreferences/)
[工匠若水](http://blog.csdn.net/yanbober/article/details/47866369)