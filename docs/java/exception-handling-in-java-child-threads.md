---
title: Java子线程中的异常处理（通用）
---

在普通的单线程程序中，捕获异常只需要通过try ... catch ... finally ...代码块就可以了。那么，在并发情况下，比如在父线程中启动了子线程，如何正确捕获子线程中的异常，从而进行相应的处理呢？

# 常见错误

也许有人会觉得，很简单嘛，直接在父线程启动子线程的地方try ... catch一把就可以了，其实这是不对的。

# 原因分析

让我们回忆一下Runnable接口的run方法的完整签名，因为没有标识throws语句，所以方法是不会抛出checked异常的。至于RuntimeException这样的unchecked异常，由于新线程由JVM进行调度执行，如果发生了异常，也不会通知到父线程。

```java
public abstract void run()
```

# 解决办法

那么，如何正确处理子线程中的异常呢？楼主想到了3种常用方法，分享给大家。  
前2种方法都是在子线程中处理，第3种方法是在父线程中处理。  
具体用哪一种方法，取决于这个异常是否适合在子线程中处理。例如有些异常更适合由调用方（父线程）处理，那么此时就应当用第3种方法。

## 方法一：子线程中try... catch...

最简单有效的办法，就是在子线程的执行方法中，把可能发生异常的地方，用try ... catch ... 语句包起来。
子线程代码：

```java
public class ChildThread implements Runnable {
    public void run() {
        doSomething1();
        try {
            // 可能发生异常的方法
            exceptionMethod();
        } catch (Exception e) {
            // 处理异常
            System.out.println(String.format("handle exception in child thread. %s", e));
        }
        doSomething2();
    }
}
```

## 方法二：为线程设置“未捕获异常处理器”UncaughtExceptionHandler

为线程设置异常处理器。具体做法可以是以下几种：

（1）Thread.setUncaughtExceptionHandler设置当前线程的异常处理器；
（2）Thread.setDefaultUncaughtExceptionHandler为整个程序设置默认的异常处理器；

如果当前线程有异常处理器（默认没有），则优先使用该UncaughtExceptionHandler类；否则，如果当前线程所属的线程组有异常处理器，则使用线程组的UncaughtExceptionHandler；否则，使用全局默认的DefaultUncaughtExceptionHandler；如果都没有的话，子线程就会退出。

注意：子线程中发生了异常，如果没有任何类来接手处理的话，是会直接退出的，而不会记录任何日志。
所以，如果什么都不做的话，是会出现子线程任务既没执行成功，也没有任何日志提示的“诡异”现象的。

设置当前线程的异常处理器：

```java
public class ChildThread implements Runnable {    
    private static ChildThreadExceptionHandler exceptionHandler;

    static {
        exceptionHandler = new ChildThreadExceptionHandler();
    }

    public void run() {
        Thread.currentThread().setUncaughtExceptionHandler(exceptionHandler);
        System.out.println("do something 1");
        exceptionMethod();
        System.out.println("do something 2");
    }

    public static class ChildThreadExceptionHandler implements Thread.UncaughtExceptionHandler {
        public void uncaughtException(Thread t, Throwable e) {
            System.out.println(String.format("handle exception in child thread. %s", e));
        }
    }
}
```

或者，设置所有线程的默认异常处理器

```java
public class ChildThread implements Runnable {
    private static ChildThreadExceptionHandler exceptionHandler;

    static {
        exceptionHandler = new ChildThreadExceptionHandler();
        Thread.setDefaultUncaughtExceptionHandler(exceptionHandler);
    }

    public void run() {
        System.out.println("do something 1");
        exceptionMethod();
        System.out.println("do something 2");
    }

    private void exceptionMethod() {
        throw new RuntimeException("ChildThread exception");
    }

    public static class ChildThreadExceptionHandler implements Thread.UncaughtExceptionHandler {
        public void uncaughtException(Thread t, Throwable e) {
            System.out.println(String.format("handle exception in child thread. %s", e));
        }
    }
}
```

命令行输出：

```
do something 1
handle exception in child thread. java.lang.RuntimeException: ChildThread exception
```

## 方法三：通过Future的get方法捕获异常（推荐）

使用线程池提交一个能获取到返回信息的方法，也就是ExecutorService.submit(Callable)
在submit之后可以获得一个线程执行结果的Future对象，而如果子线程中发生了异常，通过future.get()获取返回值时，可以捕获到ExecutionException异常，从而知道子线程中发生了异常。

子线程代码：

```java
public class ChildThread implements Callable<String> {
    public String call() throws Exception {
        System.out.println("do something 1");
        exceptionMethod();
        System.out.println("do something 2");
        return "test result";
    }

    private void exceptionMethod() {
        throw new RuntimeException("ChildThread1 exception");
    }
}
```

父线程代码：

```java
public class Main {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(8);
        Future future = executorService.submit(new ChildThread());
        try {
            future.get();
        } catch (InterruptedException e) {
            System.out.println(String.format("handle exception in child thread. %s", e));
        } catch (ExecutionException e) {
            System.out.println(String.format("handle exception in child thread. %s", e));
        } finally {
            if (executorService != null) {
                executorService.shutdown();
            }
        }
    }
}
```

命令行输出：

```
do something 1
handle exception in child thread. java.util.concurrent.ExecutionException: java.lang.RuntimeException: ChildThread1 exception
```

# 总结

以上就是3种通用的Java子线程异常处理方法。  
具体使用哪种，取决于异常是否适合在子线程中处理，楼主更推荐第3种方式，可以方便地在父线程中根据子线程抛出的异常做适当的处理（自己处理，或者忽略，或者继续向上层调用方抛异常）。  
其实楼主还想到了另外几个特定场景下的解决办法，改天再分析，谢谢大家支持~
