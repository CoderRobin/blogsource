title: Java Annotation基础
date: 2015-03-02 22:12:37
categories: java
tags: Annotation
---
#简介
在Android开发中，为了提高开发效率，各个注解框架（如Butter Knife，Dagger 等）越来越流行，为了方便理解和使用这些注解框架，需要对Java Annotation有个初步的了解。
<!--more-->
#基础入门
类、方法、变量、参数、包都可以被注解.
##作用
1.为编译器提供辅助信息 — Annotations可以为编译器提供而外信息，以便于检测错误，抑制警告等.
2.编译源代码时进行而外操作 — 软件工具可以通过处理Annotation信息来生成原代码，xml文件等等.
3.运行时处理 — 有一些annotation甚至可以在程序运行时被检测，使用.
如java自带的标准 Annotation
Override：标记此为重载方法
Deprecated:标记该方法已弃用
SuppressWarnings:标记忽略某项 Warning

##元注解
元注解的作用就是负责注解其他注解。Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。Java5.0定义的元注解：

###@Target：
@Target说明了Annotation所修饰的对象范围：
作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）
取值(ElementType)有：
1.CONSTRUCTOR:用于描述构造器
2.FIELD:用于描述域
3.LOCAL_VARIABLE:用于描述局部变量
4.METHOD:用于描述方法
5.PACKAGE:用于描述包
6.PARAMETER:用于描述参数
7.TYPE:用于描述类、接口(包括注解类型) 或enum声明
###@Retention：
@Retention定义了该Annotation的作用时间
作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期
取值（RetentionPoicy）有：
1.SOURCE:在源文件中有效
2.CLASS:在class文件中有效
3.RUNTIME:在运行时有效
###@Documented:
@Documented标记可以被文档化
###@Inherited：
@Inherited它的作用是控制注释是否会影响到子类。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。
#自定义注解（运行时注解使用示例）
该使用方法本质为在运行时通过反射获取注解的相关信息，并进行后续操作。
以下为利用反射获取注解信息的例子。
###CustomizedAnnatation类
```
package com.coderrobin;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD) //表明该注解用于注解域（成员变量）
@Retention(RetentionPolicy.RUNTIME) //表明该注解为运行时注解
public @interface CustomizedAnnatation
	public String name() default "coderrobin";
}

```
###测试类
```
package com.coderrobin;

import java.lang.reflect.Field;

public class TestClass {
	@CustomizedAnnatation(name="coderrobin")
	private String anotationName;
	/**
	 * @param args
	 */
	public static void main(String[] args) {
		 Class testClass=TestClass.class;
		 Field[] fields=testClass.getDeclaredFields();
		 for(Field field:fields){
			 field.setAccessible(true);
		 if(field.isAnnotationPresent(CustomizedAnnatation.class)){
			 CustomizedAnnatation annotation=(CustomizedAnnatation)field.getAnnotation(CustomizedAnnatation.class);
			 System.out.println(annotation);
			 System.out.println(annotation.name()); 
			 try {
				TestClass textClass=TestClass.class.newInstance();
				textClass.anotationName=annotation.name();
				 System.out.println(textClass.anotationName); 
			} catch (InstantiationException e) {
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			}
			}
		 }
	}

}

```
#自定义注解（编译时使用）
以下代码需配置环境才能看到结果，可参考鸿洋的http://blog.csdn.net/
lmj623565791/article/details/43452969
以下为在编译时产生一个txt文件的例子
```
package com.coderrobin;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface CustomizedAnnatationSource {
	public String name() default "coderrobin";
}


@SupportedAnnotationTypes({"com.coderrobin.CustomizedAnnatationSource"}) //注解处理器
public class CustomizedProcesser extends AbstractProcessor {
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
       	File file = new File("/home/coderrobin/coderrobin.txt");  
  FileWriter fw;
fw = new FileWriter(file);
        for (TypeElement te : annotations) {
            for (Element e : env.getElementsAnnotatedWith(te)) {
				try {
                   fw.append( e.toString());
                   fw.flush();
				} catch (IOException e1) {
					// TODO Auto-generated catch block
					e1.printStackTrace();
				}  

            }
        }
  fw.close();
        return true;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}


public class AnnatationSourceTest {
	@CustomizedAnnatationSource
	public String test;
	
}


```
#总结
利用编译时注解可以产生需要的文件，包括源码，以此提高生产效率。



