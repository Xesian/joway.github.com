---
layout: post
title: C++如何禁止全局对象被析构
tags: [C/C++]
keywords: 全局对象析构, 禁止全局对象被析构, 如何禁止全局对象被析构, C++全局对象
description: C++如何禁止全局对象被析构
comments: true
share: true
---
全局对象无论是在C++中，还是在C(C里面的全局变量)里面 ，都是比较难管理而且不提倡过多的使用。一方面它生命周期不好控制 ，另一方面多线程中共享需要额外的消耗。另外，全局变量或者对象的初始化顺序不固定，在进程结束的时候全局对象的析构函数会被隐式调用，调用顺序也不固定。如果全局对象之间还有引用关系，会使得程序变得更加糟糕。这里主要讲一些技巧如何防止全局对象被析构。

<!--more-->

##前言   
一般情况下，我们都要求所有new出来的对象，在程序结束的时候，都要将其析构掉。但是在某些情况下，我们希望一些很重要的全局对象在程序运行的整个过程中都存在，而且生命周期跟进程相同。在这种情况下，其实是要防止全局对象被析构，即便是进程结束也不要调用其析构函数。例如：WebKit中JavaScriptCore的VM对象。WebKit的bindings里面，VM对象就是一个跟进程生命周期相同的一个全局变量。任何时候都不允许它被析构掉，如果被析构掉，HTML就无法执行JavaScript了。


##全局指针  
如果说到全局对象 ，可能大部分人第一时间想到的就是new一个对象，将其指针保存在全局变量里面。这种做法其实比较危险。new出来的指针放在全局变量里面，代码的其他任何地方都可以获取该指针，如果不小心被其他地方delete掉了呢？另一方面，也不能满足程序设计要隐藏实现细节的要求。

##全局对象
采用直接保存全局对象的方案，可以避免上面提到的被其他地方delete掉。但是有一个缺点，进程结束的时候，该对象会隐式的被调用，也就是还有可能被析构掉。对于某些比较重要的对象，我们希望任何时候都不被析构掉，例如前面说的VM对象。

##静态局部变量
先来看一段代码：
{% highlight C++ linenos %}
class A {
public:
    A() {};
private:
    A(const A&);
    A(A&);
};

A& getInstance()
{
    static A* a = new A();
    A& obj = *a;
    return obj;
}
{% endhighlight %}
这段代码弥补了上面提到的全局指针和静态全局变量的缺点。避免了指针的直接传递，保证了全局对象的单实例。由于将拷贝构造函数声明为private，使得所有需要使用A对象的地方，都需要声明为引用，这样，即便是进程结束，其构造函数也不会被调用。   

下面这段代码摘自WebKit中WTF的StdLibExtras.h
{% highlight C++ linenos %}
// Use these to declare and define a static local variable (static T;) 
// so that it is leaked so that its destructors are not called at exit.
// Using this macro also allows workarounds a compiler bug present in 
// Apple's version of GCC 4.0.1.
#ifndef DEFINE_STATIC_LOCAL
#if COMPILER(GCC) && defined(__APPLE_CC__)  \
	&& __GNUC__ == 4 \
	&& __GNUC_MINOR__ == 0 \
	&& __GNUC_PATCHLEVEL__ == 1
#define DEFINE_STATIC_LOCAL(type, name, arguments) \
    static type* name##Ptr = new type arguments; \
    type& name = *name##Ptr
#else
#define DEFINE_STATIC_LOCAL(type, name, arguments) \
    static type& name = *new type arguments
#endif
#endif
{% endhighlight %}
WebKit中，有很多对象都是使用DEFINE\_STATIC\_LOCAL来定义的：
>>Use these to declare and define a static local variable (static T;) so that it is leaked so that its destructors are not called at exit.Using this macro also allows workarounds a compiler bug present in Apple's version of GCC 4.0.1.

##全局对象何时被析构？
很多C++编程规范中，全局对象都是禁止使用的，例如Google的C++ code style的[Static and Global Variables](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml?showone=Static_and_Global_Variables#Static_and_Global_Variables)。原因是因为类的对象作为全局对象的时候，全局对象的析构顺序是由全局对象的构造顺序决定的，但是全局对象的构造顺序又是不确定的，不同的平台，不同的系统，甚至是不同的Build之间，全局对象的构造顺序都是不一样的。但是有一点是可以确定的，全局对象的析构一定是出现在程序的main()结束之后或者exit()函数调用之后。除了上面提到的方法可以禁止全局对象被析构以外，还有一个方法即使quick\_exit。程序结束的时候调用quick\_exit来替代exit(),这样，全局对象的析构函数就不会被调用了。


##析构全局对象有什么影响呢？
很多人看了这篇文章，可能都会问：反正进程都结束了，析构或者不析构全局对象都无所谓。其实是有所谓的。特别是在有多个全局对象或者全局对象被多线程引用的场景。问题是，由于全局对象的析构顺序不确定，如果全局对象的析构函数中又对其他全局对象有引用，或者，一个全局对象已经析构掉了，但是这个全局对象还在背某个线程引用，这种情况，在程序退出的时候会出现偶现(由于全局对象析构顺序不确定，所以不是必现的)的crash的bug，这种问题一旦出现，比较难定位。所以，如果要一定要使用全局变量，那么这个变量必须是遵循[POD(Plain Old Data)](http://en.wikipedia.org/wiki/Plain_old_data_structure)。POD要求数据结构或者类不允许自定义自己的析构函数。

