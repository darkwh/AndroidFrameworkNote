# Watchdog

Watchdog最早出现在单片机系统中，由于单片机的工作环境容易受到外界磁场的干扰，导致程序"跑飞"，
造成系统无法正常工作。因此引入了"看门狗",对单片机进行实时监测，针对运行故障做出处理，譬如重启系统.

Android中也设计了一个软件层面的Watchdog，对一些重要的系统服务和线程进行监控，一旦他们出现异常就杀掉system_server进程并重启系统.

##Watchdog分析

###HandlerChecker内部类
```java
public final class HandlerChecker implements Runnable {
  ......
  private final Handler mHandler;//所要检查的线程的handler
  private final String mName;
  private final long mWaitMax;//最大等待时间
  private final ArrayList<Monitor> mMonitors = new ArrayList<Monitor>();//监控对象集合
  private boolean mCompleted;//是否完成检测标识位
  private Monitor mCurrentMonitor;//当前的监控对象
  private long mStartTime;//开始检查的时间
  ......

  /**
   * 添加监控对象到集合中
   */
  public void addMonitor(Monitor monitor) {
    mMonitors.add(monitor);
  }

  /**
   * 检查前的调度
   */
  public void scheduleCheckLocked() {
    //如果没有Monitors对象且当前Handler所在的线程处于Idle状态,则设置mCompleted标识位位true,并return
    if (mMonitors.size() == 0 && mHandler.getLooper().isIdling()) {
      mCompleted = true;
      return;
    }
    //检查mCompleted标识位,如果为false,则代表已经处于检查中，直接return
    if (!mCompleted) {
      // we already have a check in flight, so no need
      return;
    }
    //标识位设为false
    mCompleted = false;
    //当前的监控对象设为null
    mCurrentMonitor = null;
    //设置开始检查的时间
    mStartTime = SystemClock.uptimeMillis();
    //将自己推送掉消息队列的最前面，优先执行，执行自己的run函数
    mHandler.postAtFrontOfQueue(this);
  }

  @Override
  public void run() {
    final int size = mMonitors.size();
    for (int i = 0 ; i < size ; i++) {
      synchronized (Watchdog.this) {
        mCurrentMonitor = mMonitors.get(i);
      }
      mCurrentMonitor.monitor();
    }
    synchronized (Watchdog.this) {
      mCompleted = true;
      mCurrentMonitor = null;
    }
  }
}
```
