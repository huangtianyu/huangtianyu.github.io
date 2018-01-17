---
title: adb常用命令
date: 2018-01-17 10:15:31
categories: Android
tags: adb命令
---
### adb简介
adb全称Android Debug Bridge，意为安卓调试桥接，即常用于Android手机调试的工具。adb提供了一系列命令可以操作手机，比如安装卸载软件，运行shell命令等等。adb工作方式是采用监听Socket TCP5554等端口的方式让IDE和Qemu通讯。

### 常用的adb命令
1. 取得当前连接电脑的设备的状态
```
adb devices
```
2. 安装卸载软件
```
adb install  <-r> 文件.apk  //-r表示替换掉原来的apk
adb uninstall <-k> {package} //packageName表示应用包名 -k表示保留配置和缓存
```
3. 获取设备的序列号
```
adb get-serialno //获取设备序列号
```
4. 访问手机SQLit数据库（手机内部需有sqilte3命令支持）
```
adb shell
sqlite3 {数据库文件名}
```
5. 从电脑上发送文件到手机端
```
adb push {文件路径} {手机端路径}  //文件表示要发送文件的全路径（当前路径下只需写上文件名 {手机端路径}表示拷贝到手机上的路径，/sdcard/表示sd卡根路径
```
6. 从手机端拉取文件到电脑端
```
adb pull {手机端路径} {保存到电脑端路径}
```
7. 查看bug报告
```
adb bugreport  //Android7.0及以后支持该功能
```
8. 获取手机中的第三方包名
```
adb shell pm list packages -3
```
9. 查看当前正在交互的程序
```
adb shell dumpsys activity | grep "Running activities" -A 7
```
10. 获取包名所在路径
```
adb shell pm path {package}
```
11. Monkey命令
全模块：
```
adb shell "monkey --ignore-crashes --ignore-timeouts --throttle 500 --ignore-security-exceptions --monitor-native-crashes --ignore-native-crashes 10000 > sdcard/monkey.txt"
```
单模块
```
adb shell monkey -p com.android.contacts --ignore-crashes --ignore-timeouts --ignore-security-exceptions --monitor-native-crashes --ignore-native-crashes --throttle 500 -v 10000 > sdcard/monkey.txt"
```
12. 强行停止monkey
```
adb shell
ps | grep monkey
kill monkey对应的pid
```
13. 列出所有可以dump的选项
```
adb shell dumpsys -l
```
14. 查看内存信息
```
adb shell dumpsys meminfo
```
15. 列出dumpsys能提供的所有服务列表
```
adb shell dumpsys service list //得到该列表后，就可以在dumpsys后面加上service的名称来查看指定service的信息了
```
16. 查看ActvityManagerService 所有信息
```
adb shell dumpsys activity
```
17. 查看Activity组件信息
```
adb shell dumpsys activity activities
```
18. 查看Service组件信息
```
adb shell dumpsys activity services
```
19. 产看ContentProvider组件信息
```
adb shell dumpsys activity providers
```
20. 查看BraodcastReceiver信息
```
adb shell dumpsys activity broadcasts
```
21. 查看Intent信息
```
adb shell dumpsys activity intents
```
22. 查看进程信息
```
adb shell dumpsys activity processes
```
23. 关闭或开启adb服务
```
adb kill-server
adb start-server
```
24. 显示或导出log信息
```
adb logcat <-s> //在命令行显示log信息 -s表示指定标签tag
adb logcat > log.txt //将log信息保存到当前目录的log.txt文件中
```
25. 启动Activities
```
adb shell am start -n {包名}/{包名＋类名}（-n 类名,-a action,-d date,-m MIME-TYPE,-c category,-e 扩展数据,等）
```
26. 设置系统属性信息
```
adb shell setprop {key} {value}
```
27. 常用的adb shell命令
```
ps //列出所有进程
ls //列出当前目录下文件
df //检查文件系统的磁盘空间占用情况
cat //查看某个文件
kill {pid}//杀死某个进程
cd //进入其他目录
rm //删除
rmdir //删除非空文件夹（有文件的文件夹可能不成功）
```
28. 显示WiFi信息
```
adb shell dumpsys wifi
```
29. 模拟用户点击
```
adb shell input tap {x} {x} //例子：adb shell input tap 50 250 
```
30. 模拟用户滑动
```
adb shell input swipe {start.x} {start.y} {end.x} {end.y} {time} //例子：adb shell input swipe 50 250 250 250 500 //划屏操作，前四个数为坐标点，后面是滑动的时间（单位毫秒）
```
31. 模拟输入字符串
```
adb shell input text 'abc'
```
32. 模拟点击手机自带的功能键：Home，Menu，Back
```
adb shell input keyevent keyCode
keyCode对应表：
KEYCODE_UNKNOWN=0;
KEYCODE_SOFT_LEFT=1;   
KEYCODE_SOFT_RIGHT=2;   
KEYCODE_HOME=3;   
KEYCODE_BACK=4;   
KEYCODE_CALL=5;   
KEYCODE_ENDCALL=6;   
KEYCODE_0=7;   
KEYCODE_1=8;   
KEYCODE_2=9;   
KEYCODE_4=11;   
KEYCODE_5=12;   
KEYCODE_6=13;   
KEYCODE_7=14;   
KEYCODE_8=15;   
KEYCODE_9=16;   
KEYCODE_STAR=17;   
KEYCODE_POUND=18;   
KEYCODE_DPAD_UP=19;   
KEYCODE_DPAD_DOWN=20;   
KEYCODE_DPAD_LEFT=21;   
KEYCODE_DPAD_RIGHT=22;   
KEYCODE_DPAD_CENTER=23;   
KEYCODE_VOLUME_UP=24;   
KEYCODE_VOLUME_DOWN=25;   
KEYCODE_POWER=26;   
KEYCODE_CAMERA=27;   
KEYCODE_CLEAR=28;   
KEYCODE_A=29;   
KEYCODE_B=30;   
KEYCODE_C=31;   
KEYCODE_D=32;   
KEYCODE_E=33;   
KEYCODE_F=34;   
KEYCODE_G=35;   
KEYCODE_H=36;   
KEYCODE_I=37;   
KEYCODE_J=38;   
KEYCODE_K=39;   
KEYCODE_L=40;   
KEYCODE_M=41;   
KEYCODE_N=42;   
KEYCODE_O=43;   
KEYCODE_P=44;   
KEYCODE_Q=45;   
KEYCODE_R=46;   
KEYCODE_S=47;   
KEYCODE_T=48;   
KEYCODE_U=49;   
KEYCODE_V=50;   
KEYCODE_W=51;   
KEYCODE_X=52;   
KEYCODE_Y=53;   
KEYCODE_Z=54;   
KEYCODE_COMMA=55;   
KEYCODE_PERIOD=56;   
KEYCODE_ALT_LEFT=57;   
KEYCODE_ALT_RIGHT=58;   
KEYCODE_SHIFT_LEFT=59;   
KEYCODE_SHIFT_RIGHT=60;   
KEYCODE_TAB=61;   
KEYCODE_SPACE=62;   
KEYCODE_SYM=63;   
KEYCODE_EXPLORER=64;   
KEYCODE_ENVELOPE=65;   
KEYCODE_ENTER=66;   
KEYCODE_DEL=67;   
KEYCODE_GRAVE=68;   
KEYCODE_MINUS=69;   
KEYCODE_EQUALS=70;   
KEYCODE_LEFT_BRACKET=71;   
KEYCODE_RIGHT_BRACKET=72;   
KEYCODE_BACKSLASH=73;   
KEYCODE_SEMICOLON=74;   
KEYCODE_APOSTROPHE=75;   
KEYCODE_SLASH=76;   
KEYCODE_AT=77;   
KEYCODE_NUM=78;   
KEYCODE_HEADSETHOOK=79;   
KEYCODE_FOCUS=80;//*Camera*focus   
KEYCODE_PLUS=81;   
KEYCODE_MENU=82;   
KEYCODE_NOTIFICATION=83;   
KEYCODE_SEARCH=84;   
KEYCODE_MEDIA_PLAY_PAUSE=85;   
KEYCODE_MEDIA_STOP=86;   
KEYCODE_MEDIA_NEXT=87;   
KEYCODE_MEDIA_PREVIOUS=88;   
KEYCODE_MEDIA_REWIND=89;   
KEYCODE_MEDIA_FAST_FORWARD=90;   
KEYCODE_MUTE=91; 
```