# Init进程概述
init进程是Android系统中用户空间的第一个进程，其进程id为1.作为天字第一号进程，其作用主要有以下几点
- 创建系统目录
- 装载文件系统
- 启动守护进程
- 初始化Android属性系统


init.code 通过解析init.rc文件,在Android5.1中,"import /init.${ro.zygote}.rc"中的变量为
决定系统使用/system/core/rootdir下面的init.zygote32.rc，init.zygote32_64.rc，init.zy
gote64.rc，init.zygote64_32.rc的其中哪一个文件

在init.c的main函数中的最后，系统会进入一个无限for循环中，在这个循环中会重启那些已经死去的service
