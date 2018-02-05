---
title: AppLinks使用详解
date: 2018-01-18 16:58:44
categories: Android
tags: AppLinks, DeepLinks
---
### 1. 简介
官方介绍[Android App Links](https://developer.android.com/training/app-links/verify-site-associations.html#the-difference)内容是：
```
Android App Links are a special type of deep link that allow your website URLs to immediately open the
 corresponding content in your Android app (without requiring the user to select the app).

To add Android App Links to your app, define intent filters that open your app content using HTTP URLs (as 
described in [Create Deep Links to App Content]), and verify that you own both your app and the website 
URLs (as described in this guide). If the system successfully verifies that you own the URLs, the system 
automatically routes those URL intents to your app.
```
意思就是AppLinks是一个特殊的DeepLink，它可以让你的应用和你的网站URL进行绑定，这样当你在点击你网站链接的时候（非浏览器中）就能调起你的App，而不是出现选择界面，使用方法如下
[Create Deep Links to App Content](https://developer.android.com/training/app-links/deep-linking.html)
这种绑定不是在点击的时候才核对链接，下面会介绍在什么情况下核对这种绑定的。
### 2. 与DeepLink的区别
官方是这样介绍DeepLink的。
```
A [deep link] is an intent filter that allows users to directly enter a specific activity in your Android app. 
Clicking one of these links might open a disambiguation dialog, which allows the user to select one of 
multiple apps (including yours) that can hande the given URL. For example, figure 1 shows the 
disambiguation dialog after the user clicks a map link, asking whether to open the link in Maps or Chrome.
```
[Deeplink](https://developer.android.com/training/app-links/deep-linking.html)是一个intent过滤器，他可以使用户直接进入某个Activity页面。但是有个不好的是当匹配到多个intent时就会弹一个让用户选择的框。官方给了下面一张图，而AppLinks就不会有这个弹框：
<img src="https://upload-images.jianshu.io/upload_images/7170430-1a711d54789310a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/200" width = "200" height = "400" alt="The disambiguation dialog" align="center" />
具体区别官方也列了以下：

| Item      |    Deep links | App links |
|:-------- |:--------:|:--------|
| Intent URL  | scheme	http, https, or a custom scheme |  Requires http or https   |
| Intent action  |  Any action |  Requires android.intent.action.VIEW  |
| Intent category  |    Any category | Requires android.intent.category.BROWSABLE and android.intent.category.DEFAULT  |
| Link verification |  None	 | Requires a Digital Asset Links file served on you website with HTTPS  |
| User experience | May show a disambiguation dialog for the user to select which app to open the link  | No dialog; your app opens to handle your website links  |
| Compatibility | 	All Android versions  | Android 6.0 and higher  |

### 3.使用步骤
官方给的步骤如下[Handling Android App Links](https://developer.android.com/training/app-links/index.html)
#### 3.1 在manifest中开启autoVerify
```
<activity ...>

    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" android:host="www.example.com" />
        <data android:scheme="https" />
    </intent-filter>

</activity>
```
这里注意下，开启autoVerify的activity中的<intent-filter..>的action必须为android.intent.action.VIEW，category必须包含android.intent.category.BROWSABLE，data的scheme必须包含http/https,否则不生效，而且AppLinks必须在Android 6.0 以上的手机才可生效。验证包含以下几方面：
1. 系统会检查包含以下几方面的所有intent-filter
* Action: android.intent.action.VIEW
* Categories: android.intent.category.BROWSABLE and android.intent.category.DEFAULT
* Data scheme: http or https
2. 原话是：For each unique host name found in the above intent filters, Android queries the corresponding websites for the Digital Asset Links file at https://hostname/.well-known/assetlinks.json.。翻译过来就是系统会读取网站的/.well-known/assetlinks.json文件，然后验证包名和签名是否包含在assetlinks.json文件中。

当且仅当上面两个条件满足时才会形成绑定。

#### 3.2 支持多hosts的绑定。
``` xml
<intent-filter>
  ...
  <data android:scheme="https" android:host="www.example.com" />
  <data android:scheme="app" android:host="open.my.app" />
</intent-filter>
```
上面在同一个<intent-filter ..>里面写的两个<data ..>，他们除了组合https://www.example.com和app://open.my.app外app://www.example.com和 https://open.my.app也是满足上面的<intent-filter ..>的。而分开写的时候，不存在上面的问题。

#### 3.3 在网站上创建assetlinks.json文件
具体格式如下：
```
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```
其中package_name就是应用的包名，sha256_cert_fingerprints为正式版的签名。上传时修改这两个属性值。如果一个应用对于多个网站时，可以配置多个对象，配置如下：
```
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.puppies.app",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
  },
  {
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.monkeys.app",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```
然后将该文件放在网站的.well-known目录下，放了之后要试试能不能用：
```
https://domain.name/.well-known/assetlinks.json
```
提交完后确定以下几个是否正确：
```
Be sure of the following:

*   The `assetlinks.json` file is served with content-type `application/json`.
*   The `assetlinks.json` file must be accessible over an HTTPS connection, regardless of whether your app's intent filters declare HTTPS as the data scheme.
*   The `assetlinks.json` file must be accessible without any redirects (no 301 or 302 redirects) and be accessible by bots (your `robots.txt` must allow crawling `/.well-known/assetlinks.json`).
*   If your app links support multiple host domains, then you must publish the `assetlinks.json` file on each domain. See [Supporting app linking for multiple hosts](https://developer.android.com/training/app-links/verify-site-associations.html#multi-host).
*   Do not publish your app with dev/test URLs in the manifest file that may not be accessible to the public (such as any that are accessible accessible only with a VPN). A work-around in such cases is to [configure build variants](https://developer.android.com/studio/build/build-variants.html) to generate a different manifest file for dev builds
```
还有AppLinks仅支持https的网站。

#### 3.4 测试AppLinks
1. 测试json文件是否正确，请看 [Statement List Generator and Tester](https://developers.google.com/digital-asset-links/tools/generator)
也可以采用以下链接进行验证：
```
https://digitalassetlinks.googleapis.com/v1/statements:list?
   source.web.site=https://domain.name:optional_port&
   relation=delegate_permission/common.handle_all_urls
```
2. 测试intent是否正确
可以使用adb进行测试，命令如下：
```
adb shell am start -a android.intent.action.VIEW \
    -c android.intent.category.BROWSABLE \
    -d "http://domain.name:optional_port"
```
下面命令测试已经存在的绑定：
```
adb shell dumpsys package domain-preferred-apps
```
上面命令等价于：
```
adb shell dumpsys package d
```
如果存在绑定的会显示如下结果：
```
Package: com.android.vending
Domains: play.google.com market.android.com
Status: always : 200000002
```
参数含义如下：
```
* Package - Identifies an app by its package name, as declared in its manifest.
* Domains - Shows the full list of hosts whose web links this app handles, using blank spaces as delimiters.
* Status - Shows the current link-handling setting for this app. An app that has passed verification, and 
whose manifest contains android:autoVerify="true", shows a status of always. The hexadecimal number after 
this status is related to the Android system's record of the user’s app linkage preferences. This value does 
not indicate whether verification succeeded.
```

### 4. 总结
1. 优点：
* 不会弹选择框
* 可以直接通过url跳到对应的activity

2. 缺点：
* 网站需要支持https
* 有个校验过程，步骤麻烦些。

使用该机制可以直接绕过intent方式，直接通过url就能打开对应的界面。不过在设置中还是能关闭这个。目前支持该功能的应用和网站还是很少。