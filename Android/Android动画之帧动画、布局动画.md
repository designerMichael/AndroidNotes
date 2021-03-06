---
title: Android动画之帧动画、布局动画
date: 2016-7-16 18:59:25
categories: Android
tags: Android
---

Android动画分四种：
> 
1. Frame Animation 帧动画（Drawable Animation）
2. Layout Animation 布局动画
3. View Animation 视图动画（补间动画）
4. Property Animation 属性动画
<!--more-->

# 帧动画
帧动画，就像GIF图片，通过一系列Drawable依次显示来模拟动画的效果。Frame动画可以被定义在XML文件中，也可以完全编码实现。（一般都是定义在xml中）
在res-drawable目录下新建xml文件，内容如下：
```java
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
android:oneshot="true">
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```
注意点：必须以为根元素，以表示要轮换显示的图片，duration属性表示各项显示的时间,onshot如果定义为true的话，此动画只会执行一次，如果为false则一直循环。XML文件要放在/res/drawable/目录下.

然后在Activity中：
```java
imageView = (ImageView) findViewById(R.id.imageView1);
imageView.setBackgroundResource(R.drawable.drawable_anim);
anim = (AnimationDrawable) imageView.getBackground();
anim.start();
anim.stop();
```
注意start方法不能放在onCreate或者onStart中，因为可能窗口Window对象还没有完全初始化，`AnimationDrawable`不能完全追加到窗口Window对象中。你可以把start方放到`onWindowFocusChanged`方法中。

当然你也可以代码实现：
也可以代码实现：
```java
AnimationDrawable anim = new AnimationDrawable();  
for (int i = 1; i <= 4; i++) {  
    int id = getResources().getIdentifier("f" + i, "drawable", getPackageName());  
    //getDrawable(id) android5.1.1 api22 过期    
    Drawable drawable = getResources().getDrawable(id);  
    anim.addFrame(drawable, 300);  
}  
anim.setOneShot(false);  
image.setBackgroundDrawable(anim);  
anim.start();
```

# 布局动画
主要有两个类：`LayoutAnimation` 和` LayoutTransition`。
`LayoutAnimation`是API Level 1 就已经有的，`LayoutAnimation`是对于ViewGroup控件所有的child view的操作，也就是说它是用来控制ViewGroup中所有的child view 显示的动画。
`LayoutTransition` 是API Level 11 才出现的。`LayoutTransition`的动画效果，只有当ViewGroup中有View添加、删除、隐藏、显示的时候才会体现出来。
先来看`LayoutAnimation`
## LayoutAnimation
`LayoutAnimation`用于为一个layout里面的控件，或者是一个ViewGroup里面的控件设置动画效果，可以在XML文件中设置，亦可以在Java代码中设置。

1.首先在res/anim文件夹下新建一个XML文件,名为list_anim_layout.xml
```java
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"  
android:delay="30%"  
android:animationOrder="reverse"  
android:animation="@anim/slide_right" />
```
属性解释：
> 
Android:delay——子类动画时间间隔为70%，也可以是一个浮点数 如“1.2”等，单位为秒。
android:animationOrder=”random”——子类的进入显示方式，有三种方式：normal表示默认，random表示随机，reverse表示倒序。
android:animation=”@anim/slide_right”——表示孩子显示时的具体动画是什么

2.在res/anim文件夹下新建一个XML文件，名为slide_right,即上面用到的文件
```java
<set xmlns:android="http://schemas.android.com/apk/res/android"   
	android:interpolator="@android:anim/accelerate_interpolator">  
<translate android:fromXDelta="-100%p" android:toXDelta="0"  
	android:duration="@android:integer/config_shortAnimTime" />  
</set>
```
3.使用：
可以直接在XML中为控件添加如下配置：
```java
android:layoutAnimation="@anim/list_anim_layout"//即第一步的布局文件。
```
也可以通过Java代码设置：
```java
//通过加载XML动画设置文件来创建一个Animation对象；
Animation animation=AnimationUtils.loadAnimatio(this,R.anim.slide_right);   //得到一个LayoutAnimationController对象；
LayoutAnimationController controller = new LayoutAnimationController(animation);   
//设置控件显示的顺序；
controller.setOrder(LayoutAnimationController.ORDER_REVERSE); 
//设置控件显示间隔时间；
controller.setDelay(0.3);  
//为ListView设置LayoutAnimationController属性；
listView.setLayoutAnimation(controller);
listView.startLayoutAnimation();
```

## LayoutTransition
`LayoutTransition`的动画效果都是设置给ViewGroup，当被设置动画的ViewGroup中添加删除View时体现出来。该类用于当前布局容器中有View添加、删除、隐藏、显示等时候定义布局容器自身的动画和View的动画，也就是说当在一个LinerLayout中隐藏一个View的时候，我们可以自定义 整个由于LinerLayout隐藏View而改变的动画，同时还可以自定义被隐藏的View自己消失时候的动画等。

### LayoutTransition类中主要有五种容器转换动画类型
具体如下：
> 	
• LayoutTransition.APPEARING 当一个View在ViewGroup中出现时，对此View设置的动画
	• LayoutTransition.CHANGE_APPEARING 当一个View在ViewGroup中出现时，对此View对其他View位置造成影响，对其他View设置的动画
	• LayoutTransition.DISAPPEARING  当一个View在ViewGroup中消失时，对此View设置的动画
	• LayoutTransition.CHANGE_DISAPPEARING 当一个View在ViewGroup中消失时，对此View对其他View位置造成影响，对其他View设置的动画
LayoutTransition.CHANGE 不是由于View出现或消失造成对其他View位置造成影响，然后对其他View设置的动画。

要注意动画到底设置在谁身上，此View还是其他View。

### 使用系统提供的默认的LayoutTransition动画
XML方式：
`android:animateLayoutChanges=”true”`
Java方式：
```java
LayoutTransition mTransitioner = new LayoutTransition();  
mViewGroup.setLayoutTransition(mTransitioner); 
```
在ViewGroup添加如上xml属性默认是没有任何动画效果的，因为前面说了，该动画针对于ViewGroup内部东东发生改变时才有效，所以当我们设置如上属性然后调用 ViewGroup的addView、removeView方法时就能看见系统默认的动画效果了。

### 自定义LayoutTransition动画
上述方式设置动画，默认五种动画类型全都开启，也可以选择开启需要的动画类型：
```java
mTransition = new LayoutTransition();
//开启两种动画
mTransition.setAnimator(LayoutTransition.APPEARING,
        mTransition.getAnimator(LayoutTransition.APPEARING));
mTransition.setAnimator(LayoutTransition.DISAPPEARING,
        mTransition.getAnimator(LayoutTransition.DISAPPEARING));
//设置给viewgroup
mGridLayout.setLayoutTransition(mTransition);
```

如果不喜欢系统默认的动画，也可以自定义动画，然后设置给相应的类型

```java
mTransitioner = new LayoutTransition();
......
ObjectAnimator anim = ObjectAnimator.ofFloat(this, "scaleX", 0, 1);
......//设置更多动画
mTransition.setAnimator(LayoutTransition.APPEARING, anim);
......//设置更多类型的动画                
mViewGroup.setLayoutTransition(mTransitioner);
```