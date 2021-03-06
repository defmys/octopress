---
layout: post
title: "__new__ &amp;&amp; __init__"
date: 2015-12-21 17:48:09 +0800
comments: true
published: true
categories: 
- python
---

既然是第一篇笔记，那就从Python的&#95;&#95;new&#95;&#95;和&#95;&#95;init&#95;&#95;开始吧。

以下内容均针对Python3

先上官方文档：<br />
https://docs.python.org/3/reference/datamodel.html#object.__new__
https://docs.python.org/3/reference/datamodel.html#object.__init__

<h2>__new__</h2>
&#95;&#95;new&#95;&#95;是一个静态方法，被调用之后会返回一个新的object，类型为cls。如果需要自定义一些&#95;&#95;new&#95;&#95;的行为，注意不要忘了调用super().&#95;&#95;new&#95;&#95;(cls[, ...])

``` py3 example01
class TestClass:
    def __new__(cls):
        obj = super(TestClass, cls).__new__(cls)
        return obj
```

<h2>__init__</h2>
当一个对象被创建出来之后，即&#95;&#95;new&#95;&#95;执行之后，&#95;&#95;init&#95;&#95;会被调用。同样，如果基类中有&#95;&#95;init&#95;&#95;，并且派生类显示申明了&#95;&#95;init&#95;&#95;，那么必须调用super().&#95;&#95;init&#95;&#95;(self)



文档明确描述了执行顺序：
先调用&#95;&#95;new&#95;&#95;，生成一个对象。之后调用&#95;&#95;init&#95;&#95;。最后将这个对象返回给调用者。


<h2>进阶</h2>

接下来看一看CPython中这一部分的实现。版本为3.5.1。<br/>
以下代码略去一些步骤，只看我们关心的部分
``` c typeobject.c
static PyObject *
type_call(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    PyObject *obj;

    // 略去一些检查
    // ...

    // tp_new    
    // https://docs.python.org/3.5/c-api/typeobj.html#c.PyTypeObject.tp_new
    obj = type->tp_new(type, args, kids);    // tp_new完成了一些初始化以及内存分配的工作
    if (obj != NULL) {
        
        // 略去一些检查
        // ...

        type = Py_TYPE(obj);
        if (type->tp_init != NULL) {
            // tp_init    
            // https://docs.python.org/3.5/c-api/typeobj.html#c.PyTypeObject.tp_init
            int res = type->tp_init(obj, args, kwds);    
            
            // 略去一些检查
            // ...
        }
    }
    return obj;
}
```


&#95;&#95;new&#95;&#95;和&#95;&#95;init&#95;&#95;都是在对象被创建出来时调用，都可以用来给对象做初始化。那么何时应该用&#95;&#95;new&#95;&#95;，何时应该用&#95;&#95;init&#95;&#95;呢？<br/>
以下是文档中的描述：
{% blockquote docs.python.org https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new Type Objects %}
The tp_new function should call subtype->tp_alloc(subtype, nitems) to allocate space for the object, and then do only as much further initialization as is absolutely necessary. Initialization that can safely be ignored or repeated should be placed in the tp_init handler. A good rule of thumb is that for immutable types, all initialization should take place in tp_new, while for mutable types, most initialization should be deferred to tp_init.
{% endblockquote  %}
可以看到，tp_new通常只做一些必须的初始化工作，而一些可以被安全的忽略或者可以被重复调用的初始化过程应该放在tp_init中。对于immutable tpyes(比如 int, tuple, string等)，初始化工作应该放在tp_new中；对于mutable types，最好放在tp_init中。


在上面的代码中省略了一些检查，其中有一部分比较有意思：<strong>在某些条件下，tp_init是不会被执行的。</strong>暂时不看这部分代码，我们直接从文档入手。<br/>
{% blockquote docs.python.org https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_init Type Objects %}
If the tp_new function returns an instance of some other type that is not a subtype of the original type, no tp_init function is called.
{% endblockquote  %}
当tp_new返回了一个不是当前类型的对象，tp_init不会被调用。<br/>
下面我们做个简单的测试。
``` py3 test01
class ClassWithInit:
    def __new__(cls):
        print('ClassWithInit.__new__')
        # 返回一个同类型的对象，之后应该会调用__init__
        return super(ClassWithInit, cls).__new__(cls)    

    def __init__(self):
        print('ClassWithInit.__init__')


class ClassWithoutInit:
    def __new__(cls):
        print('ClassWithoutInit.__new__')
        # 返回了一个int，与当前类型不同，也不是派生类的对象，__init__应该不会被调用
        return int(0)    

    def __init__(self):
        print('ClassWithoutInit.__init__')


if __name__=='__main__':
    ClassWithInit()
    ClassWithoutInit()
```

输出结果和预期相同：
``` bash output:test01
ClassWithInit.__new__
ClassWithInit.__init__
ClassWithoutInit.__new__
```

