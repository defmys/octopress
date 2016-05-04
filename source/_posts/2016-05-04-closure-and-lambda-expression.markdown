---
layout: post
title: "闭包及lambda表达式"
date: 2016-05-04 15:45:36 +0800
comments: true
categories: 
- python
- c++
---

<p>
对于闭包这个概念，以前有过一点点了解，但实际工作中并没有真正的去用，也没有仔细的去研究。此篇笔记就献给闭包了，好好整理一下概念与思路。
</p>

<h2>概念</h2>
<p>
先看一下维基百科上的定义：
{% blockquote Wikipeida https://en.wikipedia.org/wiki/Closure_(computer_programming) Closure (computer programming) %}
A closure is a record storing a function together with an environment: a mapping associating each free variable of the function (variables that are used locally, but defined in an enclosing scope) with the value or storage location to which the name was bound when the closure was created.
{% endblockquote %}
</p>
<p>
简单的说，闭包可以看做是一个函数加上与其相关的封闭环境所组成的一个实例。例如：
``` py3 example_00
def closure(x):
    def f(y):
        return x + y
    return f

closure1 = closure(10)
closure2 = closure(20)

print(closure1(1))
print(closure2(1))
```
</p>

<p>
输出：
``` sh output
11
21
```
</p>

<p>
例子中的closure1与closure2就是两个闭包。
</p>
<p>
首先看closure函数，它接受一个参数x。当调用closure函数时，x被赋值，同时创建并返回函数对象f。x的作用域虽然为closure函数本身，但是由于f中还有x的引用，所以x并不会被释放，而仍然留在内存中。此时， 我们可以认为，被返回的不仅仅是f，<strong>还包括一个的封闭环境</strong>，x就存在于这个环境中。
</p>
<p>
所以对于closure1和closure2来说，他们不仅仅等于函数f，还包含了生成这两个对象时所创建出来的x对象。
</p>

<h2>用法</h2>
<h3>Python</h3>
<p>
前面的例子是用Python实现的一个闭包，不过还可以做一点点改进。例子中的f是在closure函数中定义的，我们可以用匿名函数来改写它。
``` py3 example_01
def closure(x):
    return lambda y: x + y

closure1 = closure(10)
closure2 = closure(20)

print(closure1(1))
print(closure2(1))
```
</p>
<p>
输出：
``` sh output
11
21
```
</p>
<p>
这样，我们就用python的lambda表达式构建出了闭包。
</p>

<h2>C++</h2>
<p>
C++11引入了lambda，可以与python一样方便的利用lambda创建闭包。
</p>
<p>
我们先看一下C++的lambda语法：<br />
<span style="font-size:12pt;">
[ capture-list ] ( params ) mutable(optional) constexpr(optional)(c++17) exception attribute -> ret {body }
<span>
</p>
<p>
其中：
<table>
<tr>
<td style="width: 20%;font-weight:bold;font-style:italic;">capture-list</td>
<td>捕获变量的列表。被捕获的变量可以在lambda表达式函数体中使用。</td>
</tr>
<tr>
<td style="width: 20%;font-weight:bold;font-style:italic;">params</td>
<td>lambda表达式的参数列表</td>
</tr>
<tr>
<td style="width: 20%;font-weight:bold;font-style:italic;">mutable</td>
<td>捕获变量时如果采用的是拷贝的方式，那么这些变量默认带const属性。加上mutable之后，可以去掉其const属性。</td>
</tr>
<tr>
<td style="width: 20%;font-weight:bold;font-style:italic;">ret</td>
<td>返回值类型</td>
</tr>
</table>
</p>
<p>
下面我们一步一步来用lambda表达式创建闭包。
</p>
<h4>1. 生成一个空的匿名函数</h4>
<p>
``` c++
[](){};
```
</p>
<p>
这个lambda表达式的capture-list是空的，也就是说它不能使用任何外层作用域的变量。同时，其参数列表和函数体也是空的。<br />
对于不含有参数，即参数列表为空的lambda表达式，可以省略():
``` c++
[]{};
```
</p>
<h4>2. 捕获变量</h4>
<p>
函数只是闭包的一部分，另一部分是封闭的环境。我们已经生成了一个空的匿名函数，但是并没有看到这个环境。下面，我们将一些变量放入这个封闭环境的中。假设x是一个int型的变量：
``` c++
[x] { std::cout<<x<<std::endl; };
```
</p>
<p>
此时，我们就可以在函数体中使用x了。下面的例子说明x存在于封闭的环境中：
``` c++ example_02
#include <iostream>
#include <functional>

std::function<void(void)> closure(int x)
{
    return [x] { std::cout<<x<<std::endl; };
}

int main()
{
    auto closure1 = closure(10);
    auto closure2 = closure(20);

    closure1();
    closure2();

    return 0;
}
```
</p>
<p>
和开头看到的python的例子类似，我们利用closure函数创建闭包。对于closure1和closure2两个闭包来说，他们不仅仅是由closure函数返回的匿名函数，同时还包含了x变量的拷贝（后面会解释为什么是拷贝）。
</p>
<p>
我们修改一下这个例子，使其实现和python例子完全一样的功能：
``` c++ example_03
#include <iostream>
#include <functional>

std::function<int(int)> closure(int x)
{
    return [x](int y)->int { return x + y; };
}

int main()
{
    auto closure1 = closure(10);
    auto closure2 = closure(20);

    std::cout << closure1(1) << std::endl;
    std::cout << closure2(1) << std::endl;

    return 0;
}
```
</p>
<p>
首先，lambda表达式被修改为：
``` c++
[x](int y)->int { return x + y; }
```
使其接受一个int型的参数y，计算并返回x+y。 -> int 表示返回值类型为int型。
</p>
<p>
之后，还需要修改一下closure函数的返回值类型。由于现在lambda表达式多了一个int型的参数，同时会返回int型的返回值，所以返回类型变为了
``` c++
std::function<int(int)>
```
</p>
<p>
严格的说，这并不是匿名函数的真实类型，不过我们写的lambda表达式可以被转换为<strong>std::function&lt;int(int)&gt;</strong>。
</p>
<p>
用python构建的闭包中变量x的生存周期是自动管理的。虽然外层的函数closure已经结束了，但是这个变量并不会被释放，直到闭包也被释放、且x的引用计数为0时才会被释放。<br />
而在c++中，问题就比较复杂了。前面的例子中，x变量是按值传递的，即x是原变量的拷贝。除了这种按值传递的方式之外，还有其他的方式可以捕获变量。
<table>
<tr>
<td style="width: 15%;">[]</td>
<td>- 什么变量都不捕获。</td>
</tr>
<tr>
<td style="width: 15%;">[=]</td>
<td>- 按值捕获在lambda表达式所在函数的函数体中提及的全部自动储存持续性变量</td>
</tr>
<tr>
<td style="width: 15%;">[&]</td>
<td>- 按引用捕获在lambda表达式所在函数的函数体中提及的全部自动储存持续性变量</td>
</tr>
<tr>
<td style="width: 15%;">[a,&b]</td>
<td>- 按值捕获a，按引用捕获b</td>
</tr>
<tr>
<td style="width: 15%;">[this]</td>
<td>- 按值捕获this指针</td>
</tr>
</table>
</p>
<p>
当我们使用[]时，即不捕获任何变量时，这个闭包的环境内是没有变量的。此时lambda表达式返回的是一个函数指针。而其他情况下，返回的是一个ClosureType对象，即闭包对象。
</p>
分别来看几种捕获变量的情况：
<h5>按值传递</h5>
<p>
在闭包对象中保存的变量是一份拷贝，闭包对象被销毁时这些对象也会被销毁。
</p>

<h5>按引用传递</h5>
<p>
闭包中保存的变量是原变量的引用。对闭包中这些变量进行修改会影响原始的变量，反之亦然。同时，原变量的生存周期也会影响到闭包中的变量，这里就有陷阱了。例如，我们修改一下例子中的lambda表达式，按引用捕获x
``` c++
std::function<int(int)> closure(int x)
{
    return [&x](int y)->int { return x + y; };
}
```
这时候闭包中保存的x是引用，但是闭包对象被创建出来之后，closure函数的参数x会被释放，闭包中保存的引用也随之失效。这时就产生了dangling reference，并导致程序运行出现未定义的行为。
<p>
{% blockquote cppreference.com http://en.cppreference.com/w/cpp/language/lambda Dangling references %}
<strong>Dangling references</strong><br />
If an entity is captured by reference, implicitly or explicitly, and the function call operator of the closure object is invoked after the entity's lifetime has ended, undefined behavior occurs. The C++ closures do not extend the lifetimes of the captured references.
Same applies to the lifetime of the object pointed to by the captured this pointer.
{% endblockquote %}
</p>
所以，在使用闭包的过程中，需要仔细考虑变量的生存周期，避免引用和指针指向失效的地址。
</p>


<h2>扩展</h2>
<p>
C++的例子中，申明一个closure函数比较麻烦，同时，每次修改closure函数返回的lambda表达式时，都需要考虑是否需要修改返回值类型。我们的例子中，一开始返回值的类型是std::function<void(void)>，到最后修改为了std::function<int(int)>。在C++11中，auto对于函数返回值类型的推演并不是很方便，但是从C++14之后，我们可以直接写成
``` c++
auto closure(int x)
{
    return [x](int y)->int { return x + y; };
}
```
这样不论怎么修改lambda表达式，我们都不用去关心返回值的类型了。
</p>

<h2></h2>
<h4>参考</h4>
<ul>
<li><a href="https://en.wikipedia.org/wiki/Closure_(computer_programming)">Closure (computer programming) - Wikipedia</a></li>
<li><a href="http://en.cppreference.com/w/cpp/language/lambda">Lambda functions (since C++11) - cppreference.com</a></li>
</ul>

