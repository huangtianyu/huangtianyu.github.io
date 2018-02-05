---
title: Android NDK开发系列教程3：基本方法调用及传参（续）
date: 2018-02-02 22:25:43
categories: Android
tags: Android NDK开发，JNI开发
---
上一节主要讲解Java向native传参，下面主要讲解从*native传相应的数据到java层*。
接着上一节，下面主要讲解内容如下：
1. native向java返回字符串类型
2. native向java返回java对象
3. native向java返回数组类型
4. native向Java返回List对象
对于上面的每个都给出对应的例子。
本节所有案例代码均已放到GitHub上，欢迎下载：
https://github.com/huangtianyu/JNILearnCourse

#### 1. native向java返回字符串类型
传基本数据类型很简单，是什么就传什么就行。传字符串类型也很简单，具体jni代码如下：
```
extern "C"
JNIEXPORT jstring JNICALL
Java_zqc_com_example_NativeTest_jni2javaMethod1(JNIEnv *env, jobject instance) {
    //jstring NewStringUTF(const char* bytes),jstring NewString(const jchar* unicodeChars, jsize len)
    char *returnValue = "你在native做你的操作后，生成char*后，通过env->NewStringUTF即可返回Java的String类型";
    return env->NewStringUTF(returnValue);
}
```
其中最主要用的是以下几个方法：
```
	//创建Unicode格式的jstring串
    jstring NewString(const jchar* unicodeChars, jsize len)
    { return functions->NewString(this, unicodeChars, len); }
	//获取jstring长度
    jsize GetStringLength(jstring string)
    { return functions->GetStringLength(this, string); }
	//获取jstring对应的字符串，isCopy表示是否拷贝生成副本。
	//这个函数返回一个指向特定jstring中字符顺序的指针，该指针保持有效直到ReleaseStringChars函数被调用：
    const jchar* GetStringChars(jstring string, jboolean* isCopy)
    { return functions->GetStringChars(this, string, isCopy); }
	//释放指针
    void ReleaseStringChars(jstring string, const jchar* chars)
    { functions->ReleaseStringChars(this, string, chars); }
	////创建UTF-8格式的jstring串
    jstring NewStringUTF(const char* bytes)
    { return functions->NewStringUTF(this, bytes); }
	//获取utf字符串的长度
    jsize GetStringUTFLength(jstring string)
    { return functions->GetStringUTFLength(this, string); }
	//同GetStringChars
    const char* GetStringUTFChars(jstring string, jboolean* isCopy)
    { return functions->GetStringUTFChars(this, string, isCopy); }
	//同ReleaseStringChars
    void ReleaseStringUTFChars(jstring string, const char* utf)
    { functions->ReleaseStringUTFChars(this, string, utf); }
```
以上是处理字符串常用的一些方法。
#### 2 native向java返回java对象
具体看native的代码如下：
```
extern "C"
JNIEXPORT jobject JNICALL
Java_zqc_com_example_NativeTest_jni2javaMethod2(JNIEnv *env, jobject instance) {
    jclass pcls = env->FindClass("zqc/com/example/Person");
    jmethodID constructor = env->GetMethodID(pcls, "<init>", "()V");
    jmethodID setIdMid = env->GetMethodID(pcls, "setId", "(J)V");
    jmethodID setNameMid = env->GetMethodID(pcls, "setName", "(Ljava/lang/String;)V");
    jmethodID setAgeMid = env->GetMethodID(pcls, "setName", "(I)V");
    jobject person = env->NewObject(pcls, constructor);
    env->CallVoidMethod(person, setIdMid, 100L);
    env->CallVoidMethod(person, setNameMid, env->NewStringUTF("天宇"));
    env->CallVoidMethod(person, setAgeMid, 18);
    return person;
}
```
常用新建Object的方法由以下几个：
```
	//将传递给构造函数的所有参数紧跟着放在 methodID 参数的后面。NewObject() 收到这些参数后，将把它们传给所要调用的Java 方法。
    jobject NewObject(jclass clazz, jmethodID methodID, ...)
    {
        va_list args;
        va_start(args, methodID);
        jobject result = functions->NewObjectV(this, clazz, methodID, args);
        va_end(args);
        return result;
    }
	//将传递给构造函数的所有参数放在 va_list 类型的参数 args 中，该参数紧跟着放在 methodID 参数的后面。NewObject() 收到这些参数后，将把它们传给所要调用的 Java 方法。
    jobject NewObjectV(jclass clazz, jmethodID methodID, va_list args)
    { return functions->NewObjectV(this, clazz, methodID, args); }
	//将传递给构造函数的所有参数放在jvalues类型的数组args中，该数组紧跟着放在methodID参数的后面。NewObject() 收到数组中的这些参数后，将把它们传给所要调用的 Java 方法。
    jobject NewObjectA(jclass clazz, jmethodID methodID, jvalue* args)
    { return functions->NewObjectA(this, clazz, methodID, args); }
    //测试对象是否为某个类的实例。
    jboolean IsInstanceOf(jobject obj, jclass clazz)
    { return functions->IsInstanceOf(this, obj, clazz); }
    //测试两个引用是否引用同一 Java 对象。
    jboolean IsSameObject(jobject ref1, jobject ref2)
    { return functions->IsSameObject(this, ref1, ref2); }
    //分配新 Java 对象而不调用该对象的任何构造函数。返回该对象的引用。
    //该方法会抛出：InstantiationException：如果该类为一个接口或抽象类。OutOfMemoryError：如果系统内存不足。
    jobject AllocObject(jclass clazz)
    { return functions->AllocObject(this, clazz); }
```

#### 3 native向java返回数组类型
##### 3.1 基本类型数组
这里直接看native层代码如下：
```
extern "C"
JNIEXPORT jintArray JNICALL
Java_zqc_com_example_NativeTest_jni2javaMethod3(JNIEnv *env, jobject instance) {
    int nat[] = {2, 1, 4, 3, 5};
    jintArray jnat = env->NewIntArray(5);
    env->SetIntArrayRegion(jnat, 0, 5, nat);
    return jnat;
}
```
基本数据类型数组都有相应的env->NewXXXArray(jsize length);通过该方法可以生成对应的数组。
```
jbooleanArray NewBooleanArray(jsize length)
    { return functions->NewBooleanArray(this, length); }
    jbyteArray NewByteArray(jsize length)
    { return functions->NewByteArray(this, length); }
    jcharArray NewCharArray(jsize length)
    { return functions->NewCharArray(this, length); }
    jshortArray NewShortArray(jsize length)
    { return functions->NewShortArray(this, length); }
    jintArray NewIntArray(jsize length)
    { return functions->NewIntArray(this, length); }
    jlongArray NewLongArray(jsize length)
    { return functions->NewLongArray(this, length); }
    jfloatArray NewFloatArray(jsize length)
    { return functions->NewFloatArray(this, length); }
    jdoubleArray NewDoubleArray(jsize length)
    { return functions->NewDoubleArray(this, length); }
```
在生成了对应的数组后，可以通过setXXXArrayRegion(jxxxArray array, jsize start, jsize len,  const jchar* buf)来填充数组
```
    void SetBooleanArrayRegion(jbooleanArray array, jsize start, jsize len,
        const jboolean* buf)
    { functions->SetBooleanArrayRegion(this, array, start, len, buf); }
    void SetByteArrayRegion(jbyteArray array, jsize start, jsize len,
        const jbyte* buf)
    { functions->SetByteArrayRegion(this, array, start, len, buf); }
    void SetCharArrayRegion(jcharArray array, jsize start, jsize len,
        const jchar* buf)
    { functions->SetCharArrayRegion(this, array, start, len, buf); }
    void SetShortArrayRegion(jshortArray array, jsize start, jsize len,
        const jshort* buf)
    { functions->SetShortArrayRegion(this, array, start, len, buf); }
    void SetIntArrayRegion(jintArray array, jsize start, jsize len,
        const jint* buf)
    { functions->SetIntArrayRegion(this, array, start, len, buf); }
    void SetLongArrayRegion(jlongArray array, jsize start, jsize len,
        const jlong* buf)
    { functions->SetLongArrayRegion(this, array, start, len, buf); }
    void SetFloatArrayRegion(jfloatArray array, jsize start, jsize len,
        const jfloat* buf)
    { functions->SetFloatArrayRegion(this, array, start, len, buf); }
    void SetDoubleArrayRegion(jdoubleArray array, jsize start, jsize len,
        const jdouble* buf)
    { functions->SetDoubleArrayRegion(this, array, start, len, buf); }
```
##### 3.2 对象类型数组
直接看native代码：
```
extern "C"
JNIEXPORT jobjectArray JNICALL
Java_zqc_com_example_NativeTest_jni2javaMethod4(JNIEnv *env, jobject instance) {
    jclass cls = env->FindClass("zqc/com/example/Person");
    jmethodID cMid = env->GetMethodID(cls, "<init>", "()V");
    jclass pcls = env->FindClass("zqc/com/example/Person");
    jmethodID cmid = env->GetMethodID(pcls, "<init>", "()V");
    jmethodID setNameMid = env->GetMethodID(pcls, "setName", "(Ljava/lang/String;)V");
    jmethodID setAgeMid = env->GetMethodID(pcls, "setAge", "(I)V");
    jobject obj = env->NewObject(pcls, cmid);
    env->CallVoidMethod(obj, setNameMid, env->NewStringUTF("天宇"));

    int len = 3;
    jobjectArray joa = env->NewObjectArray(len, cls, obj);
    for (int i = 0; i < len; ++i) {
        jobject tmp = env->GetObjectArrayElement(joa,i);
        env->CallVoidMethod(tmp, setAgeMid, i + 10);
    }
    return joa;
}
```
其在native生成的方法是<code>   jobjectArray joa = env->NewObjectArray(len, cls, obj);</code>
```
//第一个参数表示生成的长度，第二参数表示里面元素的对象类，第三个表示原始初始化时的值。在生成后每个元素都是该值。
jobjectArray NewObjectArray(jsize length, jclass elementClass,
        jobject initialElement)
    { return functions->NewObjectArray(this, length, elementClass,
        initialElement); }
```

#### 4 native向Java返回List对象
直接看native代码如下：
```
extern "C"
JNIEXPORT jobject JNICALL
Java_zqc_com_example_NativeTest_jni2javaMethod5(JNIEnv *env, jobject instance) {
    jclass listCls = env->FindClass("java/util/ArrayList");//获得ArrayList类引用
    jmethodID  listCon = env->GetMethodID(listCls, "<init>", "()V");//获取构造函数的methodID
    jmethodID addMid = env->GetMethodID(listCls,"add","(Ljava/lang/Object;)Z");//获取add函数的methodID

    jobject listObj = env->NewObject(listCls, listCon);//利用NewObject创建一个ArrayList对象
    jobject jperon = Java_zqc_com_example_NativeTest_jni2javaMethod2(env, instance);//利用上面方法新建一个Person对象
    env->CallBooleanMethod(listObj, addMid, jperon);//在listObj中add一个Person对象
    //返回ArrayList的对象
    return listObj;
}
```
对应jni而言，List，ArrayList以及Map，HashMap，Set，HashSet都只是一个Object，对应于jni而言也就都是jobject，操作jobject都可以用最开始介绍的方法。

### 总结
jni里面的方法很多，多用用就熟悉了。常用的上面都有，自己之前还总结了很多常用的类型转换函数。以后有时间再写篇博客分享下。