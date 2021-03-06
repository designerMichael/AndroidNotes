---
title: Android屏幕适配
date: 2015-12-20 21:10:25
categories: Android
tags: [Android]
---

# 一些概念

## 屏幕尺寸
> 
• 含义：手机对角线的物理尺寸
• 单位：英寸（inch），1英寸=2.54cm
Android手机常见的尺寸有5寸、5.5寸、6寸等等

<!--more-->

## 屏幕分辨率
> 
• 含义：手机在横向、纵向上的像素点数总和
    1. 一般描述成屏幕的"宽x高”=AxB
    2. 含义：屏幕在横向方向（宽度）上有A个像素点，在纵向方向
    （高）有B个像素点
    3. 例子：1080x1920，即宽度方向上有1080个像素点，在高度方向上有1    920个像素点
• 单位：px（pixel），1px=1像素点
UI设计师的设计图会以px作为统一的计量单位
• Android手机常见的分辨率：320x480、480x800、720x1280、1080x1920
	
## 屏幕像素密度
> 
• 含义：每英寸的像素点数
• 单位：dpi（dots per ich）
假设设备内每英寸有160个像素，那么该设备的屏幕像素密度=160dpi
• 安卓手机对于每类手机屏幕大小都有一个相应的屏幕像素密度：
![](http://oeiu2t0ur.bkt.clouddn.com/12454321.png)
当然有很多"不标准"的分辨率：
![](http://oeiu2t0ur.bkt.clouddn.com/123131.png)

## 屏幕尺寸、分辨率、像素密度三者关系
一部手机的分辨率是宽x高，屏幕大小是以寸为单位，那么三者的关系是：
![](http://oeiu2t0ur.bkt.clouddn.com/213131.png)

## 密度无关像素（dp）
> 
• 含义：density-independent pixel，叫dp或dip，与终端上的实际物理像素点无关。
• 单位：dp，可以保证在不同屏幕像素密度的设备上显示相同的效果
		1. Android开发时用dp而不是px单位设置图片大小，是Android特有的单位
		2. 场景：假如同样都是画一条长度是屏幕一半的线，如果使用px作为计量单位，那么在480x800分辨率手机上设置应为240px；在320x480的手机上应设置为160px，二者设置就不同了；如果使用dp为单位，在这两种分辨率下，160dp都显示为屏幕一半的长度。
• dp与px的转换
因为ui设计师给你的设计图是以px为单位的，Android开发则是使用dp作为单位的，那么我们需要进行转换：
![](http://oeiu2t0ur.bkt.clouddn.com/23546786.png)
> 
在Android中，规定以160dpi（即屏幕分辨率为320x480）为基准：1dp=1px
dp和px的转换公式为： px = dp * (dpi / 160)
举个栗子：
iPad2 和 iPad Retina的物理尺寸都是 9.7 inch，不同的是分辨率和dpi，一个是1024x768 / 132dpi，另一个是2048x1536 / 264dpi，
分别计算一下20dp在两种设备上对应多少inch？
	ipad2 = 20 * (132 / 160) * (9.7/ (math.sqrt(1024 * 1024 + 768 * 768))) 
ipad_retina = 20 * (264 / 160) * (9.7 / (math.sqrt(2048 * 2048 + 1536 * 1536)))
计算结果都是0.1018359375，这就是dp的功能，它能保证在所有的设备上显示的物理尺寸大小都一样。

## 独立比例像素（sp）
> 
• 含义：scale-independent pixel，叫sp或sip
• 单位：sp
		1. Android开发时用此单位设置文字大小，可根据字体大小首选项进行缩放
		2. 推荐使用12sp、14sp、18sp、22sp作为字体设置的大小，不推荐使用奇数和小数，容易造成精度的丢失问题；小于12sp的字体会太小导致用户看不清

# 屏幕适配解决方法
![](http://oeiu2t0ur.bkt.clouddn.com/5656.png)

- 在布局中尽量使用match_parent、wrap_content和weight；
- 少用dp来指定控件的具体长宽（因为并不是所有的屏幕的宽高都具备相同的dp长度）；
- 使用.9图；
- 使用限定符

## 限定符
使得程序在运行时根据当前设备的配置（屏幕尺寸）自动加载合适的布局资源。

限定符类型：

### 尺寸（size）限定符：

- 两套布局文件：res/layout-large/main.xml和res/layout/main.xml   
- 这种方式只适合Android 3.2版本之前。
- large具体是指多大并没有一个定量的标准，所以要用到后面的最小宽度（Smallest-width）限定符

### 最小宽度（Smallest-width）限定符
- 通过指定某个最小宽度（以dp为单位）来精确定位屏幕从而加载不同的UI资源
- 默认布局：res/layout/main.xml；宽度或者高度有一个>=600dp的设备：res/layout-sw600dp/main.xml
- 使用Android 3.2及之后版本

### 布局别名
如果要适配到3.2以前的平板，那么你可能要同时维护layout-large和layout-sw600dp的两套布局，因为前者用于3.2以前，后者适用于3.2以后，但他们的内容是完全一样的，这就很蛋疼，这就要用到布局别名。(现在一般只需设配到4.0了吧，就没这些破事了)
例如，你想手机上显示single-pane界面，而在7寸平板和更大屏的设备上显示multi-pane界面，你需要提供以下文件：

- res/layout/main.xml: single-pane布局
- res/layout-large/main_twopanes.xml : multi-pane布局(用于3.2以前)
- res/layout-sw600dp/main_twopanes.xml: multi-pane布局（用于3.2及以后）

后面这两个文件是完全相同的，为了要解决这种重复，你需要使用别名技巧。例如，你可以定义以下两个布局：

- res/layout/main.xml, single-pane布局
- res/layout/main_twopanes.xml, two-pane布局

然后加入以下两个文件：

- res/values-large/layout.xml文件，内容为：
```java
<resources>  
    <item name="main" type="layout">@layout/main_twopanes</item>  
</resources>  
```
• res/values-sw600dp/layout.xml文件，内容为：
```java
<resources>  
    <item name="main" type="layout">@layout/main_twopanes</item>  
</resources>  
```				
上面两个文件有着相同的内容，但是它们并没有真正去定义布局，它们仅仅只是将main设置成了`@layout/main_twopanes`的别名，这样两个layout.xml都只是引用了`@layout/main_twopanes`，就避免了重复定义布局文件的情况。
		
### 屏幕方向（Orientation）限定符
在前面的基础上加上-land或者-port，表示在横屏时加载这个，在竖屏时加载另一个。

## 建立多个values文件
根据分辨率建立多个values文件，系统会自动根据当前手机分辨率去加载最合适的资源文件。
![](http://oeiu2t0ur.bkt.clouddn.com/20150503174449732.png)
##　Android资源加载过程
> 
Android SDK会根据屏幕密度自动选择对应的资源文件进行渲染加载（自动渲染）。
比如说，SDK检测到你手机的分辨率是320x480（dpi=160），会优先到drawable-mdpi文件夹下找对应的图片资源；但假设你只在xhpdi文件夹下有对应的图片资源文件（mdpi文件夹是空的），那么SDK会去xhpdi文件夹找到相应的图片资源文件，然后将原有大像素的图片自动缩放成小像素的图片，于是大像素的图片照样可以在小像素分辨率的手机上正常显示。
所以理论上来说只需要提供一种分辨率规格的图片资源就可以了。（当然只是理论上）
所以如果只能做一套图，那就做xhdpi分辨率(720*1280)的图
可以设置ImageView的ScaleType属性来得到合适的显示效果，一般CenterCrop最常用




参考
> [详解Android开发中常用的 DPI / DP / SP](http://www.jianshu.com/p/913943d25829)
[Android开发：最全面、最易懂的Android屏幕适配解决方案](http://www.jianshu.com/p/ec5a1a30694b)


