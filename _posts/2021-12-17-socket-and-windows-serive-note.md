---
title: Windows服务程序之socket监听本地端口
date:  2021-12-17 14:25:13
category: work
tags: windows service
excerpt: 用python实现windows服务程序监听本地端口
---

### （1）环境介绍
win10，python 3.8.6，PyCharm 2020.3.5 x64，依赖库：pywin32 302，pyinstaller 4.7

### （2）服务端代码myService.py

```python
import  win32serviceutil
import  win32service
import  win32event
import  threading
import  socket
import  os, sys, time

class Server():

    def __init__(self):
        self.serverSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.serverSocket.bind(('127.0.0.1', 8888))
        self.serverSocket.listen(5)


class MyTestService(win32serviceutil.ServiceFramework):

    _svc_name_ = "MyWindowsService"
    _svc_display_name_ = "myservicexxx"
    _svc_description_ = "this is my windows service"

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)
        self.run = True

    def SvcRun(self):
        myListener = Server()
        while(self.run):
            file = open('E:\\demo.txt', 'a')
            tip = "get message:"
            file.write(tip)
            connectClient, addr = myListener.serverSocket.accept()
            data = connectClient.recv(1024)
            now = time.localtime(time.time())
            msg = data.decode('utf-8')
            connectClient.send(tip.encode('utf-8'))
            connectClient.send(msg.encode('utf-8'))
            connectClient.close()
            file.write(str(now))
            file.write(msg)
            file.close()

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.hWaitStop)
        self.run = False


if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(MyTestService)
```
通过`class Server`类创建socket对象，绑定本地127.0.0.1的8888号端口，最大连接数为5，`class MyTestService`类向上继承`win32serviceutil.ServiceFramework`，有3个必须定义的成员变量：\_svc\_name\_(服务名)，\_svc\_display\_name\_(服务展示名称)，\_svc\_description\_(服务描述)<br>

**PS:注意变量名后面有下划线_，不要打错了~不然会报错：has no attribute '\_svc\_name\_'...**

### （3）启动服务
这里是在PyCharm终端Terminal中通过命令行安装以及启动服务，未使用pyinstaller打包exe程序，**注意：**
1. 需要以管理员权限运行PyCharm才能够在终端中执行命令行`python myService.py install`安装服务，否则会报PermissionError错误
2. 执行`python myService.py start`，如果启动服务失败，提示Error starting service:服务没有及时响应启动或控制请求，检查一下有没有将python添加到系统环境变量PATH中（不是用户变量）
3. 执行`python myService.py stop`停止服务，提示请求的控件对此服务无效（在任务管理器中停止服务也同样）的错误，暂未解决

### （4）打包exe可执行程序
这里使用pyinstaller进行打包，先通过`pip install pyinstaller`安装依赖库，如果使用pip安装失败，可以尝试直接下载安装文件解压安装<br>

安装成功后，打开命令行工具进入所需打包py文件目录，运行`pyinstaller -F myService.py`进行打包，pyinstaller相关输入参数的含义为：

+ -F 表示生成单个可执行文件(将动态链接库都打包进去，默认为-D，会产生一个包含多个文件的目录作为可执行程序)
+ -w 表示程序运行时不显示控制台窗口，这在程序有GUI界面时非常有用，但如果是命令行程序的话就删除这个选项吧
+ -p 表示自定义设置python导入模块的路径
+ -i 表示生成的可执行文件的logo图标

**需要注意的地方有：**
1. 执行上述打包命令`pyinstaller -F myService.py`后出现api-ms-win-crt-math-\|1-1-0.dll is empty错误，提示需要的系统库文件为空，我在本机D:\Windows Kits\10\Redist\10.0.19041.0\ucrt\DLLs\x64中找到了所需的对应库文件，如果没有应该需要去下载
2. 出现了以上错误后，在相应目录下生成了myService.spec文件，打开该文件并编辑Analysis中的pathex字段，将需要的库所在路径加入该字段，这里我填的是`pathex=['D:\\WindowsKits\\10\\Redist\\10_0_19041_0\\ucrt\\DLLs\\x64']`（我修改了目录名，因为原目录名Windows Kits中有空格好像导致了错误，顺便将10.0.19041.0中的dot改为了下划线_）
3. 解决上述问题后重新执行命令，出现了no module named win32timezone错误，这是因为pyinstaller有时无法检测到全部需要的依赖包，解决方法为在上述生成的myService.spec中修改Analysis中的hiddenimports字段，添加所需要的依赖包，这里我编辑为`hiddenimports=['win32timezone']`
4. 最后执行`pyinstaller -F myService.spec`命令，在dist目录下成功生成了exe可执行程序

### （5）执行exe程序时出现的错误

生成可执行程序后，通过windows的命令行工具(同样需要以管理员权限运行)进入exe所在目录，执行`myService.exe install`安装服务，`myService.exe start`启动服务，出现Failed to execute script xxx.exe(/1053:服务没有及时响应启动或控制请求)，按照[引用1](https://www.cnblogs.com/dcb3688/p/4496934.html)中的方法，修改python文件中的main函数后重新打包程序即可正常运行服务

```python
if __name__=='__main__':
    if len(sys.argv) == 1:
        try:
            evtsrc_dll = os.path.abspath(servicemanager.__file__)
            servicemanager.PrepareToHostSingle(MyTestService)
            servicemanager.Initialize('MyTestService', evtsrc_dll)
            servicemanager.StartServiceCtrlDispatcher()
        except win32service.error as details:
            if details[0] == winerror.ERROR_FAILED_SERVICE_CONTROLLER_CONNECT:
                win32serviceutil.usage()
    else:
        win32serviceutil.HandleCommandLine(MyTestService)
```

### （6）客户端测试程序

```python
import socket
import time
import threading
socket.setdefaulttimeout(20)

if __name__ == '__main__':
    clinetSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    clinetSocket.connect(('127.0.0.1', 8888))
    msg = 'hi, this message is from client'
    clinetSocket.send(msg.encode('utf-8'))
    while True:
        recMsg = clinetSocket.recv(1024)
        if not recMsg:
            break
        print(recMsg.decode('utf-8'))
    clinetSocket.close()
```
启动服务后，打开新的终端运行客户端程序，成功在E:/demo.txt中写入传输的消息以及传输时间

**最后，感觉用python写windows服务还要打包成exe可执行程序确实意义不大，可以参考引用2**

### Reference
1. [Python 写Windows Service服务程序](https://www.cnblogs.com/dcb3688/p/4496934.html)
2. [Python打包EXE方法汇总整理](https://mp.weixin.qq.com/s/R13TprsYVrErF4quIxhGiw)
3. [Python Socket套接字编程](https://www.cnblogs.com/LyShark/p/11317152.html)
4. [python tcp 客户端服务端通信，使用json数据格式](https://www.cnblogs.com/dzswise/p/14764975.html)
5. [python实现socket互传json文件](https://blog.csdn.net/rockray21/article/details/109514873)