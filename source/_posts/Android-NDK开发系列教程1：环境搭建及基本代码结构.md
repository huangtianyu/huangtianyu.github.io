---
title: Android NDK开发系列教程1：环境搭建及基本代码结构
date: 2018-02-01 21:50:54
categories: Android
tags: Android NDK开发，JNI开发
---
### 1. Eclipse NDK开发环境搭建
在开发NDK之前，Java的SDK，Android的NDK，以及Eclipse的ADT工具都需要大家先安装好，在SDK早期版本中没有ndk相关文件，当最近的AndroidSDK中包含了ndk相关文件，所以下载NDK工具的麻烦事这里就没有了。唯一要注意的是需要配置下NDK的环境变量。这样可以方便进行编译。AndroidSDK主要文件夹参考如下：
![AndroidSDK目录结构](http://img.blog.csdn.net/20180201161034954?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这里讲Eclipse的配置就将下如何添加External Tool来快速生成.h文件以及快速进行ndk_build编译。
#### 1.1 配置快速生成.h头文件的命令
1. 点击Eclipse上面的图标，打开External Tool Configurations。
![这里写图片描述](http://img.blog.csdn.net/20180201162000196?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
2. 然后打开如下界面，在如下界面中双击Program，在底下会生成一个New_configuration。
![这里写图片描述](http://img.blog.csdn.net/20180201162930719?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
然后按照图片上面的格式填写相应的参数。
Location填写javah.exe的位置：C:\Program Files\Java\jdk1.8.0_91\bin\javah.exe
Working Directory填写当前的工作目录：${workspace_loc:/MyTest/src}
Arguments填写相应的参数：
```
-classpath ${workspace_loc:/MyTest/src/bin/classes} -d ${workspace_loc:/MyTest/jni} -jni com.scu.MyNDK
```
之后在External Tool的地方就会生成一个JavaH的命令工具，点击即可生成对应的.h头文件了。这里要注意的是生成都文件前要先编译出.class文件。
其实这个和用javah.exe命令是一样的，具体命令如下：
![这里写图片描述](http://img.blog.csdn.net/20180201163500478?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 2. Android Studio的配置
AS太强大了，所有你想要的只需要简单的添加一个依赖，AS就会自动帮你下载，完全不用你去下载。最新的Android Studio在新建工程的时候，选中Include C++ Support后，即可进行NDK开发，这里注意下在AS中的编译换成了CMake工具，这个工具配置上稍微和Android.mk有些许不同。其配置文件在新建工程的CMakeLists.txt里面配置。

在build.gradle里面也会自动配置cmake工具，配置如下：
``` xml
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
```
### 3. 基本代码结构
利用AS创建工程后，工程会自动生成如下代码：
``` java
package zqc.com.example;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

    }
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
}
```
其中在static静态代码块中会加载动态链接库。在一个方法前加上native关键字即表明该方法是一个jni方法，因而只有声明，没有实现，其具体实现在c/c++代码中。
找到cpp文件，打开后内容如下：
``` cpp
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jstring

JNICALL
Java_zqc_com_example_MainActivity_stringFromJNI(
        JNIEnv *env, jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```
其中extern "C"表示在编译的时候导出为c语言的格式，JNIEXPORT表示该函数是可以导出的，可以由外部方法进行调用，这和dll类似，jstring表示返回值，JNICALL关键字表示这是一个jni方法，Java_zqc_com_example_MainActivity_stringFromJNI其中Java是固定格式，zqc_com_example_MainActivity是全类名，stringFromJNI是具体的方法名，具体参数：JNIEnv *env为env指针，调用jni的很多方法都需要该指针，jobject /* this */这个表示当前类的this指针，这里因为没用到就没有命名。
在以往开发中可能是把.h和.cpp分开了，这个是AS自动生成的，这里并没有单独生成.h文件。c/c++开发也有自己的结构，这里除了需要对外暴露接口的需要按照上面格式编写外，其他的都可以用古老的c/c++进行编写并遵循古老的结构。你可以先定义.h文件，然后在.cpp里面具体实现。
点击Build->Make Project（快捷键Ctrl+F9）即可生成动态链接库文件.so，其路径在：
![这里写图片描述](http://img.blog.csdn.net/20180201172808764?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
AS自动生成了Debug版和Release版，并且在各个版本中又生成了不同平台的.so文件。只能说这个AS太牛叉了~
之后运行工程，安装到手机上时就把对应的so也拷贝到了手机中了。

### 总结
目前应该是绝大多数人都采用AndroidStudio进行开发，谷歌官方已经不再对Eclipse的ADT进行维护了。而AS是绝对强大的工具，当你选择Include C++ Support的时候，AS会将NDK开发的一切都下载下来。所以如果采用AS开发，那么你学NDK开发的话，只需要把Android开发需要安装的JDK，SDK，AS等工具安装好后，即可进行开发。这个系列教程我也采用AS进行讲解。