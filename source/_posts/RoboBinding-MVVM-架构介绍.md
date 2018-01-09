---
title: RoboBinding(MVVM)架构介绍
date: 2018-01-03 23:08:14
categories: 设计模式
tags: RoboBinding,MVC,MVP
---
[TOC]
### 1. 什么是RoboBinding?
<b>同步差异</b> - RoboBinding 是一个数据绑定器。<a href="http://martinfowler.com/eaaDev/PresentationModel.html"> Presentation Model framework（著名的MVVM框架）</a>是安卓上的一个框架. RoboBinding帮助你在写UI代码时能方便的阅读、测试和维护。
### 2.AndroidMVVM(Presentation Model)框架
通过RoboBinding建立的一个activity由三个部分组成：layout布局，activity类，还有一个演示模型类。这里有个简单的事例<a href="https://github.com/RoboBinding/AndroidMVVM">AndroidMVVM</a>。
#### 2.1. Layout布局
在布局文件总，我们声明一下RoboBinding的命名空间，属性和一些事件的attribute绑定。例如activity_main.xml
``` xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:tools="http://schemas.android.com/tools"
xmlns:bind="http://robobinding.org/android">
<TextView
bind:text="{hello}" />
...
<Button
android:text="Say Hello"
bind:onClick="sayHello"/>
</LinearLayout>
```
#### 2.2. Presentation Model(简单的Java对象)
在演示层中，我们声明一个对应的属性和方法。代码如下org.robobinding.androidmvvm.PresentationModel.java
``` java
org.robobinding.annotation.PresentationModel
public class PresentationModel implements HasPresentationModelChangeSupport {
private String name;
public String getHello() {
return name + ": hello Android MVVM(Presentation Model)!";
}
...
public void sayHello() {
firePropertyChange("hello");
}
}
```
#### 2.3. Activity类
通过RoboBinding框架将布局和演示层的model绑定在activity界面上。
org.robobinding.androidmvvm.MainActivity.java
``` java
public class MainActivity extends Activity {
@Override
protected void onCreate(Bundle savedInstanceState) {
...
PresentationModel presentationModel = new PresentationModel();
View rootView = Binders.inflateAndBindWithoutPreInitializingViews(this, R.layout.activity_main, presentationModel);
setContentView(rootView);
}
}
```
这样一个简单的AndroidMVVM事例就写完了。我们使用工具类Binders将UI空间和数据Model很简单的进行绑定。在真正使用时我们推荐使用<code>org.robobinding.binder.BinderFactoryBuilder</code>类。我们可以将BinderFactoryBuilder以单例模式保存在整个工程中，也可以使用第三方法库，例如 <a href="https://github.com/roboguice/roboguice">RoboGuice</a> ，来帮忙管理BinderFactory对象实例，这样可以在全局共享和重复使用该实例。
#### 2.4. 介绍视频
<code>2012年2月</code> Robert Taylor 录了一个介绍RoboBinding的视频，<a href="http://skillsmatter.com/podcast/os-mobile-server/core-dev-talk-robobinding">下载地址</a>.
<code>2014年7月</code>Cheng Wei录了一个中文的介绍视频，<a href="https://www.youtube.com/watch?v=2sSBVaX77xA">下载地址</a>.
### 3. 环境设置
#### 3.1. Eclipse下安装使用
安装Eclipse.
##### 3.1.1. 没有AspectJ时
将<em>robobinding-[version]-with-dependencies.jar</em> 或者<em>robobinding-[version].jar + [google guava]-[11.0.1+].jar</em> 放到工程的libs目录下，然后把他们添加到classpath上。点击顺序： **project→Properties→Java Build Path→Libraries→Add JARs**.在展开的列表上点击对应的jar包。
![Alt text](http://robobinding.github.io/RoboBinding/images/eclipse_build_settings.png "Figure 1. Project build settings(An example using robobinding-[version]-with-dependencies.jar)")

##### 3.1.2. 在有AspectJ时
RoboBinding能帮我们减少一些自动绑定的代码。首先安装[Android Development Tools](http://developer.android.com/tools/sdk/eclipse-adt.html) 插件。然后右击项目→Configure→Convert to AspectJ Project. 这样就可以将AspectJ特性加入到项目当中。
按同样的方法将<code> robobinding-[version]-with-aop-and-dependencies.jar </code>或者<code> the robobinding-[version]-with-aop.jar + [google guava]-[11.0.1+].jar</code>加入的libs目录下，然后添加的项目的classpath中。
最后确保RoboBinding的jar是加入到Aspect路径中，具体方法如下：
![Figure 2. AspectJ settings(An example using robobinding-[version]-with-aop-and-dependencies.jar)](http://robobinding.github.io/RoboBinding/images/eclipse_aspectj_settings.png "Figure 2. AspectJ settings(An example using robobinding-[version]-with-aop-and-dependencies.jar)")
##### 3.1.3. Annotaton Processing 设置
下载RoboBinding<code>codegen-[version]-with-dependencies.jar</code>的jar包。按如下方式添加到工程，注意项目不要依赖这个jar。
![Figure 3. Annotaton processing settings](http://robobinding.github.io/RoboBinding/images/eclipse_annotation_processing_settings.png)
#### 3.2. Android Studio安装方法
##### 3.2.1. 没有AspectJ时
将robobinding依赖添加到gradle.build中。
``` xml
dependencies {
...
compile"org.robobinding:robobinding:${robobindingVersion}"

//alternatively we can use with-dependencies jar(RoboBinding provide a minimal Proguarded with-dependencies jar.).
compile("org.robobinding:robobinding:${robobindingVersion}:with-dependencies") {
exclude group: 'com.google.guava', module: 'guava'
}
}
```
[RoboBinding](https://github.com/RoboBinding)是完全免费的，可以随意引用,例如：AndroidMVVM, RoboBinding-album-sample 和 RoboBinding-gallery.

##### 3.2.2. 有AspectJ时
在gradle.build中应用RoboBinding的Android aspcetj插件。
``` xml
buildscript {
repositories {
...
maven() {
name 'RoboBinding AspectJPlugin Maven Repository'
url "https://github.com/RoboBinding/RoboBinding-aspectj-plugin/raw/master/mavenRepo"
}
}

dependencies {
...
classpath 'org.robobinding:aspectj-plugin:0.8.+'
}
}

...
apply plugin: 'org.robobinding.android-aspectj'
```
添加RoboBinding 依赖到 gradle.build中
``` xml
dependencies {
...
compile "org.robobinding:robobinding:$robobindingVersion"
aspectPath "org.robobinding:robobinding:$robobindingVersion"

//alternatively we can use with-aop-and-dependencies jar(RoboBinding provides a minimal Proguarded with-aop-and-dependencies jar.).
compile ("org.robobinding:robobinding:$robobindingVersion:with-aop-and-dependencies") {
exclude group: 'com.google.guava', module: 'guava'
}
aspectPath ("org.robobinding:robobinding:$robobindingVersion:with-aop-and-dependencies") {
exclude group: 'com.google.guava', module: 'guava'
}
}
```
[RoboBinding](https://github.com/RoboBinding)是完全免费的，可以随意引用,例如：AndroidMVVM, RoboBinding-album-sample 和 RoboBinding-gallery.
##### 3.2.3. Annotation Processing 设置
在gradle.build中添加apt插件。
``` xml
buildscript {
repositories {
...
}

dependencies {
...
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.+'
}
}

...
apply plugin: 'com.neenbedankt.android-apt'
```
#### 3.3. ProGuard设置
在proguard中需要保存PresentationModel和生成的构造方法，我们也需要保存所以的注解。在proguard配置中进行如下设置：
``` xml
-keepattributes *Annotation*,Signature
-keep,allowobfuscation @interface org.robobinding.annotation.PresentationModel

-keep @org.robobinding.annotation.PresentationModel class * {
public *** *(...);
}

-keep class * implements org.robobinding.itempresentationmodel.ItemPresentationModel{
public *** *(...);
}

-keep class * extends org.robobinding.presentationmodel.AbstractPresentationModelObject{
public <init>(...);
}

-keep class * extends org.robobinding.presentationmodel.AbstractItemPresentationModelObject{
public <init>(...);
}
```
添加如下几行以便保持view监听的构造函数。
``` xml
-keepclassmembers class * implements org.robobinding.viewattribute.ViewListeners {
public <init>(...);
}
```
添加如下几行来抑制Google的javax.annotation.XX 引用警告。
``` xml
-dontwarn javax.annotation.**
```
在如下链接中有相关样例 [RoboBinding organization](https://github.com/RoboBinding/).
### 4. 概念和特性
![Figure 4. A RoboBinding-based Android application](http://robobinding.github.io/RoboBinding/images/robobinding_based_app.png "Figure 4. A RoboBinding-based Android application")
一个安卓应用中包含若干个activity界面和其他一些元素。在以RoboBinding架构编写的安卓应用中，一个activity包含一个activity类，一个layout布局和一个PresentationModel的普通类（而在普通安卓应用中，一个activity仅仅包含activity类和layout布局文件）。原先都在Activity类中编写展示逻辑的现在都提取到一个独立PresentationModel的简单类中。布局中视图控件的展示数据都绑定在PresentationModel的属性上，视图控件的各个事件都绑定在PresentationModel的方法上。RoboBinding可以帮助我们减少或者删除在Activity类中的UI相关代码，通过简单的在layout布局进行绑定。理想的PresentationModel仅仅包含UI展示逻辑，不包含UI代码或者UI连接代码，并且能够被单独的、简单的测试。
这部分样例代码可以通过如下链接查看[Robobinding Gallery](https://github.com/RoboBinding/RoboBinding-gallery/).

#### 4.1. 单向属性绑定
当我们将一根属性绑定到一个presentation模型时，任何对该属性的更改都会直接传给相应的View。
activity_view.xml
``` xml
<TextView
bind:visibility="{integerVisibility}"/>
```
ViewPresentationModel.java
``` java
public int getIntegerVisibility() {
return integerVisibilityRotation.value();
}
```
robobinding遵循JavaBeans规范也提供getters和setters方法。在单方向绑定中，仅仅需要提供getters方法，即view的任何改变不需要更新给对应的presentation模型。更多关于属性绑定的内容可以参考<em><b>API和属性绑定文档</b></em>
#### 4.2. 双向属性绑定
双方向属性绑定顾名思义就是对presentation model的任何改变会反映到对应的view上，同时对view的任何改变也会更改对应的presentation Model。
EditText控件中的text属性是支持双向绑定的。考虑这种情况，无论何时用户更改了EditText的text内容，对应的presentation Model都会相应的进行更改。
双向属性绑定仅仅需要在大括号外加一个美元$符，例如：
activity_edittext.xml
``` xml
<EditText
bind:text="${text}"/>
```
为了支持双向绑定，需要在对应的presentation model中添加setter方法：
org.robobinding.gallery.presentationmodel.EditTextPresentationModel.java
``` java
PresentationModel
public class EditTextPresentationModel {
private String text;

public String getText() {
return text;
}

public void setText(String text) {
this.text = text;
}
}
```
#### 4.3. 事件绑定
将view的事件绑定到presentation model的方法上，可以通过<code>bind:onClick="showDemo"</code>来实现。
activity_gallery.xml
``` xml
<Button
bind:onClick="showDemo"/>
```
org.robobinding.gallery.presentationmodel.GalleryPresentationModel.java
``` java
PresentationModel
public class GalleryPresentationModel
{
...
public void showDemo()
{
...
}
}
```
当点击事件被触发时，presentation Model绑定的方法将被执行。我们能选择性的支持一个事件的参数，在上面例子中对应的参数是<code>org.robobinding.widget.view.ClickEvent</code>
更多关于UI事件绑定可以参考<em><b>API和属性绑定文档</b></em>
#### 4.4. AdapterViews的绑定
当我们绑定一个AdapterViews时，RoboBinding首先要求我们指出对于presentation model基础的数据。可以是以下几个类型：an Array, List or org.robobinding.itempresentationmodel.TypedCursor.
在有基础数据后，RoboBinding需要知道presentation模型和子view对应的绑定关系。我们可以通过 @ItemPresentationModel注解实现。例如：
activity_adapter_view.xml
``` xml
<ListView
bind:itemLayout="@android:layout/simple_list_item_1"
bind:itemMapping="[text1.text:{value}]"
bind:source="{dynamicStrings}"/>
```
org.robobinding.gallery.presentationmodel.AdapterViewPresentationModel.java
``` java
PresentationModel
public class AdapterViewPresentationModel
{
...
@ItemPresentationModel(value=StringItemPresentationModel.class)
public List<String> getDynamicStrings()
{
return getSelectedSource().getSample();
}
···
```
提供的基础数据需要继承自ItemPresentationModel 接口。
org.robobinding.gallery.presentationmodel.StringItemPresentationModel.java
``` java
public class StringItemPresentationModel implements ItemPresentationModel<String>
{
private String value;

@Override
public void updateData(int index, String bean)
{
value = bean;
}

public String getValue()
{
return value;
}
}
```
然后我们给每一个条目行定义一个布局。在该例中我们使用Android库给的simple_list_item_1.xml布局。通过<code>bind:itemMapping="[text1.text:{value}]"</code>进行绑定我们将simple_list_item_1.xml中的text1.text绑定到StringItemPresentationModel.value中。
在@ItemPresentationModel中有一个工厂方法属性(<code>factoryMethod</code>)。当ItemPresentationModels有一些扩展的依赖时，我们能增加一个工厂方法(<code>factoryMethod</code>)到PresentationModel里面以便ItemPresentationModels能通过该方法进行创建。通过这种方式，我们能将任何依赖让到ItemPresentationModel中，通过完全自由的方式。下面是一个简单的例子。
``` java
@PresentationModel
public class PresentationModelSample
{
...
@ItemPresentationModel(value=ItemPresentationModelSample.class, factoryMethod="createItemPresentationModelSample")
public List<String> getDynamicStrings()
{
return getSelectedSource().getSample();
}

public ItemPresentationModelSample createItemPresentationModelSample() {
return ItemPresentationModelSample(dependency1, dependency2, ...);
}
```
#### 4.5. 轻量级关系cursor游标映射对应的对象cursor游标

在AdapterViews绑定中提到数据类型<code>org.robobinding.itempresentationmod el.TypedCursor</code>。由于我们经常操作对象甚过操作对象数据，因而我们希望能将操作关系数据库的代码进行分离。RoboBinding增加了一个轻量级的对象cursor游标-<code>TypedCursor</code>。通过<code>org.robobinding.itempresentationmodel.RowMapper</code>类，我们能将一条数据转换为一个对象。示例如下：
org.robobinding.gallery.presentationmodel.TypedCursorPresentationModel.java
``` java
@PresentationModel
public class TypedCursorPresentationModel {
...
@ItemPresentationModel(value=ProductItemPresentationModel.class)
public TypedCursor<Product> getProducts() {
return allProductsQuery.execute(db);
}
}
```
org.robobinding.gallery.model.typedcursor.GetAllQuery.java
``` java
public class GetAllQuery<T>
{
private String tableName;
private final RowMapper<T> rowMapper;

public GetAllQuery(String tableName, RowMapper<T> rowMapper)
{
...
this.tableName = tableName;
this.rowMapper = rowMapper;
}

public TypedCursor<T> execute(SQLiteDatabase db)
{
Cursor cursor = db.query(
tableName,
null,
null,
null,
null,
null,
BaseColumns._ID+" ASC");
return new TypedCursorAdapter<T>(cursor, rowMapper);
}
}
```
org.robobinding.gallery.model.typedcursor.ProductRowMapper.java
``` java
public class ProductRowMapper implements RowMapper<Product> {

@Override
public Product mapRow(Cursor cursor) {
String name = cursor.getString(cursor.getColumnIndex(ProductTable.NAME));
String description = cursor.getString(cursor.getColumnIndex(ProductTable.DESCRIPTION));
return new Product(name, description);
}

}
```
#### 4.6. 菜单绑定
将一个res/menu中的菜单资源绑定到一个Presentation Models中。看如下例子：
res/menu/context_menu.xml
``` xml
<menu xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:bind="http://robobinding.org/android"
xmlns:app="http://schemas.android.com/apk/res-auto">
<item android:title="Delete Product"
bind:onMenuItemClick="deleteProduct"
android:id="@+id/deleteProduct"
app:showAsAction="always"/>

</menu>
```
org.robobinding.gallery.presentationmodel.ContextMenuPresentationModel.java
``` java
@PresentationModel
public class ContextMenuPresentationModel {
...
public void deleteProduct(MenuItem menuItem) {
...
}
}
```
#### 4.7. Presentation Models（演示models）
我们将任何一个presentationModel添加<code>@org.robobinding.annotation.PresentationModel</code>注解。当一个presentationmodel需要<code>org.robobinding.presentationmodel.PresentationModelChangeSupport</code>支持时，presentationmodel必须继承自<code>org.robobinding.presentationmodel.HasPresentationModelChangeSupport </code>接口，以便于框架层能在内部使用<code>PresentationModelChangeSupport</code>对象。

有两种方式可以实现一个Presentationmodel，一个是有AspectJ时，一个是没有AspectJ时，
##### 4.7.1.没有 AspectJ

* 使用robobinding-[version].jar或者robobinding-[version]-with-dependencies.jar库
* 这种方式的好处是没有额外的依赖，可以使最终的apk尽可能的小
* 不好的地方在我们需要手动设置每一个firePropertyChange("propertyName").
以下有具体的样例
[AndroidMVVM](https://github.com/RoboBinding/AndroidMVVM) 和 [Android-CleanArchitecture](https://github.com/RoboBinding/Android-CleanArchitecture) 

##### 4.7.2. 有 AspectJ

* 使用robobinding-[version]-with-aop.jar 或者robobinding-[version]-with-aop-and-dependencies.jar库

* 好处是许多firePropertyChange("propertyName") 可以自动生成

* 不好的地方是在AspectJ运行库里面包含依赖，这轻微的增加了最终apk的大小。
以下是具体样例
[Album Sample](https://github.com/RoboBinding/RoboBinding-album-sample) 和 [Gallery](https://github.com/RoboBinding/RoboBinding-gallery)

### 5. 创造我们自己的view绑定

这一节的样例代码是： [Robobinding Gallery](https://github.com/RoboBinding/RoboBinding-gallery/).

#### 5.1. 单向绑定
在如下的例子中我们增加了View的enable属性绑定- [source code](https://github.com/RoboBinding/RoboBinding-gallery/blob/master/app/src/main/java/org/robobinding/gallery/activity/ViewBindingForView.java).
``` java
@ViewBinding(simpleOneWayProperties = {"enabled"})
public class ViewBindingForView extends CustomViewBinding<View> {
}
```
然后添加到<code> BinderFactoryBuilder</code> - [source code](https://github.com/RoboBinding/RoboBinding-gallery/blob/master/app/src/main/java/org/robobinding/gallery/activity/GalleryApp.java).
``` java
new BinderFactoryBuilder()
.add(new ViewBindingForView().extend(View.class))
.build();
```
RoboBinding能够生成如下代码：
``` java
public class ViewBindingForView$$VB implements ViewBinding<View>{
final ViewBindingForView customViewBinding;

public ViewBindingForView$$VB(ViewBindingForView customViewBinding) {
this.customViewBinding = customViewBinding;
}

@Override
public void mapBindingAttributes(BindingAttributeMappings<View> mappings) {
mappings.mapOneWayProperty(ViewBindingForView$$VB.EnabledAttribute.class, "enabled");
customViewBinding.mapBindingAttributes(mappings);
}

public static class EnabledAttribute implements OneWayPropertyViewAttribute<View, Boolean>
{
@Override
public void updateView(View view, Boolean newValue) {
view.setEnabled(newValue);
}
}
}
```
所以的绑定都是静态static的，这意味着每一性能的影响。
#### 5.2. 其他类型的绑定
除了单向属性绑定外，这里还有事件event绑定，多类型属性绑定和其他组属性类型的绑定。为了能实现这些绑定，我们需要扩展CustomViewBinding和手动实现对应的继承。具体例子如下：
``` java
@ViewBinding
public class MyCustomViewBinding extends CustomViewBinding<CustomView> {
@Override
public void mapBindingAttributes(BindingAttributeMappings<CustomView> mappings) {
mappings.mapEvent(OnCustomEventAttribute.class, "onCustomEvent");
}

public class OnCustomEventAttribute implements EventViewAttributeForView {
...
}
}
```
对于这些绑定，我们能在如下链接找到例子：- [source code under widget package](https://github.com/RoboBinding/RoboBinding/tree/develop/framework/src/main/java/org/robobinding/widget).

#### 5.3. 自定义控件或者第三方控件
我们能对于任何自定义控件、第三方控件、Android widgets创建view绑定，以便更容易的使用它们。在RoboBinding中，这些绑定是一样的。
![Figure 5. custom Title Description Bar](http://robobinding.github.io/RoboBinding/images/custom_component.png "Figure 5. custom Title Description Bar")

下面是一个简单的自定义控件的例子，该View有一个白色的边框。这个控件有一个标题和一个描述组成。当我们输入新的标题和描述后，然后点击应用（Apply），控件自己讲对于的更新。

这个需求可以用以下简单的代码实现：
activity_custom_component.xml
``` xml
<org.robobinding.gallery.model.customcomponent.TitleDescriptionBar
bind:title="{title}"
bind:description="{description}"/>
```
这个TitleDescriptionBar控件的代码如下：(对于如何实现一个自定义控件，参考如下：[Android Reference](http://developer.android.com/guide/topics/ui/custom-components.html)):
``` java
public class TitleDescriptionBar extends LinearLayout {
private TextView title;
private TextView description;

public TitleDescriptionBar(Context context, AttributeSet attrs) {
this(context, attrs, R.layout.title_description_bar);
}

protected TitleDescriptionBar(Context context, AttributeSet attrs, int layoutId) {
super(context, attrs);

LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
inflater.inflate(layoutId, this);
title = (TextView) findViewById(R.id.title);
description = (TextView) findViewById(R.id.description);
...
}

public void setTitle(CharSequence titleText) {
title.setText(titleText);
}

public void setDescription(CharSequence descriptionText) {
description.setText(descriptionText);
}
}
```
对于的布局是： title_description_bar.xml:
``` xml
<merge xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:bind="http://robobinding.org/android">
<TextView android:id="@+id/title"/>
<TextView android:text=": "/>
<TextView android:id="@+id/description"/>
```
##### 5.3.1. 实现属性绑定
这个控件有两个简单的单向属性绑定- [source code](https://github.com/RoboBinding/RoboBinding-gallery/blob/master/app/src/main/java/org/robobinding/gallery/model/customcomponent/TitleDescriptionBarBinding.java).
``` java
@ViewBinding(simpleOneWayProperties = {"title", "description"})
public class TitleDescriptionBarBinding extends CustomViewBinding<TitleDescriptionBar> {
}
```
##### 5.3.2. 注册ViewBindings

ViewBindings能够通过<code>org.robobinding.binder.BinderFactoryBuilder </code>进行绑定- [source code](https://github.com/RoboBinding/RoboBinding-gallery/blob/master/app/src/main/java/org/robobinding/gallery/activity/CustomComponentActivity.java).
``` java
BinderFactory binderFactory = new BinderFactoryBuilder()
.add(new TitleDescriptionBarBinding().forView(TitleDescriptionBar.class))
.build();
```
这样就行了。我们能用同样的方法绑定第三方控件或者Android Widgets。

#### 5.4. 替换已有的view绑定
当一个已经存在的view绑定不能满足我们需求时，我们通过注册去替代默认的继承implementations.
``` java
BinderFactory binderFactory = new BinderFactoryBuilder()
.add(new MyViewBindingForView().forView(View.class))
.build();

@ViewBinding
static class MyViewBindingForView extends CustomViewBinding<View> {
...
}
```
##### 5.4.1. 扩展一个已经存在的view绑定

扩展一个已经存在的TextViewBinding，并且增加一个属性绑定。
``` java
BinderFactory binderFactory = new BinderFactoryBuilder()
.add(new MyTextViewBinding().extend(TextView.class))
.build();

@ViewBinding(simpleOneWayProperties={"typeface"})
static class MyTextViewBinding extends CustomViewBinding<TextView> {
...
}
```
### 6. Album例子的具体步骤
Album工程是一个Martin Fowler的翻译版本[original one](http://martinfowler.com/eaaDev/PresentationModel.html). 源代码如下： [这里](https://github.com/RoboBinding/RoboBinding-album-sample).
![album_sample_prototype.png](http://robobinding.github.io/RoboBinding/images/album_sample_prototype.png "Figure 6. Album Sample project prototype")
在接下来的几节，包名是已org.robobinding.albumsample开头的

上面是这个项目的原型。这个项目遵循RoboBinding工程的架构。一个Activity由一个Activity类，布局.xml和一个原型Model组成。在项目中你能看到如下几个包：org.robobinding.albumsample.activity,该包包含所有的Activity界面类；org.robobinding.albumsample.presentationmodel,该包包含所有的原型类；org.robobinding.albumsample.model,该包包含所有的Album实现类；org.robobinding.albumsample.store,该包包含所有的AlbumStore实现类。在原型中有5个表格，分别代表如下：
[Home Activity]表格由.activity.HomeActivity,home_activity.xml和.presentationmodel.HomePresentationModel组成。
[View Albums Activity]由.activity.ViewAlbumsActivity, view_albums_activity.xml和.presentationmodel.ViewAlbumsPresentationModel组成; 每一个album条目由.presentationmodel.AlbumItemPresentationModel 和album_row.xml来构成; 当album列表是空时，用albums_empty_view.xml布局来代替。
[Create Album Activity]和[Edit Album Activity]两者都使用.activity.CreateEditAlbumActivity, create_edit_album_activity.xml 和.presentationmodel.CreateEditAlbumPresentationModel来构成。
[View Album Activity].activity.ViewAlbumActivity, view_album_activity.xml 和.presentationmodel.ViewAlbumPresentationModel组成; 它的album 删除对话框.activity.DeleteAlbumDialog, delete_album_dialog.xml和 .presentationmodel.DeleteAlbumDialogPresentationModel构成。
把[View Albums Activity]作为一个简单的例子来阐述如何使用RoboBinding。ViewAlbumsActivity是关联到view_albums_activity.xml和ViewAlbumsPresentationModel上面的。view_albums_activity.xml包含3个子View，一个TextView，一个ListView和一个Button。这个TextView不包含任何绑定信息。在ListView中<code>bind:source="{albums}"</code>绑定到 ViewAlbumsPresentationModel.albums dataset property. <code>bind:onItemClick="viewAlbum"</code> 绑定到ViewAlbumsPresentationModel.viewAlbum(ItemClickEvent) 方法上. 当album的一个条目被点击时，这个方法将被调用。当数据为空时会使用albums_empte_view.xml布局<code>bind:emptyViewLayout="@layout/albums_empty_view"</code> 。<code>bind:itemLayout="@layout/album_row"</code>对应每一行布局，在该布局中有两个TextView，绑定信息如下：<code>bind:text="{title}"</code> and <code>bind:text="{artist}"</code> 绑定到AlbumItemPresentationModel.title/artist respectively.在view_albums_activity.xml布局中的子布局是一个 Button. 它的点击事件<code>bind:onClick="createAlbum"</code> 绑定到ViewAlbumsPresentationModel.createAlbum()方法上。

### 7. Gallery Demos的组成

The entry classes mentioned below are from the package org.robobinding.gallery.activity of Robobinding Gallery project.

* Binding attributes demo for View. The entry class is ViewActivity.

* Binding attributes demo for EditText. The entry class is EditTextActivity.

* Binding attributes demo for AdapterView. The entry class is AdapterViewActivity.

* Binding attributes demo for ListView. The entry class is ListViewActivity.

* Binding attributes demo for RecyclerView. The entry class is RecyclerViewActivity.

* Binding attributes demo for Custom Components. The entry class is CustomComponentActivity.

* Demo for Object Cursor. The entry class is TypedCursorActivity.

* Demo for Fragment & ViewPager Binding. The entry class is ListFragmentDemoActivity.

* Demo for Options Menu Binding. The entry class is OptionsMenuActivity.

* Demo for Context Menu Binding. The entry class is ContextMenuDemoActivity.

* Demo for Contextual Action Mode Binding. The entry class is ContextualActionModeActivity.

### 8. 项目结构和项目实践

Involved from MVC pattern, the major motive of Presentation Model(MVVM) pattern is to further decouple UI state and logic into a pure POJO Presentation Model, which can be easily Unit tested. Meanwhile, the dependency of View→Presentation Model→Model becomes unidirectional. When applying the pattern, these are the basic rules we will follow. [Album Sample](https://github.com/RoboBinding/RoboBinding) is an example that follows the best practices. Recommend to read Martin Fowler’s original article on [Presentation Model](http://martinfowler.com/eaaDev/PresentationModel.html).
#### 8.1. 整个项目框架
![project_structure.png](http://robobinding.github.io/RoboBinding/images/project_structure.png "Figure 7. Project structure")
In Android app, the view layer consists of activities(fragments) and their layouts and the model layer(or business model layer) consists of various services, persistence layer, networking services, business services and so on. The diagram indicates the dependency between different layers. The view layer for example never directly accesses the business model.
#### 8.2. 常见问题解决方案
* When we are not using a third-party dependency injection lib, we may instantiate business model objects in Activities and then pass them into presentation models, but the view layer(or any activities) will not directly access any business model objects.

* Sometimes presentation models may need to call some functionalities in the view layer. We can add view interfaces in between to decouple the relationship. Presentation models depends on view interfaces instead of the view layer, which keeps the testability of presentation models. If you prefer, you can shift these view interfaces into presentation model layer or presentation model package, so that the dependency remains unidirectional. Let us have a look a simple example below:
``` java
interface MainView {
void doSomeViewLogic();
}

class MainActivity extends Activity implements MainView {
...
@Override
protected void onCreate(Bundle savedInstanceState) {
...
PresentationModel presentationModel = new PresentationModel(this);
...
}

public void doSomeViewLogic() {
...
}
}

class PresentationModel {
private MainView mainView;

public PresentationModel(MainView mainView) {
this.mainView = mainView;
}

public void someEvent() {
mainView.doSomeViewLogic();
}
}
```
### 9. 其他资源

<strong>Jan 2012</strong> Robert Taylor has written a couple of introductory articles [here](http://roberttaylor426.blogspot.com/2011/11/hello-robobinding-part-1.html) and [here](http://roberttaylor426.blogspot.com/2012/01/hello-robobinding-part-2.html).

<strong>Feb 2012</strong> A video of a talk on RoboBinding at SkillsMatter, London can be found[ here](http://skillsmatter.com/podcast/os-mobile-server/core-dev-talk-robobinding).

<strong>Jul 2014</strong> A video of a talk on RoboBinding in Chinese by Cheng Wei can be found [here](https://www.youtube.com/watch?v=2sSBVaX77xA).

Sep 2014 A talk at [YOW 2014 Android MVVM](http://adilmughal.github.io/YOW2014-Android-MVVM/) by Adil Mughal on <em>Write cleaner, maintainable and testable code in Android using MVVM</em>.

[AndroidMVVM](https://github.com/RoboBinding/AndroidMVVM) A minimal android app with MVVM.

[RoboBinding album sample](https://github.com/RoboBinding/RoboBinding-album-sample) is an Android translation of Martin Fowler’s original sample code on [Presentation Model](http://martinfowler.com/eaaDev/PresentationModel.html) pattern.

[RoboBinding Gallery](https://github.com/RoboBinding/RoboBinding-gallery) demonstrates RoboBinding features.
- - -
Version 0.8.10
Last updated 2015-10-24 16:50:07
