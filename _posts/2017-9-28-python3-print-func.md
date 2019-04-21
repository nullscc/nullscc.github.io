---
layout: post
title: 'python3的print函数'
tags: python
---
print函数是在python3下才有的说法，python2下print是一个关键字，我们一般用print函数时，一般只使用一些简单的参数。但是print函数是很强大的。

先来看看print函数的声明：
```python
	print(*objects, sep=’ ‘, end=’\n’, file=sys.stdout, flush=False)
```
看它有这么多参数，所以它不像它表面上看的那么简单(其实它也很简单)，来看看这些参数代表什么意思吧。

* objects，我们一般使用print时传入的非关键字参数都被打包成tuple传递给这个参数了
* sep，表示分隔符，看看以下代码的输出就知道了

		print("hello", "world")
		print("hello", "world", sep = '_') 
		
* end表示输出一行时的结尾是什么
* file表示输出输出到哪个文件，默认是`sys.stdout`，所以一般是输出到标准输出(终端窗口)，需要注意的是file参数需要有一个`write(string)`方法
* flush，意思是说是否立即刷新缓冲区。如果立即刷新缓冲区的话，代码就能立即输出(在`end!='\n'`时可能会出现“意想不到”的“输出”)，看看以下代码的输出就知道啦。
```python
		print("hello", end=" ")
		time.sleep(1)
		print("world", end=" ")
		
		print("love", end=" ", flush=True)
		time.sleep(1)
		print("python", end=" ", flush=True)
```