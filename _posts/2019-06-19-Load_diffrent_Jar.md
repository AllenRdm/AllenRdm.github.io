---
layout: post
title: '加载同一类的不同版本'
subtitle: '使用自定义ClassLoader，加载同一类的不同版本'
date: 2019-06-19
categories: 技术
tags: java ClassLoader 反射
---

有一次在项目里做富文本导出时，发现需要导出所用的poi比项目里用的poi版本要高。由于项目里所用其他导出功能所用的方法，在新版的poi里面又不支持，所以就希望新版的poi只在我所需要的类里面使用。当时没有找到合适方法，后来在便了解到可以用自定义的类加载器来达到效果，于是有了这篇博客的内容。

## 一、准备的类
由于要加载同一类的不同版本，所以要先将两个版本的class导出jar包。分别导出为test1.jar和test2.jar。
两个jar包中的内容分别为：
1、test1.jar
<pre><code class="language-java">
package com;

public class DifferentVersionTest {
	public void test(String name) {
		System.out.println("test version 1 is Run :" + name);
	}
}
</code></pre>

2、test2.jar
<pre><code class="language-java">
package com;

public class DifferentVersionTest {
	public void test() {
		System.out.println("test version 2 is Run");
	}
}
</code></pre>
可以看到两个版本的DifferentVersionTest的test方法是不一样的。

## 二、自定义的MyClassLoader类
MyClassLoader类主要是用于自定义的ClassLoader类，来加载不同的版本。由于，同样的全限定名类由不同的ClassLoader加载，也会被jvm认为是不同的类，所以用自定义的ClassLoader能实现加载同一类的不同版本。代码如下：
<pre><code class="language-java">
 class MyClassLoader extends URLClassLoader {

    public MyClassLoader(String version) {
        super(new URL[] {}, null); // 将 Parent 设置为 null，打破双亲委托的机制
        loadResource(version);
    }

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
    // 测试时可打印看一下
        System.out.println("Class loader: " + name);
        return super.loadClass(name);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            return super.findClass(name);
        } catch(ClassNotFoundException e) {
            return MyClassLoader.class.getClassLoader().loadClass(name);
        }
    }

    private void loadResource(String version) {
        addURL("C:\\Users\\DELL\\Desktop\\" + version);//此处是所加载jar包的目录。
    }

    private void addURL(String path) {
     try {
         super.addURL(new URL("file", null, path));
     } catch (MalformedURLException e) {
         e.printStackTrace();
     } catch (IOException e) {
         e.printStackTrace();
     }
    }
}
</code></pre>

## 三、进行测试
首先我将test2.jar,加入到了project的依赖包里面。这样默认的加载的类是test2.jar中的DifferentVersionTest，
然后我用自定义的MyClassLoader加载test1.jar，以及通过反射得到test1.jar中的DifferentVersionTest以及调用test方法。
代码如下：
<pre><code class="language-java">
package LoadDifferentVersion;

import java.io.IOException;
import java.lang.reflect.Method;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class TestDifferentVersion {

    public static void main(String[] args) throws Exception{
        com.DifferentVersionTest a = new com.DifferentVersionTest();
        //此时运行的是test2.jar中的DifferentVersionTest，因为test2.jar加入了工程的依赖包，而test2.jar没有加入
        a.test();


        /*//设置当前线程的ClassLoader,据说在web容器中的话要用,暂时没有验证
        ClassLoader before = Thread.currentThread().getContextClassLoader();
        Thread.currentThread().setContextClassLoader(loader);*/
        
        MyClassLoader loader = new MyClassLoader("test1.jar");
        //通过自定义的MyClassLoader得到test1.jar中DifferentVersionTest类的Class对象
        Class<?> clazz = loader.loadClass("com.DifferentVersionTest");
        //通过Class对象获取对应的实例对象
        Object instance = clazz.newInstance();
        //通过反射得到test方法
        Method method = clazz.getMethod("test", String.class);
        //通过反射调用test1.jar中DifferentVersionTest类的test方法
        method.invoke(instance, "Linus");
        //Thread.currentThread().setContextClassLoader(before);
    }
}
</code></pre>
运行main方法之后得到如下结果：

test version 2 is Run
Class loader: com.DifferentVersionTest<br>
Class loader: java.lang.Object<br>
Class loader: java.lang.String<br>
Class loader: java.lang.System<br>
Class loader: java.lang.StringBuilder<br>
Class loader: java.io.PrintStream<br>
test version 1 is Run :Linus<br>

Process finished with exit code 0

## 四、结果分析
可以看到最开始调用的是test2.jar中的DifferentVersionTest类的test方法。而后可以看到MyClassLoader加载了test1.jar中的DifferentVersionTest类。在输出的最后，我们可以看到test1.jar中DifferentVersionTest类的test方法被成功调用。从而达到了在一个工程中加载不同版本的类。

