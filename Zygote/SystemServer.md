# SystemServer

## SystemServer的创建
system_server是通过Zygote.forkSystemServer()创建出来的.之后ZygoteInit通过handleSystemServerProcess()来分配给system_server任务.

在handleSystemServerProcess()方法中，最后进入RuntimeInit.zygoteInit().

### RuntimeInit.zygoteInit():
- nativeZygoteInit():首先进行了native层初始化,在调用该函数后，system_server会与Binder系统建立联系.
- invokeStaticMain():在方法的最后面会抛出一个ZygoteInit.MethodAndArgsCaller(Method method, String[] args)异常,该异常会在ZygoteInit.main()中被捕获处理.注意此时抛出的异常中方法名字为main方法.而ZygoteInit.main()对该异常的处理方式为调用MethodAndArgsCaller.run(), run()方法中通过反射调用了SystemServer.main().

### SystemServer.main():
- nativeInit():初始化传感器Service
- 创建SystemServiceManager
- 创建引导Service(BootstrapService)
- 创建核心Service(CoreService)
- 创建其它Service(OtherService)

#### 创建引导Service(BootstrapService)
一些为了让系统跑起来所必须的Service,并且它们之间有复杂的相互依赖性，这些Service在该方法中被创建出来.

这些Service有:
- ActivityManagerService
- PowerManagerService
- DisplayManagerService
- PackageManagerService

#### 创建核心Service
启动一些在引导过程中不会有相互依赖的基本Service.

这些Service有:
- LightsService
- BatteryService
- UsageStatsService
- WebViewUpdateService

#### 创建其它Service
创建其它的还没有被创建的Service:WindowManagerService,InputManagerService,AudioServiced等
