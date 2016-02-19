---
layout: post
title: "Coding Style"
date: 2016-02-18 15:32:45 +0800
comments: true
categories: 
- python
---

<p>
最近在整理前段时间写的代码。之前一直使用IDE进行检查，我使用的是PyCharm，能够使代码基本满足PEP8的要求。但还是有一些例外。利用其他工具检查的时候就能发现一些问题。我一直觉得代码必须要遵循一定的规则，以增强可读性。曾经看到过一句话，大概是说，代码是给人看的，至于如何让机器理解，那是编译器的工作。一般情况下公司对代码都会有一定的要求，不过是否严格执行就不一定了。在没有严格要求的情况下，纯靠自觉。。<br />
<ul>
    <li>C++可以参考<a href="https://google.github.io/styleguide/cppguide.html">Google C++ Style Guide</a></li>
    <li>Python主要参考<a href="https://www.python.org/dev/peps/pep-0008/">PEP8</a> </li>
</ul>
</p>

<h2>PEP8</h2>
<p>
PEP8涵盖了命名规则、缩进、行长度等规则，也有不少工具可以自动进行检查。
</p>
<p>
其中比较有争议的是单行的最大长度。PEP8中规定一行代码的最大长度为79个字符，如果项目组内可以达成共识，可将长度扩展到99个字符。有一些编辑器默认会将行长度限制到80字符，超出限制之后会自动换行。换行之后代码会变得非常乱，影响阅读。将行长度限制到79个字符而不是80个字符是因为部分编辑器会显示出行尾的换行符，当一行有80个字符时，加上最后这个换行符，就会变成81个字符。这种情况下，编辑器可能会自动换行。另外，我也搜到了一些其他的文章，讨论“80个字符”这个限制的各种原因，包括终端显示、人阅读时的行为习惯等等。
</p>
<p>
将代码限制到79个字符是一个比较痛苦过程。PyCharm默认限制为120个字符，我也一直是按120字符去进行换行。重新限制了长度之后，有几百处代码需要修改。但修改完之后发现还是有很多好处的。比如可以同时开两个vim放在屏幕左右两边，阅读代码时不需要左右滚动；PyCharm中也可以左右开两个标签页；阅读时眼睛基本集中在一片区域等等。目前项目中还没有遇到无法修改到79个字符以内的代码。
</p>
<p>
至于其他规则，目前感觉没什么好纠结的。
</p>

<h2>Flake8</h2>
<p>
Flake8是一个执行python代码检查的工具集合。目前我使用Flake8来对项目代码进行检查。除PEP8之外，Flake8还会执行一些额外的检查。可以通过命令行参数调整flake8的行为，也可以将参数写入配置文件中。项目的setup.cfg中如果存在[flake8]的配置，flake8会自动读取这些配置。例如我们的setup.cfg文件中：
``` ini setup.cfg
[flake8]
exclude = some_dir/*
max-complexity = 10
```
检查时会跳过some_dir，同时开启复杂度检查（默认是关闭的），允许的最大复杂度为10。
</p>
<p>
可以把flake8做成hook，提交代码之后自动进行检查。我们的项目一直使用jenkins做一些自动化的工作，任何一个任务执行失败之后会自动发邮件给相关人员，也能看到具体执行的命令和输出，非常方便。所以我直接把检查的步骤交给jenkins去做，每半个小时检查一次svn是否有更新，如果有更新，执行一次检查，如果出现错误，自动给这段时间提交过代码的人发邮件。
</p>

<h2>McCabe复杂度</h2>
<p>
Flake8中集成了<a href="https://en.wikipedia.org/wiki/Cyclomatic_complexity">McCabe复杂度（循环复杂度）</a>检查，默认情况下是关闭的。循环复杂度是一段代码中线性独立路径的数量。例如，一段代码中存在一个if语句，那么就存在true和false两种情况，对应就有两条路径。<br />
循环复杂度M定义为：<br /><br />
<strong>M = E - N + 2P</strong><br />
{% img right /images/Control_flow_graph_of_function_with_loop_and_an_if_statement_without_loop_back.svg 192 240 complexity example%}
<ul>
其中：
<li>E = 图中边的数量</li>
<li>N = 图中顶点的数量</li>
<li>P = 图中连通分支的数量</li>
</ul>
<br />

借用wikipedia中的一个例子：<br>

右图中有9条边，8个顶点，1个连通分支。E=9，N=8，P=1，所以循环复杂度M = 9 - 8 + 2*1 = 3
</p>

<p>
我们项目的代码使用MVC模式。View纯粹是前端的工作，对于只提供API的后端来说，只有M与C。Model与Model之间基本无耦合，逻辑比较清晰，在检查复杂度的时候基本没有问题，偶尔有一两处复杂度达到了11或12，简单修改之后就能降低到10以下。问题在于Controller。Controller将不同的Model在逻辑上联系在一起，对于逻辑上稍复杂一些的Controller，代码很容易写的很长，分支也很多。检查之后发现，复杂度高的基本都是Controller，最大的能达到26。写代码的时候没什么感觉，但是回过头再看这些代码的时候，发现真的是又长又难理解。经过一番重构之后，最终把所有代码的复杂度都降到了10以下。这样的重构带来的好处是使代码变得清晰易懂。我一直希望自己写的代码能满足self-documenting code的要求，简单易懂好维护，无需多余的注释。通过这次重构，我发现在降低复杂度的同时，代码能向着self-documenting的方向更进一步。降低复杂度时我将大段的、逻辑上可分离的分支抽取成函数，函数名即可表名这段代码的作用。逻辑主干上看到的只是这个函数名，而实现细节被封装了起来。阅读时看到函数名就等于是看到了一行注释，比直接阅读实现细节要清晰明了。
</p>

<h2></h2>
<p>
规范代码风格有时是一个比较痛苦的过程，但是带来的收益却是可观的。偶尔犯懒，对于不好修改的代码，总想加个例外。但静下心来修改完之后，再看这段代码，会发现其实并没有什么例外。自己认为不好修改的部分，很多时候其实是原本的实现存在问题。
<strong><em>Special cases aren't special enough to break the rules.</em></strong>
</p>


<br />
<p>
最后，附上 The Zen of Python
{% blockquote The Zen of Python https://www.python.org/dev/peps/pep-0020/ PEP20 %}
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
{% endblockquote %}
</p>


<h2>参考</h2>
<ul>
<li><a href="https://www.python.org/dev/peps/pep-0008/">PEP 0008 -- Style Guide for Python Code</a></li>
<li><a href="https://flake8.readthedocs.org/en/latest/">Flake8</a></li>
<li><a href="https://en.wikipedia.org/wiki/Cyclomatic_complexity">Cyclomatic complexity</a></li>
<li><a href="https://en.wikipedia.org/wiki/Connected_component_(graph_theory)">Connected component (graph theory)</a></li>
<li><a href="https://en.wikipedia.org/wiki/Self-documenting_code">https://en.wikipedia.org/wiki/Self-documenting_code</a></li>
<li><a href="https://www.python.org/dev/peps/pep-0020/">PEP 20 -- The Zen of Python</a></li>
</ul>


