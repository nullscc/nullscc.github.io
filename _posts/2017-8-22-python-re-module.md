---
layout: post
title: 'python的RE模块'
tags: python
---
python的RE模块是很有必要学习的，因为很多python的功能都需要用到它。把它用熟会对工作有很大的帮助

### 标志项
re模块的很多函数都会用到标志参数，它对应python正则表达式里面的标志

### re.A | re.ASCII
在unicode字符串模式的情况下影响`\w, \W, \b, \B, \d, \D, \s, \S`，让它们只匹配ASCII码  
在bytes字符串模式下会被忽略

### re.DEBUG
显示编译表达式的调试信息

### re.I | re.IGNORECASE
忽略大小写

### re.L | re.LOCALE
让`\w, \W, \b, \B, \s, \S`依赖于本地化  
在python3.6中，`re.LOCALE`只能用于bytes模式，并且不与`re.ASCII`兼容

### re.M | re.MULTILINE
默认的`^`与`$`只匹配字符串的开头与结尾，但是如果指定了这个标记，那么它们还会匹配每一行的开头与结尾

### re.S | re.DOTALL
默认的，`.`会匹配除了表示新行的（例如`\n`）其他所有字符，但是如果加上了这个标记，那么`.`会匹配所有字符

### re.X | re.VERBOSE
允许你写一些可读性更强的正则表达式，可以增加注释，但是其中的空白符会被忽略(除非这些空白符之前是未转义的反斜杠)，不在 character class 里面并且前面没有未转义的反斜杠的`#`后的字符会被忽略

    a = re.compile(r"""\d +  # the integral part
                    \.    # the decimal point
                    \d *  # some fractional digits""", re.X)
    b = re.compile(r"\d+\.\d*")

re 模块允许多个线程共享同一个已编译的正则表达式对象，也支持命名子组。

## re模块的函数
### re.compile
re.compile会产生一个正则表达式对象，python会将它缓存起来
```python
    prog = re.compile(pattern)
    result = prog.match(string)
```
等价于
```python
    result = re.match(pattern, string)
```

### re.search(pattern, string, flags=0)
搜索模式匹配的第一个位置，并且返回相应的匹配对象  
如果没有匹配项，那么返回None

### re.match(pattern, string, flags=0)
匹配字符串的起始位置，如果匹配到，则返回相应的匹配对象。如果没匹配到，那么返回None。  
如果想在字符串的任意位置都能找到模式对应的项，那么使用`re.search`会更好  

### re.fullmatch(pattern, string, flags=0)
仅仅在模式完全匹配字符串的时候才返回一个匹配对象  

### re.split(pattern, string, maxsplit=0, flags=0)
根据模式分隔字符串，返回一个列表，如果使用了括号分组模式，那么分组表示的字符串也会在返回列表中  
如果maxsplit指定了一个非零值，那么最多多少次分割，剩下的会成为最后一个元素  
```python
    >>> re.split('\W+', 'Words, words, words.')
    ['Words', 'words', 'words', '']
    >>> re.split('(\W+)', 'Words, words, words.')
    ['Words', ', ', 'words', ', ', 'words', '.', '']
    >>> re.split('\W+', 'Words, words, words.', 1)
    ['Words', 'words, words.']
    >>> re.split('[a-f]+', '0a3B9', flags=re.IGNORECASE)
    ['0', '3', '9']
```

如果模式使用了括号进行分组，并且模式匹配在字符串的开始位置发生了，那么结果会以一个空字符串开头
```python
    >>> re.split('(\W+)', '...words, words...')
    ['', '...', 'words', ', ', 'words', '...', '']
```
`split()`不会匹配一个空的模式匹配：
```python
    >>> re.split('x*', 'axbc')
    ['a', 'bc']
```
上面例子中0个`x`就是一个空的模式匹配

空字符串的模式会返回一个`ValueError`
```python
    >>> re.split("^$", "foo\n\nbar\n", flags=re.M)
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    ...
    ValueError: split() requires a non-empty pattern match.
```
需要注意的是，即便指定了`re.M | re.MULTILINE`标记，此函数还是只会匹配字符串的开始位置而不会匹配每行字符串的开始位置

### re.findall(pattern, string, flags=0)
按顺序返回所有的匹配列表，如果pattern中有group，那么返回一个group列表；如果pattern有不止一个group，那么返回元组列表。

### re.finditer(pattern, string, flags=0)
返回一个迭代器，这个迭代器会按顺序yield出匹配对象

### re.sub(pattern, repl, string, count=0, flags=0)
* 在字符串string中用repl代替找到的pattern
* 如果没找到pattern，那么返回原字符串
* repl可以是字符串或者函数
* 如果repl是字符串，任何反斜杠的转换都会执行，例如`\n`会被转换为单个的新行字符，反向引用，例如`\1`会被第1个子组的字符串代替
```python
        >>> re.sub(r'def\s+([a-zA-Z_][a-zA-Z_0-9]*)\s*\(\s*\):',
        ...        r'static PyObject*\npy_\1(void)\n{',
        ...        'def myfunc():')
        'static PyObject*\npy_myfunc(void)\n{'
```
* 如果repl是一个函数，那么它会在每次模式匹配成功时被调用，这个函数有一个匹配对象参数，并且返回替换后的字符串，例如：
```python
        >>> def dashrepl(matchobj):
        ...     if matchobj.group(0) == '-': return ' '
        ...     else: return '-'
        >>> re.sub('-{1,2}', dashrepl, 'pro----gram-files')
        'pro--gram files'
        >>> re.sub(r'\sAND\s', ' & ', 'Baked Beans And Spam', flags=re.IGNORECASE)
        'Baked Beans & Spam'
```
* pattern可以是一个字符串或者一个正则表达式对象
* count是最大替换次数，count必须是一个非负数
* 空的匹配可能发生，不过他们必须互不相邻，所以`sub('x*', '-', 'abc')`返回一个`-a-b-c-`
* 当repl是一个字符串时，`\g<name>`对象的是`(?p<name>...)`，`\g<number>`会使用对应的数字组，因此`\g<2>`与`\2`是等同的。但是不会引起歧义，例如`\g<2>0`会被翻译成`\20`而不是`2`加上一个字符`0`，`\g<0>`代替整个匹配的字符串

### re.subn(pattern, repl, string, count=0, flags=0)
等同于`sub`但是会返回一个tuple：`(new_string, number_of_subs_made)`

### re.escape(pattern)
用\来转换所有的字符(除了ASCII字符、数字、'_')，通常用在如果你想匹配任意有正则表达式元字符的地方
```python
    >>> print(re.escape('python.exe'))
    python\.exe

    >>> legal_chars = string.ascii_lowercase + string.digits + "!#$%&'*+-.^_`|~:"
    >>> print('[%s]+' % re.escape(legal_chars))
    [abcdefghijklmnopqrstuvwxyz0123456789\!\#\$\%\&\'\*\+\-\.\^_\`\|\~\:]+

    >>> operators = ['+', '-', '*', '/', '**']
    >>> print('|'.join(map(re.escape, sorted(operators, reverse=True))))
    \/|\-|\+|\*\*|\*
```
### re.purge()
正则表达式对象会被缓存的，这个函数用来清除正则表达式缓存

### exception re.error(msg, pattern=None, pos=None)
当正则表达式书写错误、表达式匹配出错(不包括没有匹配到)、表达式编译出错时可能抛出的异常  

* msg：正则表达式格式错误的消息
* pattern：正则表达式模式
* pos：编译错误发生的索引点，可能为None
* lineno：与pos对应的行数，可能为None
* colno：与pos对应的列数，可能为None

## 正则表达式对象的方法和属性
什么是正则表达式对象？ 由`re.compile`返回的就是正则表达式对象

### regex.search(string[, pos[, endpos]])
扫描第一个正则表达式产生匹配的位置，并且返回一个匹配对象(如果没找到则返回一个None)。

* pos参数指定从字符串的哪里开始扫描，默认是0，它不完全等于字符串的切片
* `^` 会匹配真正的字符串起始处与每行的开始处，但是不必要一定是整个字符串的开始处
* endpos，对应于pos，表示结束搜索的索引
```python
    >>> pattern = re.compile("d")
    >>> pattern.search("dog")     # Match at index 0
    <_sre.SRE_Match object; span=(0, 1), match='d'>
    >>> pattern.search("dog", 1)  # No match; search doesn't include the "d"
```
### regex.match(string[, pos[, endpos]])
和regex.search不一样，`regex.match`仅仅从字符串的最开始处匹配，返回相应的匹配对象
```python
    >>> pattern = re.compile("o")
    >>> pattern.match("dog")      # No match as "o" is not at the start of "dog".
    >>> pattern.match("dog", 1)   # Match as "o" is the 2nd character of "dog".
    <_sre.SRE_Match object; span=(1, 2), match='o'>
```
### regex.fullmatch(string[, pos[, endpos]])
如果正则表达式匹配全部的字符串，那么返回一个匹配对象。
pos和endpos参数和在`search()`中一样
```python
    >>> pattern = re.compile("o[gh]")
    >>> pattern.fullmatch("dog")      # No match as "o" is not at the start of "dog".
    >>> pattern.fullmatch("ogre")     # No match as not the full string matches.
    >>> pattern.fullmatch("doggie", 1, 3)   # Matches within given limits.
    <_sre.SRE_Match object; span=(1, 3), match='og'>
```
### regex.split(string, maxsplit=0)
和`split()`函数功能一样，只不过使用的是编译过后的模式

### regex.findall(string[, pos[, endpos]])
和`findall()`函数功能一样，只不过使用的是编译过后的模式

### regex.finditer(string[, pos[, endpos]])
和`finditer()`函数功能一样，只不过使用的是编译过后的模式

### regex.sub(repl, string, count=0)
和`sub()`函数功能一样，只不过使用的是编译过后的模式

### regex.subn(repl, string, count=0)
和`subn()`函数功能一样，只不过使用的是编译过后的模式

### regex.flags
代表匹配的标记，它是一个标记的联合，包括`compile` `(?...)` 或者隐式的标记(例如如果是一个Unicode字符串模式`UNICODE`

### regex.groups
在模式中的匹配的数量

### regex.groupindex
如果存在使用`(?P<id>)`标记的组，那么会返回一个字典，如果不存在也会返回一个空的字典

### regex.pattern
表示这个正则表达式对象是由哪个模式字符串编译而来的


## 匹配对象
匹配对象就是由`re.search()`、`re.match()`等函数返回的对象

它有一些方法和属性
### match.expand(template)
它做的是一些`\`的替换工作，返回替换后的字符串，和`sub()`函数的功能类似

一些例子：

* `\n`会被转换成合适的字符
* `\1, \2`会被转换成对应的数字反向引用
* `\g<1>, \g<name>`会被相应的组替换

### match.group([group1, ...])
* 返回一个或多个匹配的子组
* 如果它只有一个参数，那么结果是一个单个的字符串
* 如果它有多个参数，那么结果是一个元组
* 如果没有任何参数，group1默认是0，那么整个匹配会返回。
* 如果groupN是0，那么响应的返回值是整个匹配字符串
* 如果参数是1-99，那么会返回响应的括号分组
* 如果参数为负数或者大于定义的子组数，那么`IndexError`异常会被抛出。
* 如果一个分组匹配多次，那么仅仅最后一次匹配会返回
```python
        >>> m = re.match(r"(..)+", "a1b2c3")  # Matches 3 times.
        >>> m.group(1)                        # Returns only the last match.
        'c3'
```
* 如果正则表达式使用`?P<name>...`标记，那么参数也可以是分组名，如果字符串参数不是一个分组名，那么抛出`IndexError`异常

数字分组：
```python
    >>> m = re.match(r"(\w+) (\w+)", "Isaac Newton, physicist")
    >>> m.group(0)       # The entire match
    'Isaac Newton'
    >>> m.group(1)       # The first parenthesized subgroup.
    'Isaac'
    >>> m.group(2)       # The second parenthesized subgroup.
    'Newton'
    >>> m.group(1, 2)    # Multiple arguments give us a tuple.
    ('Isaac', 'Newton')
```
字符串分组：
```python
    >>> m = re.match(r"(?P<first_name>\w+) (?P<last_name>\w+)", "Malcolm Reynolds")
    >>> m.group('first_name')
    'Malcolm'
    >>> m.group('last_name')
    'Reynolds'
```
字符串分组也可以以数字分组的形式访问(还是上面的例子)
```python
    >>> m.group(1)
    'Malcolm'
    >>> m.group(2)
    'Reynolds'
```
### match.__getitem__(g)
和`m.group(g)`是一样的。这允许

### match.groups(default=None)
返回匹配的所有子组
```python
    >>> m = re.match(r"(\d+)\.(\d+)", "24.1632")
    >>> m.groups()
    ('24', '1632')
```
关于default，如果匹配的到的分组少于定义的分组，那么返回的元组中对应的元素会用default指代的元素进行表示
```python
    >>> m = re.match(r"(\d+)\.?(\d+)?", "24")
    >>> m.groups()      # Second group defaults to None.
    ('24', None)
    >>> m.groups('0')   # Now, the second group defaults to '0'.
    ('24', '0')
```
### match.groupdict(default=None)
返回所有的命名子组
```python
    >>> m = re.match(r"(?P<first_name>\w+) (?P<last_name>\w+)", "Malcolm Reynolds")
    >>> m.groupdict()
    {'first_name': 'Malcolm', 'last_name': 'Reynolds'}
```
default的含义和`match.groups()`是一样的

### match.start([group]) | match.end([group])
返回子组的索引值

默认参数均为0，如果group匹配空字符串，那么`m.start(group)=m.end(group)`

例子：
```python
    m = re.search('b(c?)', 'cba')
    # m.start(0) 是 1
    # m.end(0) 是 2 
    # m.start(1) 和 m.end(1) 都是 2
    # m.start(2)会抛出异常
```
另外一个例子(去除邮箱中的`remove_this`)：
```python
    >>> email = "tony@tiremove_thisger.net"
    >>> m = re.search("remove_this", email)
    >>> email[:m.start()] + email[m.end():]
    'tony@tiger.net'
```
### match.span([group])
返回一个元组：`(m.start(group), m.end(group))`，group默认是0，如果group没有匹配到，那么返回`(-1, -1)`

### match.pos
正则表达式对象的`search()`和`match()`方法的第二个参数

### match.endpos
正则表达式对象的`search()`和`match()`方法的第三个参数

### match.lastindex
最后匹配到的子组的索引，如果没有分子匹配到，那么返回`None`  
例如，对于字符串`'ab'`，正则表达式`(a)b`、`((a)(b))`、`((ab))`的`lastindex`均为1  
表达式`(a)(b)`的`lastindex`均为2

### match.lastgroup
最后匹配到的命名分组的名字，如果分组没有名字返回`None`

### match.re
匹配对象的正则表达式对象

### match.string
正则表达式对象的`match()`或者`search()`方法的参数值


## 一些特殊用例
### 模拟scanf()
* %c : .
* %5c : .{5}
* %d : [-+]?\d+
* %e, %E, %f, %g : [-+]?(\d+(\.\d*)?|\.\d+)([eE][-+]?\d+)?
* %i : [-+]?(0[xX][\dA-Fa-f]+|0[0-7]*|\d+)
* %o :[-+]?[0-7]+
* %s : \S+
* %u : \d+
* %x, %X : [-+]?(0[xX])?[\dA-Fa-f]+

### 在search函数中使用^
本来search函数可以
```python
    >>> re.match("c", "abcdef")    # No match
    >>> re.search("^c", "abcdef")  # No match
    >>> re.search("^a", "abcdef")  # Match
    <_sre.SRE_Match object; span=(0, 1), match='a'>
```
### 打乱文本
```python
    >>> def repl(m):
    ...     inner_word = list(m.group(2))
    ...     random.shuffle(inner_word)
    ...     return m.group(1) + "".join(inner_word) + m.group(3)
    >>> text = "Professor Abdolmalek, please report your absences promptly."
    >>> re.sub(r"(\w)(\w+)(\w)", repl, text)
    'Poefsrosr Aealmlobdk, pslaee reorpt your abnseces plmrptoy.'
    >>> re.sub(r"(\w)(\w+)(\w)", repl, text)
    'Pofsroser Aodlambelk, plasee reoprt yuor asnebces potlmrpy.'
```

## 总结
python的模块很强大，其实主要弄懂re模块的函数与两个对象的方法和属性就行了，一个是正则表达式对象，它是`re.compile`函数返回的对象，一个是匹配对象，它是`search`和`match`返回的对象，其中的正则表达式对象python缓存起来，所以速度会更快。

最妙的是re模块的函数和正则表达式的方法名称是一样的，不过这有时候会成为困扰，不过弄懂它们之间的区别就好办了。