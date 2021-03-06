---
title: Java注解
date: 2016-06-12 10:00:00
categories: Java
tags: 注解
---




# 概念

注解（Annotation），也叫元数据。它是JDK 1.5及以后版本引入的一个特性，与类、接口、枚举是在同一个层次。它可以声明在包、类、字段、方法、局部变量、方法参数等的前面，用来对这些元素进行说明，注释。比如常见的@Override，butterknife库里的@Bind, 该功能并不影响程序的运行，主要作用就是提供类型安全，对编译器警告等辅助工具产生影响。Annotation不能运行，它只有成员变量，没有方法,Annotation不能作为一个程序元素使用。

<!--more-->

# 注解的作用

1、标记作用，用于告诉编译器一些信息让编译器能够实现基本的编译检查，如@Override、Deprecated

2、编译时动态处理，动态生成代码，如Butter Knife、Dagger 2

3、运行时动态处理，获得注解信息，如Retrofit




# 注解的分类

注解的分类有两种分法：

## 第一种分法

1、基本内置注解，是指Java自带的几个Annotation，如@Override、Deprecated、@SuppressWarnings等

2、元注解（meta-annotation），是指负责注解其他注解的注解，JDK 1.5及以后版本定义了4个标准的元注解类型，如下：

	1、@Target  
	2、@Retention  
	3、@Documented  
	4、@Inherited  


3、自定义注解，根据需要可以自定义注解，自定义注解需要用到上面的meta-annotation


##第二种分法

根据作用域分类

1、源码时注解（RetentionPolicy.SOURCE）
2、编译时注解（RetentionPolicy.CLASS）
3、运行时注解（RetentionPolicy.RUNTIME）


# 注解基础


一个简单的Java注解类似与@Entity。其中@interface的意思是告诉编译器这是一个注解。而Entity则是注解的名字。通常在文件中，写法如下

```java
public @interface Entity { 
}
```

# 注解元素

Java注解可以使用元素来进行设置一些值，注解中的元素类似于属性或者参数。定义包含元素的注解示例代码(为什么这么写？看后面的自定义注解)。Annotation只有成员变量，没有方法。Annotation的成员变量在Annotation定义中以“无形参的方法”形式来声明，其方法名定义了该成员变量的名字，其返回值定义了该成员变量的类型

```java
public @interface Entity { 
  String tableName();
}
```

使用包含元素的注解示例代码
```java
@Entity(tableName = "vehicles")
```

上面代码的意思是：注解的元素名称为tableName，设置的值为vehicles。没有元素的注解不需要使用括号，如@Entity

如果注解包含多个元素，使用方法如下
```java
@Entity(tableName = "vehicles", primaryKey = "id")
```

如果注解只有一个元素，通常我们将元素名称设为value：
```java
@InsertNew(value = "yes")
```

因为当且仅当元素名为value,我们可以简写，即不需要填写元素名value，效果如下
```java
@InsertNew("yes")
```

# 注解使用

注解可以用来修饰代码中的以下元素：

- 类
- 接口
- 方法
- 方法参数
- 属性
- 局部变量

下面是一个完整示例：

```java
@Entity //修饰类
public class Vehicle {

    @Persistent  //修饰属性
    protected String vehicleName = null;

    @Getter  //修饰方法
    public String getVehicleName() {
        return this.vehicleName;
    }

    public void setVehicleName(@Optional vehicleName) {//修饰方法参数
        this.vehicleName = vehicleName;
    }

    public List addVehicleNameToList(List names) {

        @Optional  //修饰局部变量
        List localNames = names;

        if(localNames == null) {
            localNames = new ArrayList();
        }
        localNames.add(getVehicleName());

        return localNames;
    }

}
```

# Java基本内置注解

Java中有三种内置注解，这些注解用来为编译器提供指令。它们是
```java
@Deprecated
@Override
@SuppressWarnings
```

## @Deprecated

作用：
  
- 可以用来标记类，方法，属性。
- 如果上述三种元素不再使用，使用@Deprecated注解
- 如果代码使用了@Deprecated注解的类，方法或属性，编译器会进行警告。

使用：

如下为注解一个弃用的类：
```java
@Deprecated
public class MyComponent {

}
```

当我们使用@Deprecated注解后，建议配合使用对应的@deprecated JavaDoc符号，并解释说明为什么这个类，方法或属性被弃用，已经替代方案是什么。

```java
@Deprecated
/**
  @deprecated This class is full of bugs. Use MyNewComponent instead.
*/
public class MyComponent {

}
```

## @Override

@Override注解用来修饰对父类进行重写的方法。如果一个并非重写父类的方法使用这个注解，编译器将提示错误。

实际上在子类中重写父类或接口的方法，@Overide并不是必须的。但是还是建议使用这个注解，在某些情况下，假设你修改了父类的方法的名字，那么之前重写的子类方法将不再属于重写，如果没有@Overide，你将不会察觉到这个子类的方法。有了这个注解修饰，编译器则会提示你这些信息。

使用Override注解的例子

```java
public class MySuperClass {

    public void doTheThing() {
        System.out.println("Do the thing");
    }
}

public class MySubClass extends MySuperClass{

    @Override
    public void doTheThing() {
        System.out.println("Do it differently");
    }
}
```

## @SuppressWarnings

作用：

- @SuppressWarnings用来抑制编译器生成警告信息。
- 可以修饰的元素为类，方法，方法参数，属性，局部变量

使用场景：

当我们一个方法调用了弃用的方法或者进行不安全的类型转换，编译器会生成警告。我们可以为这个方法增加@SuppressWarnings注解，来抑制编译器生成警告。

注意：

使用@SuppressWarnings注解，采用就近原则，比如一个方法出现警告，我们尽量使用@SuppressWarnings注解这个方法，而不是注解方法所在的类。虽然两个都能抑制编译器生成警告，但是范围越小越好，因为范围大了，不利于我们发现该类下其他方法的警告信息。

使用示例：
```java
@SuppressWarnings
public void methodWithWarning() {

}
```


# 元注解

调用下，元注解是用来定义其他注解的注解。
有四种

1、@Target  
2、@Retention  
3、@Documented  
4、@Inherited  

## @Target

使用@Target注解，我们可以设定自定义注解可以修饰哪些java元素。简单示例

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;

@Target({ElementType.METHOD})
public @interface MyAnnotation {
    String   value();
}
```

上述的代码说明MyAnnotation注解只能修饰方法。

@Target可以选择的参数值有如下这些

- ElementType.ANNOTATION_TYPE（注：修饰注解）
- ElementType.CONSTRUCTOR(构造函数)
- ElementType.FIELD（属性）
- ElementType.LOCAL_VARIABLE（布局变量）
- ElementType.METHOD（方法）
- ElementType.PACKAGE（包）
- ElementType.PARAMETER（参数）
- ElementType.TYPE（注：任何类型，即上面的的类型都可以修饰）

## @Retention

@Retention是用来修饰注解的注解，使用这个注解，我们可以做到:

- 控制注解是否写入class文件
- 控制class文件中的注解是否在运行时可见


控制很简单，使用使用以下三种策略之一即可。

- RetentionPolicy.SOURCE 表明注解仅存在源码之中，不存在.class文件，更不能运行时可见。常见的注解为@Override, @SuppressWarnings。
- RetentionPolicy.CLASS 这是默认的注解保留策略。这种策略下，注解将存在与.class文件，但是不能被运行时访问。通常这种注解策略用来处于一些字节码级别的操作。
- RetentionPolicy.RUNTIME 这种策略下可以被运行时访问到。通常情况下，我们都会结合反射来做一些事情。

使用示例:

```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotation {
    String   value() default "";
}
```

## @Documented

用来将注释包含在JavaDoc中。

如果定义注解A时，使用了@Documented修饰定义，则在用javadoc命令生成API文档后，所有使用注解A修饰的程序元素，将会包含注解A的说明。

```java
@Documented
public @interface Testable {
}
public class Test {
	@Testable
	public void info() {
	}
}
```


## @Inherited

如果你想让一个类和它的子类都包含某个注解，就可以使用@Inherited来修饰这个注解。

```java
//1
@Inherited
public @interface MyAnnotation {

}

//2
@MyAnnotation
public class MySuperClass { ... }

//3
public class MySubClass extends MySuperClass { ... }
```

上述代码的大致意思是

1.使用@Inherited修饰注解MyAnnotation  
2.使用MyAnnotation注解MySuperClass  
3.实现一个类MySubclass继承自MySuperClass      
通过以上几步，MySubClass也拥有了MyAnnotation注解。





# 自定义注解

在Java中，我们可以创建自己的注解。自定义的注解和类，接口文件一样定义在自己的文件里面。如下

```java
public @interface MyAnnotation {
    String   value();
    String   name();
    int      age();
    String[] newNames();
}
```

上述代码定义了一个叫做MyAnnotation的注解，它有4个元素。再次强调一下，@interface这个关键字用来告诉java编译器这是一个注解。注解元素的定义很类似于接口的方法。这些元素有类型和名称。这些类型可以是:

> 原始数据类型  
> String  
> Class  
> annotation  
> 枚举  
> 以上类型的一维数组 


 
使用自定义的注解：

```java
@MyAnnotation(
    value="123",
    name="Jakob",
    age=37,
    newNames={"Jenkov", "Peterson"}
)
public class MyClass {

}
```

注意，我们需要为所有的注解元素设置值，一个都不能少。

对于注解中的元素，我们可以为其设置默认值，示例：

```java
public @interface MyAnnotation {
    String   value() default "";
    String   name();
    int      age();
    String[] newNames();
}
```

上述代码，我们设置了value元素的默认值为空字符串。当我们在使用时，可以不设置value的值，即让value使用空字符串默认值。使用示例代码

```java
@MyAnnotation(
    name="Jakob",
    age=37,
    newNames={"Jenkov", "Peterson"}
)
public class MyClass {

}
```

# 提取Annotation信息（反射）

待撸<http://www.open-open.com/lib/view/open1423558996951.html>

# Android中的注解


待撸<http://www.jianshu.com/p/1942ad208927>

# 第三方库

待撸<http://sunfusheng.com/java/2016/04/12/reflection-annotation-injection.html>

# 注解原理

待撸<http://www.trinea.cn/android/java-annotation-android-open-source-analysis/>



# 参考链接

<http://droidyue.com/blog/2016/04/24/look-into-java-annotation/>