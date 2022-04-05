### 0. 小红被重用了

有一天老板对小红说：你的勤勤恳恳的工作，公司都看在眼里的。放心，公司不会亏待像你这样爱工作的好同志的。所以决定交给你一份重要任务，以后每天下班的时候把销售统计报表发一份到我的邮箱里。

小红：excuse me?...

一脸诧异，然后开心的接下了任务。

作为一名老资历的搬砖大师，什么大风大浪没见过。然后打开了Google，搜索怎么每天自动发报表给老板？懂小红心思的搜索引擎，便在第一条呈现了定时任务。

接下来让我们看看给小红呈现的自动大法长什么样。

### 1. Java对定时任务的支持

Java提供了多种方式来实现定时任务，常用的有如下两种：

* java.util.Timer

  使用Timer进行调度的定时任务，需要为`java.util.TimerTask`类型。为单线程进行任务调度。

* java.util.concurrent.ScheduledExecutorService
  定时任务调度的并发支持，用于有多个任务需要同一时段调用。普通的`Runnable`和`Callable`都能用于ScheduledExecutorService定时任务调度。
  
由于两种方式对于定时任务调度方式如出一辙，后面我们主要以Timer为例说明不同场景的调度实现。
  
### 2. 定时任务调度类型

为了后面故事的顺利发展，我们先定义一个可执行的任务用于调度：

```java
 public class Task extends TimerTask {
        public Task() {
            System.out.println(
                "task initialized at " + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        }

        @Override
        public void run() {
            System.out.println(
                "task executed at " + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                //ignore
            }
        }
    }
```

#### 2.1 延后指定间隔执行一次

比如延后两秒执行：

```java
Timer timer = new Timer();
timer.schedule(new Task(), 2000);
```

启动后运行结果：

```java
task initialized at 2019-06-08 17:37:12
task executed at 2019-06-08 17:37:14
```

可以看到程序启动后，推迟了2s才真正执行任务。

#### 2.2 指定时间执行一次

比如在当天的晚上18点执行一次：

```java
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, 18);
calendar.set(Calendar.MINUTE, 0);
calendar.set(Calendar.SECOND, 0);
Timer timer = new Timer();
timer.schedule(new Task(), calendar.getTime());
```

如果在18点之前启动，输出：

```java
task initialized at 2019-06-08 17:58:39
task executed at 2019-06-08 18:00:00
```

如果在18点之后启动，输出：

```java
task initialized at 2019-06-08 18:16:51
task executed at 2019-06-08 18:16:51
```

可以看到，如果指定的时间点为未来的某个时间，则会按照预期进行执行。否则，启动就会立即执行任务。

#### 2.3 延迟指定时间间隔的周期性任务

Timer允许指定首次执行的延迟，以及后续每次执行的周期间隔。
比如初始化后延迟5秒执行任务，以后每隔2秒执行一次。

```java
Timer timer = new Timer();
timer.schedule(new Task(), 5000, 2000);
```

输出：

```java
task initialized at 2019-06-08 18:58:13
task executed at 2019-06-08 18:58:18
task executed at 2019-06-08 18:58:20
task executed at 2019-06-08 18:58:22
...
```

可以看到第一次执行在初始化后的第5s才执行，后面每隔2s就会执行一次。虽然任务本身需要1s的执行时间。

如果我们将TimerTask的休眠时间调整为比间隔更大的时间，比如3s。定时任务还会如期的每隔2s执行一次吗？
如下为调整休眠时间为3s的输出:

```java
task initialized at 2019-06-08 18:59:51
task executed at 2019-06-08 18:59:56
task executed at 2019-06-08 18:59:59
task executed at 2019-06-08 19:00:02
task executed at 2019-06-08 19:00:05
...
```

可以看到，后续的定时任务，虽然指定每隔2s执行一次，但实际每3s执行了一次。不过一想，这也合理啊。Timer为单线程调度定时任务，它会竭尽所能地让任务在指定时间执行。但当有任务比较耗时，一直忙着呢，对于并没有三头六臂的Timer，肯定需要忙完手上的活，才有时间调度后续的任务。

#### 2.4 指定第一次执行的时间，后续每隔一定间隔执行

这和2.3类似，只不过指定第一次执行的时间为绝对时间，而不是延迟一个相对时间。
比如从当天19:01开始，每隔2s执行一次。

```java
Timer timer = new Timer();
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, 19);
calendar.set(Calendar.MINUTE, 1);
calendar.set(Calendar.SECOND, 0);

timer.scheduleAtFixedRate(new Task(), calendar.getTime(), 2000);
```

输出：

```java
task initialized at 2019-06-08 19:00:44
task executed at 2019-06-08 19:01:00
task executed at 2019-06-08 19:01:02
task executed at 2019-06-08 19:01:04
...
```

如果在19:01之后执行，则输出如下：

```java
task initialized at 2019-06-08 19:23:28
task executed at 2019-06-08 19:23:28
task executed at 2019-06-08 19:23:30
task executed at 2019-06-08 19:23:32
...
```

我们看到，当指定的绝对时间在当前时间之前，初始化完成后会立即执行。
并且，Timer会从当前的时间开始推算下一次需要执行的时间，在这里，第二次执行时间为`2019-06-08 19:23:30`。

#### 2.5 固定频率，延迟指定时间间隔的周期性任务

Timer允许指定首次执行的延迟，以及后续每次执行的周期间隔。
比如初始化后延迟5秒执行任务，以后每隔2秒执行一次。

```java
Timer timer = new Timer();
timer.scheduleAtFixedRate(new Task(), 5000, 2000);
```

输出：

```java
task initialized at 2019-06-08 18:32:41
task executed at 2019-06-08 18:32:46
task executed at 2019-06-08 18:32:48
task executed at 2019-06-08 18:32:50
...
```

可以看到第一次执行在初始化后的第5s才执行，后面每隔2s就会执行一次。虽然任务本身需要1s的执行时间。

如果我们将TimerTask的休眠时间调整为比间隔更大的时间，比如3s。定时任务还会如期的每隔2s执行一次吗？
如下为调整休眠时间为3s的输出:

```java
task initialized at 2019-06-08 18:34:50
task executed at 2019-06-08 18:34:55
task executed at 2019-06-08 18:34:58
task executed at 2019-06-08 18:35:01
task executed at 2019-06-08 18:35:04
...
```

可以看到，后续的定时任务，虽然指定每隔2s执行一次，但实际每3s执行了一次。和2.3中一致。

#### 2.6 固定频率，指定第一次执行的时间，后续每隔一定间隔执行

这和2.5类似，只不过指定第一次执行的时间为绝对时间，而不是延迟一个相对时间。
比如从当天18点45开始，每隔2s执行一次。

```java
Timer timer = new Timer();
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, 18);
calendar.set(Calendar.MINUTE, 45);
calendar.set(Calendar.SECOND, 0);

timer.scheduleAtFixedRate(new Task(), calendar.getTime(), 2000);
```

输出：

```java
task initialized at 2019-06-08 18:44:49
task executed at 2019-06-08 18:45:00
task executed at 2019-06-08 18:45:02
task executed at 2019-06-08 18:45:04
...
```

如果在18点45之后执行，则输出如下：

```java
task initialized at 2019-06-08 18:48:39
task executed at 2019-06-08 18:48:39
task executed at 2019-06-08 18:48:40
task executed at 2019-06-08 18:48:41
task executed at 2019-06-08 18:48:42
...
```

我们看到，当指定的绝对时间在当前时间之前，初始化完成后会立即执行。
并且，Timer会从**指定的时间**开始推算下一次需要执行的时间，在这里，第二次执行时间为`2019-06-08 18:45:02`。而这个时间已经比当前时间小了，所以会立即执行。

所以，虽然我们指定每隔2s执行一次，但由于Timer会在补偿执行由于指定时间早于第一次开始执行的时间之间的任务。所以会一直连续不断的执行任务（每次任务耗时1s），直到下次执行时间为未来某个时间为止。

#### 2.7 小结

固定频率和其他调度方式，主要差别在于如何计算下次执行时间

* 固定频率： 用上次任务执行时间计算下次执行时间
* 其他周期性调度： 用当前时间计算下次执行时间

### 3. 小红的自动大法

Java为小红提供了行行种种的调度方式，小红决定写个定时任务，这样就不用每天自己亲自操作了。

小红需要每天下班的时候（18:00）准时发送销售统计邮件给老板，综合上述调度方式，固定频率方式更适合。伪代码如下：

```java
Timer timer = new Timer();
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, 18);
calendar.set(Calendar.MINUTE, 0);
calendar.set(Calendar.SECOND, 0);

timer.scheduleAtFixedRate(()->sendEmailToBoss(), calendar.getTime(), 24*3600*1000);//delay one day
```

小红当天就开启了这个服务，从这天开始，就会每天18:00整准时给老板报告销售统计情况。小红脸上浮出满意的微笑，离走上人生巅峰，又近了一大截。