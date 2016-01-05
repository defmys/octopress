---
layout: post
title: "Metaclass"
date: 2016-01-05 10:36:01 +0800
published: true
comments: true
categories: 
- python
---
<p>
在Python中，任何东西都是对象。函数是对象，类也是对象，类的类也是对象。
``` py3 Example0.py
def foo():
    pass

type(foo)  
type(type(foo))
type(type(type(foo)))
```

输出：
``` sh
<class 'function'>
<class 'type'>
<class 'type'>
```

可以看到，foo是function的对象，function是type的对象，type是type的对象。
</p>
<h2>type</h2>
type是python中的一个builtin类，并非关键字(<a href="https://docs.python.org/3/reference/lexical_analysis.html#keywords">Python Keywords</a>)。<br />
有两种方式可以调用type，分别产生两种效果:

    * 当只有一个参数时，也就是上面例子中的用法，type返回参数中对象的类
    * 当有三个参数时，type生成一个新的类。

我们只看第一种用法。以下是python3.5.1中的C实现：
``` c typeobject.c
static PyObject *
type_new(PyTypeObject *metatype, PyObject *args, PyObject *kwds)
{
    // 省略部分代码
    /* Special case: type(x) should return x->ob_type */
    {
        const Py_ssize_t nargs = PyTuple_GET_SIZE(args);
        const Py_ssize_t nkwds = kwds == NULL ? 0 : PyDict_Size(kwds);

        if (PyType_CheckExact(metatype) && nargs == 1 && nkwds == 0) {
            PyObject *x = PyTuple_GET_ITEM(args, 0);
            Py_INCREF(Py_TYPE(x));
            return (PyObject *) Py_TYPE(x);
        }

        /* SF bug 475327 -- if that didn't trigger, we need 3
           arguments. but PyArg_ParseTupleAndKeywords below may give
           a msg saying type() needs exactly 3. */
        if (nargs + nkwds != 3) {
            PyErr_SetString(PyExc_TypeError,
                            "type() takes 1 or 3 arguments");
            return NULL;
        }
    }

    // 省略部分代码
}
```
<p>可以看到这里有一个特殊处理：当只args中只有一个参数时，type(x)会直接返回这一个参数的类。</p>

<h2>Metaclass</h2>
<p>通过type，我们可以知道一个对象的类究竟是什么。从第一个例子中可以看出，函数是function类的对象，类是type类的对象，type类自己也是type类的对象。<strong>类的类就是metaclass</strong>。</p>
<p>如果我们在创建一个类时，没有特别指明一个metaclass，那么这个类就会使用type类作为自己的metaclass。</p>
<p>类决定了对象的行为，同理，作为类的类，metaclass决定了一个类在生成对象时的行为。</p>

<p>
metaclass可以是任何可被调用的类型，类或者函数都可以。<br />
Python3为metaclass引入了一个新的函数:<strong>&#95;&#95;prepare&#95;&#95;</strong>，这个函数会在对象创建前被调用并返回一个字典。这个字典不一定是内置的dict，只要实现了部分必须的接口就可以。这个字典中存储了对象的成员。由于可以是自己实现的dict，我们可以对其进行各种操作。
</p>
<p>所以，通过&#95;&#95;prepare&#95;&#95;, &#95;&#95;new&#95;&#95;, &#95;&#95;init&#95;&#95;等函数，我们可以控制生成对象时的一些行为。</p>

以下是PEP3115中的例子，最后稍做了一点改动，用来输出结果：
``` python3 PEP3115_Example1.py
# Here's a simple example of a metaclass which creates a list of
# the names of all class members, in the order that they were
# declared:

# The custom dictionary
class member_table(dict):
    def __init__(self):
        self.member_names = []

    def __setitem__(self, key, value):
        # if the key is not already defined, add to the
        # list of keys.
        if key not in self:
            self.member_names.append(key)

        # Call superclass
        dict.__setitem__(self, key, value)


# The metaclass
class OrderedClass(type):

    # The prepare function
    @classmethod
    def __prepare__(metacls, name, bases): # No keywords in this case
        return member_table()

    # The metaclass invocation
    def __new__(cls, name, bases, classdict):
        # Note that we replace the classdict with a regular
        # dict before passing it to the superclass, so that we
        # don't continue to record member names after the class
        # has been created.
        result = type.__new__(cls, name, bases, dict(classdict))
        result.member_names = classdict.member_names
        return result

class MyClass(metaclass=OrderedClass):
    # method1 goes in array element 0
    def method1(self):
        pass

    # method2 goes in array element 1
    def method2(self):
        pass

if __name__=='__main__':
    print(MyClass().member_names)
```

输出：
``` sh
['__module__', '__qualname__', 'method1', 'method2']
```


该例子中：

    1. 创建了一个member_table，继承自dict并重载了__setitem__函数，按顺序记录所有key。
    2. 声明了一个新的metaclass -- OrderedClass。
        2.1 __prepare__函数返回一个member_table对象
        2.2 __new__函数中将member_table里记录的key复制给自己的一个成员变量member_names。
    3. 申明了一个metaclass为OrderedClass的类，声明两个成员函数。这两个成员函数的名称会被存储在member_names中。



当一个对象被创建时，以下工作会被按顺序执行
{% blockquote docs.python.org https://docs.python.org/3/reference/datamodel.html#customizing-class-creation Customizing class creation %}
When a class definition is executed, the following steps occur:

    * the appropriate metaclass is determined
    * the class namespace is prepared
    * the class body is executed
    * the class object is created
{% endblockquote %}

<p>以下是对应的代码：
``` py3 Lib/types.py
# Provide a PEP 3115 compliant mechanism for class creation
def new_class(name, bases=(), kwds=None, exec_body=None):
    """Create a class object dynamically using the appropriate metaclass."""
    meta, ns, kwds = prepare_class(name, bases, kwds)
    if exec_body is not None:
        exec_body(ns)
    return meta(name, bases, ns, **kwds)

def prepare_class(name, bases=(), kwds=None):
    """Call the __prepare__ method of the appropriate metaclass.

    Returns (metaclass, namespace, kwds) as a 3-tuple

    *metaclass* is the appropriate metaclass
    *namespace* is the prepared class namespace
    *kwds* is an updated copy of the passed in kwds argument with any
    'metaclass' entry removed. If no kwds argument is passed in, this will
    be an empty dict.
    """
    if kwds is None:
        kwds = {}
    else:
        kwds = dict(kwds) # Don't alter the provided mapping
    if 'metaclass' in kwds:
        meta = kwds.pop('metaclass')
    else:
        if bases:
            meta = type(bases[0])
        else:
            meta = type
    if isinstance(meta, type):
        # when meta is a type, we first determine the most-derived metaclass
        # instead of invoking the initial candidate directly
        meta = _calculate_meta(meta, bases)
    if hasattr(meta, '__prepare__'):
        ns = meta.__prepare__(name, bases, **kwds)
    else:
        ns = {}
    return meta, ns, kwds
```

以上代码解释了metaclass中的&#95;&#95;prepare&#95;&#95;在何时被调用以及完成了哪些工作。</p>


<p>再简单看一下metaclass在type_new函数中如何工作。以下代码只保留了我们关心的部分。
``` c typeobject.c
static PyObject *
type_new(PyTypeObject *metatype, PyObject *args, PyObject *kwds)
{
    // ..........................

    /* Determine the proper metatype to deal with this: */
    winner = _PyType_CalculateMetaclass(metatype, bases);
    if (winner == NULL) {
        return NULL;
    }

    if (winner != metatype) {
        if (winner->tp_new != type_new) /* Pass it to the winner */
            return winner->tp_new(winner, args, kwds);
        metatype = winner;
    }

    // ............................

     /* Allocate the type object */
    type = (PyTypeObject *)metatype->tp_alloc(metatype, nslots);
    if (type == NULL)
        goto error;

    // ..............................

    return (PyObject *)type;
}
```

首先还是选取一个合适的metaclass，如果选出的metaclass不是当前参数中的metatype，那么执行选出的metatype->tp_new。然后为类申请空间、进行初始化工作。最后返回该类。</p>


<h2>实践</h2>
<p>最后，我们用metaclass来实现一个简单的singleton。
``` py3 Example2.py
class Singleton(type):
    instance = None
    def __call__(cls, *args, **kwargs):
        if not cls.instance:
            cls.instance = super().__call__(*args, **kwargs)
        return cls.instance

    def __repr__(cls):
        return cls.__name__


class ASingleton(metaclass=Singleton):
    pass


class BSingleton(metaclass=Singleton):
    pass


if __name__ == '__main__':
    a1 = ASingleton()
    a2 = ASingleton()
    print('a1 is a2? %s' % (a1 is a2))
    print(a1.__class__)
    print(a2.__class__)

    print('===============')
    b = BSingleton()
    print('a1 is b? %s' % (a1 is b))
    print(b.__class__)
```

输出：
``` sh
a1 is a2? True
ASingleton
ASingleton
===============
a1.instance is ASingleton.instance?
True
===============
a1 is b? False
BSingleton
```

</p>
<p>
在这个例子中，我们创建了一个名为Singleton的metaclass。Singleton带有一个静态成员变量instance。在调用ASingleton或BSingleton创建对象时，会先检查instance是否为空。<br />需要注意的是，在第一次调用时，我们访问的是metaclass的静态成员变量instance。当对象被创建出来之后，这个对象会被赋值给各自class的instance（注意__call__的第一个参数 cls）。之后再通过ASingleton或BSingleton创建对象时，访问的是它们各自的类的instance，而不是metaclass的instance。<br />另外，由于使用的是类的instance，即相当于该class的静态成员变量，而不是生成出的对象的成员变量，所以相同的类的object所访问的都是同一个instance。
</p>


<h2></h2>
<h4>参考：</h4>
<ul>
<li><a href="https://www.python.org/dev/peps/pep-3115/">PEP 3115 -- Metaclasses in Python 3000</a></li>
<li><a href="https://docs.python.org/3/reference/datamodel.html#customizing-class-creation">Data model - Python 3.5.1 documentation</a></li>
<li><a href="http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python">oop - What is a metaclass in Python? - Stack Overflow</a></li>
<li><a href="http://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/">Understanding Python metaclasses | ionel's codelog</a></li>
<li><a href="https://en.wikibooks.org/wiki/Python_Programming/Metaclasses">Python Programming/Metaclasses - Wikibooks, open books for an open world</a></li>
<li><a href="http://python-3-patterns-idioms-test.readthedocs.org/en/latest/Metaprogramming.html">Metaprogramming - Python 3 Patterns, Recipes and Idioms</a></li>
</ul>


