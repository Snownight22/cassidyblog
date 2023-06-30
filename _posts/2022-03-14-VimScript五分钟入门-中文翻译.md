---
layout: post
title:  "VimScript五分钟入门-中文翻译"
date:   2022-03-14 14:04:00 +0800
categories: VIM
tags: 翻译
topping: true
---


这篇文章主要是为了让你能够尽可能快地学习到vimscript的基础。你可以把这当做一个速查表。  

在读这篇文章之前，你应该可能已经有了一些编程经验。Vim的内建文档已经很出色了。你可以在vim里尝试`:h <searchterm>`来阅读更多信息。你可以通过在normal模式下键入gQ命令启动REPL环境来调试vimscript。  

***注意***：下面的例子中可能会包含像`<token>`一样的符号。这意味着它们可以被完全替代，包括`<`和`>`。 Vimscript使用`<`和`>`作为比较运算符。  

### 变量

`let` 用来设置一个变量。  

`unlet` 用来取消设置一个变量。  

`unlet!` 取消设置一个变量,并且忽略在变量不存在时的错误显示。  

默认情况下，在函数外初始化定义的变量是全局可见的，而在函数内定义的变量是函数局部可见的，你可能通过在变量名前加一个特定的前缀来限定变量的范围：  

>g:var - 全局范围。  
>a:var - 函数参数。  
>l:var - 函数局部变量。  
>b:var - buffer局部变量。  
>w:var - window局部变量。  
>t:var - tab局部变量。  
>v:var - VIM预定义的变量。  

你可以以`$variable`的形式引用变量来设置和获取环境变量，同时你可以以`$option`的形式引用变量来处理vim内置选项。  

### 数据类型

#### Number：32 位有符号数。  

>-123  
>0x10  
>0177  

#### Float：浮点型数。需要在 vim 编译中添加 +float 选项。  

>123.456  
>1.15e-6  
>-1.1e3  

#### String：以 NUL 结尾的由 8 位无符号字符组成的字符串。  

>"ab\txx\"--"  
>'x-z''a,c'  

#### Funcref：函数的引用，用做函数引用对象的变量必须以大写字母开头。  

>:let Myfunc = function("strlen")  
>:echo Myfunc('foobar') `"Call strlen on 'foobar'`  
>6  

#### List：有序列表  

>:let mylist = [1, 2, ['a', 'b']]  
>:echo mylist[0]  
>1  
>:echo mylist[2][0]  
>a  
>:echo mylist[-2]  
>2  
>:echo mylist[999]  
>E684: list index out of range: 999  
>:echo get(mylist, 999, "THERE IS NO 1000th ELEMENT")  
>THERE IS NO 1000th ELEMENT  

#### Dictionary： 一个相关联的无序数组。每个条目都有一个关键字和一个值。   

>:let mydict = {'blue': "#0000ff", 'foo': {999: "baz"}}  
>:echo mydict["blue"]  
>#0000ff  
>:echo mydict.foo  
>{999: "baz"}  
>:echo mydict.foo.999  
>baz  
>:let mydict.blue = "BLUE"  
>:echo mydict.blue  
>BLUE  

没有布尔类型. 整数 0 被当做`假`来处理, 其它都当做`真`.  

字符串在检查真假性前先被转化成数字。大多数字符串会转化成 0，除非字符串以一个数字开头。  

Vimscript是动态且弱类型的。  

>:echo 1 . "foo"  
>1foo  
>:echo 1 + "1"  
>2  
>  
>:function! TrueFalse(arg)  
>:&nbsp;&nbsp;&nbsp;&nbsp;return a:arg? "true" : "false"  
>:endfunction  
>
>:echo TrueFalse("foobar")  
>false  
>:echo TrueFalse("1000")  
>true  
>:echo TrueFalse("x1000")  
>false  
>:echo TrueFalse("1000x")  
>true  
>:echo TrueFalse("0")  
>false  

### 字符串条件和运算符  

\<string> == \<string>：字符串相等。  

\<string> != \<string>：字符串不相等。  

\<string> =~ \<pattern>：字符串匹配模式。  

\<string> !~ \<pattern>：字符串不匹配模式。  

\<operator>#：额外的匹配情况。  

\<operator>?：额外的不匹配情况。  

***注意***: Vim 选项 `ignorecase`设置大小写敏感度会影响`==`和`!=`运算符的比较结果。在运算符后面添加`?`或`#`可以根据大小写进行匹配。  

\<string> . \<string> ：连接两个字符串。  

>:function! TrueFalse(arg)  
>:&nbsp;&nbsp;&nbsp;&nbsp;return a:arg? "true" : "false"  
>:endfunction  
>
>:echo TrueFalse("X start" =~ 'X$')  
>false  
>:echo TrueFalse("end X" =~ 'X$')  
>true  
>:echo TrueFalse("end x" =~# 'X$')  
>false  

### If, For, While, 和 Try/Catch

>if \<expression>  
>&nbsp;&nbsp;&nbsp;&nbsp;...  
>elseif \expression>  
>&nbsp;&nbsp;&nbsp;&nbsp;...  
>else  
>&nbsp;&nbsp;&nbsp;&nbsp;...  
>endif  

>for \<var> in \<list>  
>&nbsp;&nbsp;&nbsp;&nbsp;continue  
>&nbsp;&nbsp;&nbsp;&nbsp;break  
>endfor  
>
>for [var1, var2] in [[1, 2], [3, 4]]
>&nbsp;&nbsp;&nbsp;&nbsp;"on 1st loop, var1 = 1 and var2 = 2  
>&nbsp;&nbsp;&nbsp;&nbsp;"on 2nd loop, var1 = 3 and var2 = 4  
>endfor  

>while \<expression>  
>endwhile  

>try  
>&nbsp;&nbsp;&nbsp;&nbsp;...  
>catch \<pattern (optional)>  
>&nbsp;&nbsp;&nbsp;&nbsp;"HIGHLY recommended to catch specific error.  
>finally  
>&nbsp;&nbsp;&nbsp;&nbsp;...  
>endtry  

### 函数

使用`function`关键字来定义函数。如果你想覆盖一个函数，则使用`function!`来代替。就像变量一样，函数也可以定义在特定范围内。  

***注意***：函数名必须以大写字母开头。  

>function! \<Name>(arg1, arg2, etc)  
>&nbsp;&nbsp;&nbsp;&nbsp;\<function body>  
>endfunction  

`delfunction <function>` 删除一个函数。  

`call <function>` 执行一个函数，并且除非是表达式的一部分,否则是需要使用`call`的。  

**例子**:强制创建一个全局函数(使用!)。用`...`当做最后一个参数，来表示可变参数列表。使用`a:1`来从`...`中获取第一个参数值，使用`a:2`获取第二个参数值，等等。`a:0` 特指`...`中拥有的参数数量。  

>function! g:Foobar(arg1, arg2, ...)  
>&nbsp;&nbsp;&nbsp;&nbsp;let first_argument = a:arg1  
>&nbsp;&nbsp;&nbsp;&nbsp;let index = 1  
>&nbsp;&nbsp;&nbsp;&nbsp;let variable_arg_1 = a:{index} " same as a:1  
>&nbsp;&nbsp;&nbsp;&nbsp;return variable_arg_1  
>endfunction  

有一种特定的方式来调用函数，就是可以在 buffer 中按照行的范围来调用。这种调用函数的方式就像 `1,3call Foobar()`。这个函数会在范围内的每一行都被执行一次。这个事例中，`Foobar`函数总共被调用了三次。  

如果你在函数列表后面添加关键字`range`，那么函数只会被调用一次。两个特定的变量在函数范围内是有效的：`a:firstline`和`a:lastline`。这些变量代表函数范围的开始和结束行号。  

**例子**：强制创建一个 buffer 函数用来打印函数被调用范围的大小。  

>function! b:RangeSize() range  
>&nbsp;&nbsp;&nbsp;&nbsp;echo a:lastline - a:firstline  
>endfunction  

### 类类型

Vim 并没有内置类类型，但你可以利用`Dictionary`对象来获得基本的类功能。需要定义一个类的方法时，在函数中使用`dict`关键字来将内部字典公开为`self`。  

>let MyClass = {foo: "Foo"}  
>function MyClass.printFoo() dict  
>&nbsp;&nbsp;&nbsp;&nbsp;echo self.foo  
>endfunction  

类默认为单例模式的。为了在 vimscript 中创建类的实例，需要在字典上调用`copy()`。  

>:let myinstance = copy(MyClass)  
>:call myinstance.printFoo()  
>Foo  
>:let myinstance.foo = "Bar"  
>:call myinstance.printFoo()  
>Bar  

### 接下来做什么

现在你已经入门了，这里有一些好的资源可以学习到更多：  

教程：  
[Vim's internal documentation](http://vimdoc.sourceforge.net/htmldoc/usr_41.html) -- (中文版 [Vim中文帮助手册](http://vimcdoc.sourceforge.net/doc/usr_41.html))   
[IBM's "Scripting the Vim Editor"](www.ibm.com/developerworks/linux/library/l-vim-script-1/index.html)  

其他人的代码：  
[Github: tpope](https://github.com/tpope)  
[Github: scrooloose](https://github.com/scrooloose)  

### 感谢

希望本文对你有用，感谢阅读。  

### 译注-vim资源

[vim中文速查手册](https://github.com/skywind3000/awesome-cheatsheets/blob/master/editors/vim.txt)  

[learn vimscript the hard way](https://learnvimscriptthehardway.stevelosh.com/) -- (中文版 [笨方法学习vimscript](https://www.kancloud.cn/kancloud/learn-vimscript-the-hard-way/49321))

---
**参考**：  
翻译工具：  
 - [有道翻译](https://fanyi.youdao.com)  
 - [Goldendict](http://goldendict.org)  

原文链接：<http://andrewscala.com/vimscript/>  


