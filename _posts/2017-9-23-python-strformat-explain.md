---
layout: post
title: 'python的str.format详解'
tags: python
---
python中的另一种字符串格式化的方式是使用str对象的`format`函数，这里就来稍微详细的看一下这个函数的功能。其实我们在搜索引擎上看到这个函数的用法只是其中的很少一部分，它是一个很灵活、多变的函数，熟悉它会使很多工作都事半功倍。

先来看看这个函数的官方说明：

> Perform a string formatting operation. The string on which this method is called can contain literal text or replacement fields delimited by braces {}. Each replacement field contains either the numeric index of a positional argument, or the name of a keyword argument. Returns a copy of the string where each replacement field is replaced with the string value of the corresponding argument.

说明很简单，大概意思就是说字符串中可以使用`{}`这样一个代替区域来控制变化显示的内容，然后它返回一个替换后的字符串。

看起来好像和 % 类似，但是在这里我们可以看出一个区别是： % 是一个操作符，而format是一个函数。

然后我们来看看详细的格式字符串语法来看看他们到底有什么区别。

`{}`中的内容可以有很多内容，它们中的所有内容都是可以省略的，正是这样一个特性使得这个函数可以做很简单的事，也可以做很复杂的事。

`{}`中的语法主要概括为以下：
```python
    replacement_field ::=  “{” [field_name] [“!” conversion] [“:” format_spec] “}”
    field_name        ::=  arg_name (“.” attribute_name | “[” element_index “]”)*
    arg_name          ::=  [identifier | integer]
    attribute_name    ::=  identifier
    element_index     ::=  integer | index_string
    index_string      ::=  <any source character except “]”> +
    conversion        ::=  “r” | “s” | “a”
    format_spec       ::=  <described in the next section>
```

看的有点懵逼？没关系，一个个来解释。

field_name，可以是数字或者标识符(注意如果是标识符的情况，由于它没有引号来限定它，所以不可能制定任意的字典key，比如`10`、`:-]`)，然后后面还可以使用`.`来访问它的属性或者使用`[]`来访问它的索引。来个例子说明下：
```python
    class hello():
        hi = 1

    h = hello()

    "hello {0.hi}".format(h)

    "hello {haha.hi}".format(haha=h)

    L = [1, 2, 3]
    "hello {0[1]}".format(L)

    D = {"ha":"python", "hi":"lua"}
    "hello {0[ha]}".format(D)
```
conversion，它的存在是用来改变format函数的默认行为的，format函数的显示默认是用对象的`__format__`函数，但是我们可以通过conversion选项来改变它的默认行为(可以理解为在`__format__`之前执行，所以先返回了)，它的选项有三个：

    1. `r` 使用`repr()`函数来显示
    2. `s` 使用`str()`函数来显示
    3. `a` 使用`ascii()`函数来显示

最后再来说说`format_spec`，先来个总览：
```bash
    format_spec     ::=  [[fill]align][sign][#][0][width][grouping_option][.precision][type]
    fill            ::=  <any character>
    align           ::=  “<” | “>” | “=” | “^”
    sign            ::=  “+” | “-” | ” “
    width           ::=  integer
    grouping_option ::=  “_” | “,”
    precision       ::=  integer
    type            ::=  “b” | “c” | “d” | “e” | “E” | “f” | “F” | “g” | “G” | “n” | “o” | “s” | “x” | “X” | “%”
```
当然他们也都是可选的，相信知道 % 操作符用法的人看到这里有点熟悉了。

* fill，填充字符，一般来说会和align、width配合使用，它表示如果要显示的字符不够了用什么来代替，默认是空格，它可以是任何字符
* align，指定对齐方式，它有四个选项：

    1. `<` 左对齐
    2. `>` 右对齐
    3. `=` 表示填充符会在正负号之后数字之前(比如我要显示成：`+000000120`)
    4. `^` 居中对齐

* sign，指定正负数字的显示方式(主要是正数怎么显示)

    1. `+` 当一个数是整数应该在它前面显示`+`
    2. `-` 指示只有当数字是负数时才显示符号：`-`
    3. ` ` 指示整数之前应该有个空格，以便和负数的`-`对齐

* grouping_option，主要用来显示大数字(上千)，这样便于提高数字的辨识度
    1. `,` 用逗号来分隔
    2. `_` 用下划线来分隔

        '{:,}'.format(1234567890)
        '{:_}'.format(1234567890)

* precision，主要用来控制字符串或者浮点型数据的显示精度
* type，控制是要显示字符串(s)还是浮点型数据(f)

这里需要注意的是`#`可以用来显示一些特殊的对象，比如一个数字是`0o10`，那么默认来说打印这个数字是不会显示八进制标识符:`0o`

可以看出str.format比 % 操作符更加复杂多变，但是它也可以用很简单的方式实现和 % 操作符相同的功能，推荐使用 str.format函数，而不是 %