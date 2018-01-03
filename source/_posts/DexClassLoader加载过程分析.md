---
title: DexClassLoader加载过程分析
date: 2017-12-27 18:02:07
categories: Android
tags: Dex类加载
---
在动态加载dex时，均采用的是DexClassLoader类进行加载，网上也有很多对如何加载进行分析的，现结合Android5.1源码进行分析。
#### 实例
先来看一个实例，实例完成动态加载SD卡上的一个jar包，在jar包中Toast一段话。
``` java
public class ToastTest {
    private Context context;
    public ToastTest(Context context){
        this.context = context;
    }
    public void call() {
        Toast.makeText(context, "call method", 0).show();
    }
    public String getData() {
        return "ToastTest";
    }
}
```
将这个Java类打包生成test.jar包，然后放到SD目录下，然后利用DexClassLoader动态加载该jar包。
``` java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        File file = new File(Environment.getExternalStorageDirectory()
                .toString() + File.separator + "test.jar");
        final File optimizedDexOutputPath = getDir("temp", Context.MODE_PRIVATE);
        /*
         * Parameters
         * dexPath    需要装载的APK或者Jar文件的路径。包含多个路径用File.pathSeparator间隔开,在Android上默认是 ":" 
         * optimizedDirectory    优化后的dex文件存放目录，不能为null
         * libraryPath    目标类中使用的C/C++库的列表,每个目录用File.pathSeparator间隔开; 可以为 null
         * parent    该类装载器的父装载器，一般用当前执行类的装载器
         */
        new DexClassLoader(file.getAbsolutePath(),
                optimizedDexOutputPath.getAbsolutePath(), null,
                getClassLoader());
          //利用反射原理去调用
      try {
            Class<?> testClass = classLoader.loadClass("com.demo.ToastTest");
            Constructor<?> istructor = testClass.getConstructor(Context.class);
            Method method = iclass.getMethod("call", null);
            String data = (String) method.invoke(istructor.newInstance(this), null);
            //System.out.println(data);
            Log.d("ToastTest",data);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
上面代码<code>new DexClassLoader(file.getAbsolutePath(),optimizedDexOutputPath.getAbsolutePath(), null, getClassLoader());</code>完成了jar包的加载，下面从该步骤开始分析DexClassLoader是如何加载一个jar包或者dex的。
#### 代码分析
DexClassLoader类文件在\libcore\dalvik\src\main\java\dalvik\system\ DexClassLoader.java文件下
``` java
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code DexClassLoader} that finds interpreted and native
     * code.  Interpreted classes are found in a set of DEX files contained
     * in Jar or APK files.
     *
     * <p>The path lists are separated using the character specified by the
     * {@code path.separator} system property, which defaults to {@code :}.
     *
     * @param dexPath the list of jar/apk files containing classes and
     *     resources, delimited by {@code File.pathSeparator}, which
     *     defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     *     should be written; must not be {@code null}
     * @param libraryPath the list of directories containing native
     *     libraries, delimited by {@code File.pathSeparator}; may be
     *     {@code null}
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```
代码很简单，直接调用了父类的构造方法。
``` java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    /**
     * Constructs an instance.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     * should be written; may be {@code null}
     * @param libraryPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
	...
}
```
父类构造函数也很简单，首先调用了它自己的父类构造函数，然后new DaxPathList。下面先分析下DexPathList的构造函数。
``` java
	/**
     * Constructs an instance.
     *
     * @param definingContext the context in which any as-yet unresolved
     * classes should be defined
     * @param dexPath list of dex/resource path elements, separated by
     * {@code File.pathSeparator}
     * @param libraryPath list of native library directory path elements,
     * separated by {@code File.pathSeparator}
     * @param optimizedDirectory directory where optimized {@code .dex} files
     * should be found and written to, or {@code null} to use the default
     * system directory for same
     */
    public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }
		//前面做了一些检查和异常处理
        this.definingContext = definingContext;
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions);
        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
        this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
    }
```
其中关键代码是makeDexElements
``` java
	/**
     * Makes an array of dex/resource path elements, one per element of
     * the given array.
     */
    private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
                                             ArrayList<IOException> suppressedExceptions) {
        ArrayList<Element> elements = new ArrayList<Element>();
        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        for (File file : files) {
            File zip = null;
            DexFile dex = null;
            String name = file.getName();
			//对每个文件进行处理
            if (file.isDirectory()) {
            //如果是文件直接加入到elements中
                // We support directories for looking up resources.
                // This is only useful for running libcore tests.
                elements.add(new Element(file, true, null, null));
            } else if (file.isFile()){
                if (name.endsWith(DEX_SUFFIX)) {//是dex
                    // Raw dex file (not inside a zip/jar).
                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException ex) {
                        System.logE("Unable to load dex file: " + file, ex);
                    }
                } else {//不是dex。即是压缩文件：jar zip apk
                    zip = file;
                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException suppressed) {
                        /*
                         * IOException might get thrown "legitimately" by the DexFile constructor if
                         * the zip file turns out to be resource-only (that is, no classes.dex file
                         * in it).
                         * Let dex == null and hang on to the exception to add to the tea-leaves for
                         * when findClass returns null.
                         */
                        suppressedExceptions.add(suppressed);
                    }
                }
            } else {
                System.logW("ClassLoader referenced unknown path: " + file);
            }

            if ((zip != null) || (dex != null)) {
                elements.add(new Element(file, false, zip, dex));
            }
        }

        return elements.toArray(new Element[elements.size()]);
    }
```
先看DexPathList内部是如何解析dex文件的，即分析loadDexFile代码
``` java
    /**
     * Constructs a {@code DexFile} instance, as appropriate depending
     * on whether {@code optimizedDirectory} is {@code null}.
     */
    private static DexFile loadDexFile(File file, File optimizedDirectory)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0);
        }
    }
```
如果被优化后的路径是空，那么直接返回new DexFile(file)。optimizedPathFor是判断优化路径，代码如下：
``` java
	/**
     * Converts a dex/jar file path and an output directory to an
     * output file path for an associated optimized dex file.
     */
    private static String optimizedPathFor(File path,
            File optimizedDirectory) {
        /*
         * Get the filename component of the path, and replace the
         * suffix with ".dex" if that's not already the suffix.
         *
         * We don't want to use ".odex", because the build system uses
         * that for files that are paired with resource-only jar
         * files. If the VM can assume that there's no classes.dex in
         * the matching jar, it doesn't need to open the jar to check
         * for updated dependencies, providing a slight performance
         * boost at startup. The use of ".dex" here matches the use on
         * files in /data/dalvik-cache.
         */
        String fileName = path.getName();
        if (!fileName.endsWith(DEX_SUFFIX)) {
            int lastDot = fileName.lastIndexOf(".");
            if (lastDot < 0) {
                fileName += DEX_SUFFIX;
            } else {
                StringBuilder sb = new StringBuilder(lastDot + 4);
                sb.append(fileName, 0, lastDot);
                sb.append(DEX_SUFFIX);
                fileName = sb.toString();
            }
        }
        File result = new File(optimizedDirectory, fileName);
        return result.getPath();
    }
```
真正执行代码优化的是DexFile.loadDex，代码如下：
``` java
	/**
     * Open a DEX file, specifying the file in which the optimized DEX
     * data should be written.  If the optimized form exists and appears
     * to be current, it will be used; if not, the VM will attempt to
     * regenerate it.
     *
     * This is intended for use by applications that wish to download
     * and execute DEX files outside the usual application installation
     * mechanism.  This function should not be called directly by an
     * application; instead, use a class loader such as
     * dalvik.system.DexClassLoader.
     *
     * @param sourcePathName
     *  Jar or APK file with "classes.dex".  (May expand this to include
     *  "raw DEX" in the future.)
     * @param outputPathName
     *  File that will hold the optimized form of the DEX data.
     * @param flags
     *  Enable optional features.  (Currently none defined.)
     * @return
     *  A new or previously-opened DexFile.
     * @throws IOException
     *  If unable to open the source or output file.
     */
    static public DexFile loadDex(String sourcePathName, String outputPathName,
        int flags) throws IOException {

        /*
         * TODO: we may want to cache previously-opened DexFile objects.
         * The cache would be synchronized with close().  This would help
         * us avoid mapping the same DEX more than once when an app
         * decided to open it multiple times.  In practice this may not
         * be a real issue.
         */
        return new DexFile(sourcePathName, outputPathName, flags);
    }
```
也就是直接返回了DexFile，下面看看DexFile构造函数做了哪些事情：
``` java
	/**
     * Opens a DEX file from a given filename, using a specified file
     * to hold the optimized data.
     *
     * @param sourceName
     *  Jar or APK file with "classes.dex".
     * @param outputName
     *  File that will hold the optimized form of the DEX data.
     * @param flags
     *  Enable optional features.
     */
    private DexFile(String sourceName, String outputName, int flags) throws IOException {
        if (outputName != null) {
            try {
                String parent = new File(outputName).getParent();
                if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                    throw new IllegalArgumentException("Optimized data directory " + parent
                            + " is not owned by the current user. Shared storage cannot protect"
                            + " your application from code injection attacks.");
                }
            } catch (ErrnoException ignored) {
                // assume we'll fail with a more contextual error later
            }
        }
		//前面是异常判断
        mCookie = openDexFile(sourceName, outputName, flags);
        mFileName = sourceName;
        guard.open("close");
        //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
    }
```
关键代码:openDexFile
``` java
    /*
     * Open a DEX file.  The value returned is a magic VM cookie.  On
     * failure, an IOException is thrown.
     */
    private static long openDexFile(String sourceName, String outputName, int flags) throws IOException {
        // Use absolute paths to enable the use of relative paths when testing on host.
        return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                 (outputName == null) ? null : new File(outputName).getAbsolutePath(),
                                 flags);
    }
    /*
     * Open a DEX file.  The value returned is a magic VM cookie.  On
     * failure, an IOException is thrown.
     */
    private static native long openDexFileNative(String sourceName, String outputName, int flags);
```
这里也就直接调用了native方法进行优化。继续跟进代码在\dalvik\vm\native\dalvik_system_DexFile.cpp文件中的openDexFileNative() 函数，接下重点就在这个函数：
``` cpp
static void Dalvik_dalvik_system_DexFile_openDexFileNative(const u4* args,
    JValue* pResult)
{
//args[0]: sourceName java层传入的
//args[1]: outputName    
    StringObject* sourceNameObj = (StringObject*) args[0];
    StringObject* outputNameObj = (StringObject*) args[1];
    DexOrJar* pDexOrJar = NULL;
    JarFile* pJarFile;
RawDexFile* pRawDexFile;
//DexOrJar*  JarFile*   RawDexFile* 目录
    char* sourceName;
    char* outputName;
    //……
    sourceName = dvmCreateCstrFromString(sourceNameObj);
    if (outputNameObj != NULL)
        outputName = dvmCreateCstrFromString(outputNameObj);
    else
        outputName = NULL;
/*判断要加载的dex是否为系统中的dex文件
* gDvm ？？？
*/
    if (dvmClassPathContains(gDvm.bootClassPath, sourceName)) {
        ALOGW("Refusing to reopen boot DEX '%s'", sourceName);
        dvmThrowIOException(
            "Re-opening BOOTCLASSPATH DEX files is not allowed");
        free(sourceName);
        free(outputName);
        RETURN_VOID();
    }

    /*
     * Try to open it directly as a DEX if the name ends with ".dex".
     * If that fails (or isn't tried in the first place), try it as a
     * Zip with a "classes.dex" inside.
     */
    //判断sourcename扩展名是否是.dex
    if (hasDexExtension(sourceName)
            && dvmRawDexFileOpen(sourceName, outputName, &pRawDexFile, false) == 0) {
        ALOGV("Opening DEX file '%s' (DEX)", sourceName);
        pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
        pDexOrJar->isDex = true;
        pDexOrJar->pRawDexFile = pRawDexFile;
        pDexOrJar->pDexMemory = NULL;
    //.jar文件
    } else if (dvmJarFileOpen(sourceName, outputName, &pJarFile, false) == 0) {
        ALOGV("Opening DEX file '%s' (Jar)", sourceName);
        pDexOrJar = (DexOrJar*) malloc(sizeof(DexOrJar));
        pDexOrJar->isDex = false;
        pDexOrJar->pJarFile = pJarFile;
        pDexOrJar->pDexMemory = NULL;
} else {
//都不满足，抛出异常
        ALOGV("Unable to open DEX file '%s'", sourceName);
        dvmThrowIOException("unable to open DEX file");
    }
if (pDexOrJar != NULL) {
        pDexOrJar->fileName = sourceName;
    //把pDexOr这个结构体中的内容加到gDvm中的userDexFile结构的hash表中，便于Dalvik以后的查找
        addToDexFileTable(pDexOrJar);
    } else {
        free(sourceName);
    }
    free(outputName);
    RETURN_PTR(pDexOrJar);
}
```
再看对.dex文件的处理函数dvmRawDexFileOpen（\dalvik\vm\RawDexFile.cpp）的处理
``` cpp
/* See documentation comment in header. */
int dvmRawDexFileOpen(const char* fileName, const char* odexOutputName,
    RawDexFile** ppRawDexFile, bool isBootstrap)
{
    DvmDex* pDvmDex = NULL;
    char* cachedName = NULL;
    int result = -1;
    int dexFd = -1;
    int optFd = -1;
    u4 modTime = 0;
    u4 adler32 = 0;
    size_t fileSize = 0;
    bool newFile = false;
    bool locked = false;
    dexFd = open(fileName, O_RDONLY);  //打开dex文件
    if (dexFd < 0) goto bail;
    /* If we fork/exec into dexopt, don't let it inherit the open fd. */
dvmSetCloseOnExec(dexFd);//dexfd不继承
//校验dex文件的标志，将第8字节开始的4个字节赋值给adler32。
    if (verifyMagicAndGetAdler32(dexFd, &adler32) < 0) {
        ALOGE("Error with header for %s", fileName);
        goto bail;
    }
    //得到dex文件的大小和修改时间，保存在modTime和filesize中
    if (getModTimeAndSize(dexFd, &modTime, &fileSize) < 0) {
        ALOGE("Error with stat for %s", fileName);
        goto bail;
    }

    //odexOutputName就是odex文件名，如果odexOutputName为空，则自动生成一个。
    if (odexOutputName == NULL) {
        cachedName = dexOptGenerateCacheFileName(fileName, NULL);
        if (cachedName == NULL)
            goto bail;
    } else {
        cachedName = strdup(odexOutputName);   
    }
    //主要是验证缓存文件名的正确性，之后将dexOptHeader结构写入fd中
    optFd = dvmOpenCachedDexFile(fileName, cachedName, modTime,
        adler32, isBootstrap, &newFile, /*createIfMissing=*/true);
    locked = true;
  
    if (newFile) {
        u8 startWhen, copyWhen, endWhen;
        bool result;
        off_t dexOffset;
        dexOffset = lseek(optFd, 0, SEEK_CUR);  //文件指针的位置
        result = (dexOffset > 0);
        if (result) {
            startWhen = dvmGetRelativeTimeUsec();
    //将dex文件中的内容拷贝到当前odex文件，也就是dexOffset开始
            result = copyFileToFile(optFd, dexFd, fileSize) == 0;
            copyWhen = dvmGetRelativeTimeUsec();
        }
        if (result) {
    //优化odex文件
            result = dvmOptimizeDexFile(optFd, dexOffset, fileSize,
                fileName, modTime, adler32, isBootstrap);
        }
    }
    /*
     * Map the cached version.  This immediately rewinds the fd, so it
     * doesn't have to be seeked anywhere in particular.
     */
//将odex文件映射到内存空间(mmap)，并用mprotect将属性置为只读属性，并将映射的dex结构放在pDvmDex数据结构中，具体代码在下面。
    if (dvmDexFileOpenFromFd(optFd, &pDvmDex) != 0) {
        ALOGI("Unable to map cached %s", fileName);
        goto bail;
    }
……
}
```

``` cpp
//Dalvik/vm/RewDexFile.cpp
static int verifyMagicAndGetAdler32(int fd, u4 *adler32)
{
    u1 headerStart[12];
    ssize_t amt = read(fd, headerStart, sizeof(headerStart));
    if (amt < 0) {
        ALOGE("Unable to read header: %s", strerror(errno));
        return -1;
    }
    if (amt != sizeof(headerStart)) {
        ALOGE("Unable to read full header (only got %d bytes)", (int) amt);
        return -1;
    }
    if (!dexHasValidMagic((DexHeader*) (void*) headerStart)) {
        return -1;
    }
    *adler32 = (u4) headerStart[8]
        | (((u4) headerStart[9]) << 8)
        | (((u4) headerStart[10]) << 16)
        | (((u4) headerStart[11]) << 24);

    return 0;
}
```

``` cpp
//dalvik\vm\DvmDex.cpp
/*
 * Given an open optimized DEX file, map it into read-only shared memory and
 * parse the contents.
 *
 * Returns nonzero on error.
 */
int dvmDexFileOpenFromFd(int fd, DvmDex** ppDvmDex)
{
    DvmDex* pDvmDex;
    DexFile* pDexFile;
    MemMapping memMap;
    int parseFlags = kDexParseDefault;
    int result = -1;

    if (gDvm.verifyDexChecksum)
        parseFlags |= kDexParseVerifyChecksum;
    if (lseek(fd, 0, SEEK_SET) < 0) {
        ALOGE("lseek rewind failed");
        goto bail;
    }
    //mmap映射fd文件,就是我们之前的odex文件
   if (sysMapFileInShmemWritableReadOnly(fd, &memMap) != 0) {
        ALOGE("Unable to map file");
        goto bail;
    }
    pDexFile = dexFileParse((u1*)memMap.addr, memMap.length, parseFlags);
    if (pDexFile == NULL) {
        ALOGE("DEX parse failed");
        sysReleaseShmem(&memMap);
        goto bail;
    }
    pDvmDex = allocateAuxStructures(pDexFile);
    if (pDvmDex == NULL) {
        dexFileFree(pDexFile);
        sysReleaseShmem(&memMap);
        goto bail;
    }
/* tuck this into the DexFile so it gets released later */
//将映射odex文件的内存拷贝到DvmDex的结构中
    sysCopyMap(&pDvmDex->memMap, &memMap);
    pDvmDex->isMappedReadOnly = true;
    *ppDvmDex = pDvmDex;
    result = 0;

bail:
    return result;
}
```

``` cpp
/*dalvik\libdex\SysUtil.cpp
*/
int sysMapFileInShmemWritableReadOnly(int fd, MemMapping* pMap)
{
    off_t start;
    size_t length;
    void* memPtr;
assert(pMap != NULL);
//获得文件长度和文件开始地址
    if (getFileStartAndLength(fd, &start, &length) < 0)
        return -1;
//映射文件
    memPtr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_FILE | MAP_PRIVATE,
            fd, start);
    //……
//将保护属性置为只读属性
    if (mprotect(memPtr, length, PROT_READ) < 0) {
      //…….
    }
    pMap->baseAddr = pMap->addr = memPtr;
    pMap->baseLength = pMap->length = length;
return 0;
//……
}
```
这些就是对dex的文件处理，对压缩包zip,jar,apk的有兴趣的可以直接分析下源码。

#### 总结
首先是对文件名的修正，后缀名置为”.dex”作为输出文件，然后生个一个DexPathList对象函数直接返回一个DexPathList对象，

在DexPathList的构造函数中调用makeDexElements()函数，在makeDexElement()函数中调用loadDexFile()开始对.dex或者是.jar .zip .apk文件进行处理，

跟入loadDexFile()函数中，会发现里面做的工作很简单，调用optimizedPathFor()函数对optimizedDiretcory路径进行修正。

之后才真正通过DexFile.loadDex()开始加载文件中的数据，其中的加载也只是返回一个DexFile对象。

在DexFile类的构造函数中，重点便放在了其调用的openDexFile()函数，在openDexFile()中调用了openDexFileNative()真正进入native层，

在openDexFileNative()的真正实现中，对于后缀名为.dex的文件或者其他文件(.jar .apk .zip)分开进行处理：

.dex文件调用dvmRawDexFileOpen()；
其他文件调用dvmJarFileOpen()。

在dvmRawDexFileOpen()函数中，检验dex文件的标志，检验odex文件的缓存名称，之后将dex文件拷贝到odex文件中，并对odex进行优化

调用dvmDexFileOpenFromFd()对优化后的odex文件进行映射，通过mprotect置为"只读"属性并将映射的内存结构保存在DvmDex*结构中。

dvmJarFileOpen()先对文件进行映射，结构保存在ZipArchive中，然后再尝试以文件名作为dex文件名来“打开”文件，
如果失败，则调用dexZipFindEntry在ZipArchive的名称hash表中找名为"class.dex"的文件,然后创建odex文件，下面就和
dvmRawDexFileOpen()一样了，就是对dex文件进行优化和映射。
