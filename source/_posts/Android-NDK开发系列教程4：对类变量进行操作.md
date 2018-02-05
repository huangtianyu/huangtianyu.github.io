---
title: Android NDK开发系列教程4：对类变量进行操作
date: 2018-02-04 10:54:57
categories: Android
tags: Android JNI开发
---
通常我们也可以直接利用jni来访问和处理类的变量，不一定非要通过Java方法来操作Java类变量。对类变量操作时，类的静态变量和类的实例变量的操作稍微有些不同，下面进行讲解。
### 对类的静态变量进行操作
类的静态变量属于类，是所有该类实例共享的。操作该变量时，不需要指定具体的实例是哪个。
```
jclass clazz;  
jfieldID fid;  
jint num;  

//1.获取类的Class引用  
clazz = env->FindClass("zqc/com/example/Person");  
if (clazz == NULL) {    // 错误处理  
return;  
}  

//2.获取类静态变量num的属性ID  
fid = env->GetStaticFieldID( clazz, "num", "I");  
if (fid == NULL) {  
return;  
}  

// 3.获取静态变量num的值  
num = env->GetStaticIntField(clazz,fid);  
printf("In C--->ClassField.num = %d\n", num);  

// 4.修改静态变量num的值  
env->SetStaticIntField(clazz, fid, 80);  
```
主要步骤就是代码里面注释的。

### 对类的实例变量进行操作
代码如下：
```
jclass clazz;  
jfieldID fid;  
jstring j_str;  
jstring j_newStr;  
const char *c_str = NULL;  

// 1.获取类的Class引用,obj是该类的某个实例jobject obj;
clazz = env->GetObjectClass(obj);  
if (clazz == NULL) {  
return;  
}  

// 2. 获取类实例变量str的属性ID  
fid = env->GetFieldID(clazz,"str", "Ljava/lang/String;");  
if (clazz == NULL) {  
return;  
}  

// 3. 获取实例变量str的值  
j_str = (jstring)env->GetObjectField(obj,fid);  

// 4. 将unicode编码的java字符串转换成C风格字符串  
c_str = env->GetStringUTFChars(j_str,NULL);  
if (c_str == NULL) {  
return;  
}  
printf("In C--->ClassField.str = %s\n", c_str);  
env->ReleaseStringUTFChars(j_str, c_str);  

// 5. 修改实例变量str的值  
j_newStr = env->NewStringUTF("This is C String");  
if (j_newStr == NULL) {  
return;  
}  

env->SetObjectField(obj, fid, j_newStr);  

// 6.删除局部引用  
env->DeleteLocalRef(clazz);  
env->DeleteLocalRef(j_str);  
env->DeleteLocalRef(j_newStr);  
```
JNI开发也有JNI开发的套路，按照上面套路来，即可修改类的实例变量。操作过程也很好理解，我们在native操作的时候都需要借助JNI提供的函数获取相应的引用。利用引用去进行操作。由于JNI函数是直接操作JVM中的数据结构，所以即使是private的变量，我们也可以进行修改。

### 总结
1. 由于JNI函数是直接操作JVM中的数据结构，不受Java访问修饰符的限制。即，在本地代码中可以调用JNI函数可以访问Java对象中的非public属性和方法
2. 访问和修改静态变量操作步聚：
	1. 调用FindClass函数获取类的Class引用

	2. 调用GetStaticFieldID函数获取Class引用中某个静态变量ID

	3. 调用GetStaticXXXField函数获取静态变量的值，需要传入变量所属Class的引用和变量ID

	4. 调用SetStaticXXXField函数设置静态变量的值，需要传入变量所属Class的引用、变量ID和变量的值

3. 访问和修改实例变量操作步聚：

	1. 调用GetObjectClass函数获取实例对象的Class引用

	2. 调用GetFieldID函数获取Class引用中某个实例变量的ID

	3. 调用GetXXXField函数获取变量的值，需要传入实例变量所属对象和变量ID

	4. 调用SetXXXField函数修改变量的值，需要传入实例变量所属对象、变量ID和变量的值


