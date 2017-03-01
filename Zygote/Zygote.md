# Zygote概述
zygote本身是一个native应用程序，与驱动，内核等均无关系。
zygote最初的名字叫做"app_process"(在Android.mk中指定的),在运行过程中通过Linux下的pctrl的调用将名字换成了"zygote".

zygote的原型app_process所对应的文件为app_main.cpp

app_main::main中最终会调用AndroidRuntime::start方法

AndroidRuntime::start中主要完成一下三件事情：
- startVm(&mJavaVM, &env, zygote):创建虚拟机
- startReg(env):注册android的native函数
- CallStaticVoidMethod:调用com.android.internal.os.ZygoteInit的main函数

通过以上三件事情，便开启了从native通往java世界的大门

## ZygoteInit分析
ZygoteInit.main()中主要完成以下五件事情：
- 建立IPC通信服务端:registerZygoteSocket()
- 预加载相关资源:preload().类，资源文件，OpenGL相关，共享库，WebView
- 启动system_server进程:startSystemServer()
- 处理客户端连接和客户端请求:runSelectLoop()
