---
title: python、main函数和argv参数
date: 2017-02-08 21:57:00
tags: [python,mian,argv]
categories: Python
description: "Python家的main函数"
---

笔者学习和使用过的语言中：C语言，C++语言，C#语言，Java语言都时有main函数在的，main是程序执行的起点，Python中，也有类似的运行机制，但方式却截然不同：Python使用缩进对齐组织代码的执行，所有没有缩进的代码（非函数定义和类定义），都会在载入时自动执行，这些代码，可以认为是Python的main函数。

举个列子，我们可以清楚的了解：
```python
im@58user:~/PythonProjects$ cat test.py 
#!/usr/bin/python
# -*- coding: utf-8 -*-
def main():
	print('world~!')
print('hello')
if __name__ == '__main__':
	main()
im@58user:~/PythonProjects$ python test.py 
hello
world~!
``` 

> 这样看来是否main函数没有多大的作用呢？

每个文件（模块）都可以任意写一些没有缩进的代码，并且在载入时自动执行，为了区分主执行文件还是被调用的文件，Python引入了一个变量__name__，当文件是被调用时，__name__的值为模块名，当文件被执行时，__name__为'__main__'。这个特性，我们可以在每个模块中写上测试代码，这些测试代码仅当模块被Python直接执行时才会运行，代码和测试完美的结合在一起。

```python
im@58user:~/PythonProjects$ cat test.py 
#!/usr/bin/python
# -*- coding: utf-8 -*-
import sys
def main(argv):
	if argv == None:
		print('world~!')
	else:
		print(argv)
print('hello')
if __name__ == '__main__':
	main(sys.argv)
im@58user:~/PythonProjects$ python test.py Tom
hello
['test.py', 'Tom']
```

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>