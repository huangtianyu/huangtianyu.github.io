---
title: ' Android NDK开发系列教程2：基本方法调用及传参'
date: 2018-02-02 21:56:11
categories: Android
tags: Android NDK开发，JNI开发
---
### 1. 简介
有时候我写了个Java层的方法，希望native层也能够调用（尤其是一个实体类的get，set方法在native一般都会用到）。这在jni开发中也很常见，jni.h中也提供了很多方法。下面利用具体实例进行说明。这里直接使用AS3.0里面的CMake进行编译了，之后会讲解下Android.mk和Application.mk的用法和含义。这里我主要介绍一下几个：
1. java向native传递常用基本数据类型和字符串类型
2. java向native传递数组类型
3. java向native传递自定义java对象
4. java向native传递List对象

#### 1.1 java和jni类型对照表
在我们调用方法时会用到方法的签名，使用类变量时需要用该变量对应的jni类型。下面给出对应的类型对照表。
1. 基本数据类型对照表：
![这里写图片描述](http://img.blog.csdn.net/20180201185144006?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
2. 对象类型对照表：
![这里写图片描述](http://img.blog.csdn.net/20180201185721731?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
3. 简写对应表
![这里写图片描述](http://img.blog.csdn.net/20180202143637829?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 2. 具体例子
#### 2.1 java向native传递常用基本数据类型和字符串类型
强大的AS在你写了java的native方法后，直接快捷键按Alt+Enter后即可生成对应的方法。
java层的方法：
```
package zqc.com.example;
public class NativeTest {
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }
    //定义一个native方法，然后传入基本数据类型和String型
    public native void java2jniMethod1(boolean b, int i, float f, String s);
}
```
生成后的native方法：
```
extern "C"
JNIEXPORT void JNICALL
Java_zqc_com_example_NativeTest_java2jniMethod1(JNIEnv *env, jobject instance, jboolean b, jint i,
                                                jfloat f, jstring s_) {
    //在native层会把string转换成c/c++都特别熟悉的char*，由char*可以转string,wstring等等。
    //在Java层String是对象，这里讲char*指针指向了该对象，在方法结束的时候记得要是否该指针引用
    if (b == JNI_TRUE) {
        LOGE("b is true");
    } else {
        LOGE("b is false");
    }
    float nativi = i + f;
    LOGE("native i: %f", nativi);
    const char *s = env->GetStringUTFChars(s_, 0);
    LOGE("native string: %s", s);
    env->ReleaseStringUTFChars(s_, s);
}
```
在上面可以看到，Java层的基本类型方法都会经过jni进行转换，转换成相应的jni类型。其操作也很方便。Java的String类型需要注意下，一般是将jstring先转换为char*然后对char *进行操作。由于这获取了一个局部引用，一般在调用结束后需要释放该局部引用。

#### 2.2 java向native传递数组类型
```
    //向native传递数组类型
    public native void java2jniMethod2(int[] as, String[] strs);
```
对应的jni方法是：
```
extern "C"
JNIEXPORT void JNICALL
Java_zqc_com_example_NativeTest_java2jniMethod2(JNIEnv *env, jobject instance, jintArray as_,
                                                jobjectArray strs) {
    //获取数组里面内容
    jint *as = env->GetIntArrayElements(as_, NULL);
    int result = 0, len = env->GetArrayLength(as_);
    for (int i = 0; i < len; ++i) {
        result += as[i];
    }
    LOGE("intarray sum is %d", result);
    env->ReleaseIntArrayElements(as_, as, 0);
    //这里可以看出String[]对应的是jobjectArray
    len = env->GetArrayLength(strs);
    for (int i = 0; i < len; ++i) {
        jstring temp = (jstring) env->GetObjectArrayElement(strs, i);
        const char *ctemp = env->GetStringUTFChars(temp, JNI_FALSE);
        LOGE("第%d个：%s", i, ctemp);
    }
}
```
其中函数： jsize GetArrayLength(jxxxarray array);用于获取数组的长度
在Java端调用代码如下：
```
    NativeTest test = new NativeTest();
    int a[] = new int[3];
    for (int i=0;i<a.length;i++) {
        a[i] = i + 10;
    }
    String[] strs = new String[4];
    for (int i=0;i<strs.length;i++) {
        strs[i] = "我的值："+i;
    }
    test.java2jniMethod2(a,strs);
```
##### 2.2.1 处理基本数据类型有以下几个相关函数：
(1) GetXXXArrayElements(Array arr , jboolean* isCopide);
这类函数可以把Java基本类型的数组转换到C/C++中的数组，有两种处理方式，一种JNI\_TRUE是拷贝一份传回本地代码，另一个是JNI\_FALSE把指向Java数组的指针直接传回到本地代码中，处理完本地化的数组后，通过ReleaseXXXArrayElements来释放数组

(2) ReleaseXXXArrayElements(Array arr , * array , jint mode)
用这个函数可以选择将如何处理Java跟C++的数组，是提交，还是撤销等，内存释放还是不释放等mode可以取下面的值:
0 ：对Java的数组进行更新并释放C/C++的数组
JNI_COMMIT ：对Java的数组进行更新但是不释放C/C++的数组
JNI_ABORT：对Java的数组不进行更新,释放C/C++的数组

(3) GetPrimitiveArrayCritical(jarray arr , jboolean* isCopied);
在获得数组上的锁后将返回一个句柄给数组。如果没有建立任何锁，则isCopy被置为JNI\_TRUE，否则置为NULL或JNI\_FALSE：

(4) ReleasePrimitiveArrayCritical(jarray arr , void* array , jint mode);
释放从GetPrimitiveArrayCritical调用中返回的数组。也是JDK1.2出来的，为了增加直接传回指向Java数组的指针而加入的函数，同样的也会有同GetStringCritical的死锁的问题。mode取值如下：
0：从carray中复制值到数组中，并释放分配给carray的存储器
JNI_COMMIT：从carray中复制值到数组中，但是不释放分配给carray的存储器
JNI_ABORT：不从carray中复制值到数组中

(5) GetXXXArrayRegion(Array arr , jsize start , jsize len , * buffer);
在C/C++预先开辟一段内存，然后把Java基本类型的数组拷贝到这段内存中。用于一个数组子集的复制操作。参数start指定了从何处复制的起始索引，参数len则指定了从数组中复制到本机数组的多个位置数量。

(6) SetXXXArrayRegion(Array arr , jsize start , jsize len , const * buffer);
用来复制本机数组的一段内容回Java数组中。元素一般从本机数组起始处（索引为0）开始复制，但是只是从位置start开始将len个元素复制到Java数组中。

(7) Array NewXXXArray(jsize sz)
创建一个包含length个元素的Java数组。

##### 2.2.2 处理对象数组类型有以下几个相关函数：
(1)  jobjectArray NewObjectArray(jsize length, jclass elementClass, jobject initialElement );
创建对象数组，创建一个长度为length，并且持有类型为elementClass的对象的对象数组，数组中的所有元素都被置为initialElement

(2)  jobject GetObjectArrayElement(jobjectArray array, jsize Index);
获取数组元素，通过Index指定的索引在array中获取一个对象，如果索引超出边界，会抛出一个IndexOutOfBoundsException

(3) void SetObjectArrayElement(jobjectArray array, jsize index,jobject value);
设置元素值。在array中通过index指定的索引处设置元素值为value，如果index超出边界，会抛出一个IndexOutOfBoundException。

#### 2.3  java向native传递自定义java对象
定义一个Java层方法：
```java
package zqc.com.example;
/**
 * Created by zhangqianchu on 2018/2/1.
 */
class Person {
    long id;
    String name;
    int age;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
定义一个Java的native方法：
```
    //Java向native传自定义类对象
    public native void java2jniMethod3(Person person);
```
在native层实现
```
extern "C"
JNIEXPORT void JNICALL
Java_zqc_com_example_NativeTest_java2jniMethod3(JNIEnv *env, jobject instance, jobject person) {
    jclass cls = env->FindClass("zqc/com/example/Person");
    if (cls == 0) {
        LOGE("find class fail");
        return;
    }
    jmethodID mid_ID = env->GetMethodID(cls, "setId", "(J)V");
    jmethodID mid_Name = env->GetMethodID(cls, "setName", "(Ljava/lang/String;)V");
    jmethodID mid_Age = env->GetMethodID(cls, "setAge", "(I)V");
    if (mid_ID && mid_Name && mid_Age) {
        env->CallVoidMethod(person, mid_ID, 100L);
        jstring name = env->NewStringUTF("Tianyu");
        env->CallVoidMethod(person, mid_Name, name);
        env->CallVoidMethod(person, mid_Age, 18);
        return;
    }
}
```
在Java端调用
```
Person person = new Person();
test.java2jniMethod3(person);
Toast.makeText(this, person.toString(),Toast.LENGTH_SHORT).show();
```
从Java端传对象实例给native时，到native端任何对象都变为jobject类型，如果要做对该对象实例的任何操作需先获取该对象的jfieldID ,jmethodID,然后通过env->CallXXXMethod来操作该对象的方法其中第一个参数是该对象的具体实例，其中env->CallStaticXXXMethod方法用来调用该类的静态方法，调用静态方法的时候就不用传具体的对象过去了。

#### 2.4 java向native传递List对象
定义Java的native方法
```
//Java向native传List对象
public native void java2jniMethod4(List<Person>people);
```
在native中实现具体方法：
```
extern "C"
JNIEXPORT void JNICALL
Java_zqc_com_example_NativeTest_java2jniMethod4(JNIEnv *env, jobject instance, jobject people) {
    //下面所有操作都得先判断是否为空。。。
    jclass cls = env->GetObjectClass(people);
    jclass pcls = env->FindClass("zqc/com/example/Person");
    //jmethodID getNameMid = env->GetMethodID(pcls, "getName", "()Ljava/lang/String;");
    jmethodID  setNameMid = env->GetMethodID(pcls, "setName", "(Ljava/lang/String;)V");
    //获取List的get方法id
    jmethodID getMid = env->GetMethodID(cls, "get", "(I)Ljava/lang/Object;");
    //获取List的长度
    jmethodID sizeMid = env->GetMethodID(cls, "size", "()I");
    int len = env->CallIntMethod(people, sizeMid);
    for (int i = 0; i < len; ++i) {
        //获取第i个元素
        jobject  data = env->CallObjectMethod(people, getMid, i);
        env->CallVoidMethod(data, setNameMid, env->NewStringUTF("全部随我native"));
    }
}
```
<code> jclass cls = env->GetObjectClass(people);</code>这是获取一个对象实例相应的类的最好的办法。从上面可以看出List在传到native时也是变成了jobject，然后具体操作都得通过env->GetObjectClass先获取到该类，然后获取到该类的具体jmethodID，jfieldID来完成相应的操作。调用的方法也是env->CallXXXMethod()。
然后在Java端调用该native方法：
```
    NativeTest test = new NativeTest();
    List<Person> people = new ArrayList<>();
    for (int i = 0; i < 3; i++) {
        Person person1 = new Person();
        person1.setName("我是Java层");
        people.add(person1);
    }
    test.java2jniMethod4(people);
    Iterator<Person> iterator = people.iterator();
    while (iterator.hasNext()) {
        Person person1 = iterator.next();
        Log.e("myndk", person1.getName() + "\n");
    }
```
在上面即可通过native将Person的name全部进行了更改。

<B>上面</B>都是Java向native传参，基本用法都类似。基本数据类型有相应的对照表，对象类型的都转为jobject，对对象的操作都是先获取该对象jclass,jmethodID,jfielID后再对对象实例进行操作。