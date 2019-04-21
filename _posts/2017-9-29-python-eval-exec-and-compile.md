---
layout: post
title: 'python的eval、exec和compile'
tags: python
---
python和其他动态语言一样，可以调用一个函数来执行字符串的代码。主要函数为`eval`、`exec`

看看两个函数的声明吧。
```python
    eval(expression, globals=None, locals=None)
    
    exec(object[, globals[, locals]])
```
用法很简单，主要看下面的说明吧：

1. eval 主要用来算表达式
2. exec主要用来执行代码
3. exec中的return和yield基本无效
4. 如果code object是带`exec`编译的，那么`eval`的返回值会是None(因为exec不需要有返回值)
5. 为防止不信任的来源恶意执行一些代码，可以使用`ast.leteral_eval()`函数代替eval函数，缺点是它的限制比eval更多

它们的共同点：
1. 都可以在一定程度上执行代码
2. 第一个参数都可以是字符串或者code object

然后再来看看生成 code object 的函数吧
```python
    compile(source, filename, mode, flags=0, dont_inherit=False, optimize=-1)
```

它返回一个code object或者 AST object。看看它们的参数：

* source，可以是常规字符串(str或byte都可)或者一个AST object
* filename，存放代码的文件，如果代码没有放在文件里面，一般可以设置成一个空字符串
* mode，可选项为`exec`、`eval`、`single`
* optimization，影响优化等级
* 另外两个参数影响 Future Statement

如果source有问题，那么会抛出`SyntaxError`，如果source包含空byte，那么会抛出`ValueError`异常