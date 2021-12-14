---
title: python调用C/C++动态链接库以及socket通信
date:  2021-12-14 10:25:13
category: work
tags: DLL
excerpt: 在python中调用DLL动态链接库以及socket服务
---

### 一、python调用C/C++动态链接库
#### （1）配置环境

实验环境：win10，python 3.8.6，PyCharm 2020.3.5 x64<br>
在PyCharm下新建项目，将所需DLL动态链接库复制到项目文件夹下，创建测试用例dllTest.py<br>
通过`from ctypes import *`导入ctypes库，使用`mydll = CDLL(".\\testDLL.dll")`导入动态链接库，如果出现FileNotFoundError: Could not find module 'xxx.dll'错误，注意dll库的拼写(别打错了...)或者路径是否正确

**注意python应用程序与所调用的dll动态链接库是32位还是64位的**<br>
如果出现OSError: [WinError 193] %1 不是有效的 Win32 应用程序错误，则是因为当前的python环境与动态链接库不匹配<br>

#### （2）测试用例

dllTest.py源文件

```python
import platform
from ctypes import *

class MyPoint(Structure):
    _fields_ = [("x", c_float), ("y", c_float)]

mydll = CDLL(".\\tsetDLL.dll")
print(mydll.add(10, 20))

mydll.addf.restype = c_float
mydll.addf.argtypes = [c_float, c_float]
print(mydll.addf(21.5, 40.3))

mydll.Print_point.restype = None
mydll.Print_point.argtypes = [POINTER(MyPoint), ]
point = MyPoint(1.2, 92.1)
print(mydll.Print_point(byref(point)))
```

`argtypes`和`restype`属性用于描述DLL动态链接库中的函数接口的参数类型和返回值类型(python中默认类型为int)，ctypes库的更多用法参考[引用(1)](https://www.cnblogs.com/night-ride-depart/p/4907613.html)

### 二、socket通信
#### （1）demo用例

```python
import socket
import threading

class Client(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.threadName = "client thread"
        self.clinetSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    def run(self):
        self.clinetSocket.connect(('127.0.0.1', 8888))
        msg = 'Hi server, this is client'
        self.clinetSocket.send(msg.encode('utf-8'))
        recMsg = self.clinetSocket.recv(1024)
        print(recMsg.decode('utf-8'))
        self.clinetSocket.close()

class Server(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.threadName = "server thread"
        self.serverSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    def run(self):
        self.serverSocket.bind(('127.0.0.1', 8888))
        self.serverSocket.listen(5)
        while(True):
            connectClientSocket, clientAddr = self.serverSocket.accept()
            print('Hello, connect server success, client from: {0}'.format(str(clientAddr)))
            msg = 'this is a text from server'
            connectClientSocket.send(msg.encode('utf-8'))
            receiveMsg = connectClientSocket.recv(1024)
            print(receiveMsg.decode(('utf-8')))
            connectClientSocket.close()

if __name__ == '__main__':
    myClinet = Client()
    myServer = Server()
    myClinet.start()
    myServer.start()
```
服务端Server发送消息使用`self.serverSocket.send()`会报错，应该使用accept()方法返回的socket<br>
发送消息以及接收消息需要对字符串先编码以及解码<br>

python中使用线程有两种方式：函数或者用类来包装线程对象

1. 函数式，直接创建`class threading.Thread(group=None, target=None, name=None, args=(), kwargs={}, *, daemon=None)`，将一个callable对象从类的构造器传递进去，这个callable就是回调函数，用来处理任务
2. 从threading.Thread继承创建一个新的子类(在构造函数中实现threading.Thread.\_\_init\_\_())，然后重写父类的run()方法，实例化后调用start()方法启动新线程

### Reference

1. [python ctypes 探究 ---- python 与 c 的交互](https://www.cnblogs.com/night-ride-depart/p/4907613.html)
2. [Python3 网络编程](https://www.runoob.com/python3/python3-socket.html)
3. [Python3 多线程](https://www.runoob.com/python3/python3-multithreading.html)
4. [Python多线程编程(一）：threading 模块 Thread 类的用法详解](https://blog.csdn.net/briblue/article/details/85101144)
5. [Source code: skf2py](https://gist.github.com/zz-png/507d66c951d9a84d59c9040ca914bca2)