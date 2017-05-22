# Zygote的分裂
zygote分裂出了system_server后，便通过runSelectLoopMode等待并处理客户端的消息。Zygote是如何分裂呢？以一个Activity的启动为例来分析：

## Client建立联系并发送请求
通过startActivity方法开启一个新的Activity，经过层层调用会调用到AMS.startProcessLocked方法中

ActivityManagerService.startProcessLocked
```java
        private final void startProcessLocked(ProcessRecord app, String hostingType,
                    String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {

                    ......一些处理
                    // Start the process.  It will either succeed and return a result containing
                    // the PID of the new process, or else throw a RuntimeException.
                    boolean isActivityProcess = (entryPoint == null);
                    if (entryPoint == null) entryPoint = "android.app.ActivityThread";
                    checkTime(startTime, "startProcess: asking zygote to start proc");
                    Process.ProcessStartResult startResult = Process.start(entryPoint,
                            app.processName, uid, uid, gids, debugFlags, mountExternal,
                            app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                            app.info.dataDir, entryPointArgs);
                    .......
        }
```
AMS.startProcessLocked接下来会调用Process.start，并传入相关参数(ClassName为"android.app.ActivityThread")

```java
        public static final ProcessStartResult start(final String processClass,
                                          final String niceName,
                                          int uid, int gid, int[] gids,
                                          int debugFlags, int mountExternal,
                                          int targetSdkVersion,
                                          String seInfo,
                                          String abi,
                                          String instructionSet,
                                          String appDataDir,
                                          String[] zygoteArgs) {
                try {
                    return startViaZygote(processClass, niceName, uid, gid, gids,
                            debugFlags, mountExternal, targetSdkVersion, seInfo,
                            abi, instructionSet, appDataDir, zygoteArgs);
                } catch (ZygoteStartFailedEx ex) {
                    Log.e(LOG_TAG,
                            "Starting VM process through Zygote failed");
                    throw new RuntimeException(
                            "Starting VM process through Zygote failed", ex);
                }
        }
```
Process.startViaZygote:
```java
        private static ProcessStartResult startViaZygote(final String processClass,
                                          final String niceName,
                                          final int uid, final int gid,
                                          final int[] gids,
                                          int debugFlags, int mountExternal,
                                          int targetSdkVersion,
                                          String seInfo,
                                          String abi,
                                          String instructionSet,
                                          String appDataDir,
                                          String[] extraArgs)
                                          throws ZygoteStartFailedEx {
                synchronized(Process.class) {
                    ArrayList<String> argsForZygote = new ArrayList<String>();

                    // --runtime-init, --setuid=, --setgid=,
                    // and --setgroups= must go first
                    argsForZygote.add("--runtime-init");
                    argsForZygote.add("--setuid=" + uid);
                    argsForZygote.add("--setgid=" + gid);
                    ......一些debug模式下的参数
                    argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
                    ......又添加了一些参数
                    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
                }
        }
```
Process.startViaZygote会调用Process.zygoteSendArgsAndGetResult方法，并且会传入一个ZygoteState对象，ZygoteState对象中包含一个Socket对象。
```java
        private static ProcessStartResult zygoteSendArgsAndGetResult(
                    ZygoteState zygoteState, ArrayList<String> args)
                    throws ZygoteStartFailedEx {
                try {
                    /**
                     * See com.android.internal.os.ZygoteInit.readArgumentList()
                     * Presently the wire format to the zygote process is:
                     * a) a count of arguments (argc, in essence)
                     * b) a number of newline-separated argument strings equal to count
                     *
                     * After the zygote process reads these it will write the pid of
                     * the child or -1 on failure, followed by boolean to
                     * indicate whether a wrapper process was used.
                     */
                    final BufferedWriter writer = zygoteState.writer;
                    final DataInputStream inputStream = zygoteState.inputStream;

                    writer.write(Integer.toString(args.size()));
                    writer.newLine();

                    int sz = args.size();
                    for (int i = 0; i < sz; i++) {
                        String arg = args.get(i);
                        if (arg.indexOf('\n') >= 0) {
                            throw new ZygoteStartFailedEx(
                                    "embedded newlines not allowed");
                        }
                        writer.write(arg);
                        writer.newLine();
                    }

                    writer.flush();

                    // Should there be a timeout on this?
                    ProcessStartResult result = new ProcessStartResult();
                    result.pid = inputStream.readInt();
                    if (result.pid < 0) {
                        throw new ZygoteStartFailedEx("fork() failed");
                    }
                    result.usingWrapper = inputStream.readBoolean();
                    return result;
                } catch (IOException ex) {
                    zygoteState.close();
                    throw new ZygoteStartFailedEx(ex);
                }
        }
```
以上过程总结来说为ActivityManagerService中经过方法调用与ZygoteIinit建立了Socket联系，并发送请求命令来给ZygoteInit处理。

## Server(Zygote)对于Client请求的处理
在Zygote概述中说到Zygote开启了system_server子进程后，便通过runSelectLoop进入循环状态，来处理Client发送过来的命令请求

ZygoteInit.runSelectLoop:
```java
      private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
              ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
              ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
              FileDescriptor[] fdArray = new FileDescriptor[4];

              ......
              else if (index == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDescriptor());
              } else {
                boolean done;
                done = peers.get(index).runOnce();
                if (done) {
                    peers.remove(index);
                    fds.remove(index);
                }
              }
              ......
      }
```
runSelectLoop方法中会调用ZygoteConnection的runOnce函数

ZygoteConnection.runOnce:
```java
        boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
              ......
              try {
                  args = readArgumentList();//读取SystemServer发送过来的参数
                  descriptors = mSocket.getAncillaryFileDescriptors();
              }
              ......
              //分裂出一个子进程
              pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
              parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
              parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
              parsedArgs.appDataDir);
              ......
              // in child
              IoUtils.closeQuietly(serverPipeFd);
              serverPipeFd = null;
              //子进程处理
              handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
              // should never get here, the child is expected to either
              // throw ZygoteInit.MethodAndArgsCaller or exec().
              return true;
              ......
        }

```

ZygoteConnection.handleChildProc:
```java
        private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {

              if (parsedArgs.runtimeInit) {
                  if (parsedArgs.invokeWith != null) {
                    ......
                  } else {
                    //根据SS发来的参数设置新进程的一些属性
                    RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
                  }
              } else {
                .......
              }
              ......
        }
```

RuntimeInit.zygoteInit:
```java
        public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
                  throws ZygoteInit.MethodAndArgsCaller {
              if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
              //重定向Log输出
              redirectLogStreams();

              commonInit();
              //native函数，最终会调用AppRuntime.onZygoteInit,在那里建立了和Binder的关系
              nativeZygoteInit();

              applicationInit(targetSdkVersion, argv, classLoader);
        }
```

RuntimeInit.applicationInit:
```java
        private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
                    throws ZygoteInit.MethodAndArgsCaller {
                ......
                //最终还是会调用invokeStaticMain函数
                // Remaining arguments are passed to the start class's static main
                invokeStaticMain(args.startClass, args.startArgs, classLoader);
        }
```
