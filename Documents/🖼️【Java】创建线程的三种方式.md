### 1 继承 Thread 类
通过继承 Thread 类，并重写它的 run 方法，我们就可以创建一个线程。​
```java
// 定义一个继承自 Thread 类的子类
class MyThread extends Thread {
    @Override
    public void run() {
        // 线程要执行的任务
        System.out.println("线程 " + Thread.currentThread().getName() + " 正在运行");
    }
}

public class ThreadCreationByExtending {
    public static void main(String[] args) {
        // 创建 MyThread 类的实例
        MyThread myThread = new MyThread();
        // 启动线程
        myThread.start();
        System.out.println("主线程 " + Thread.currentThread().getName() + " 正在运行");
    }
}
```
### 2 实现 Runnable 接口
通过实现 Runnable ，并实现 run 方法，也可以创建一个线程。
```java
// 定义一个实现 Runnable 接口的类
class MyRunnable implements Runnable {
    @Override
    public void run() {
        // 线程要执行的任务
        System.out.println("线程 " + Thread.currentThread().getName() + " 正在运行");
    }
}

public class ThreadCreationByImplementing {
    public static void main(String[] args) {
        // 创建 MyRunnable 类的实例
        MyRunnable myRunnable = new MyRunnable();
        // 创建 Thread 类的实例，并将 MyRunnable 实例作为参数传递
        Thread thread = new Thread(myRunnable);
        // 启动线程
        thread.start();
        System.out.println("主线程 " + Thread.currentThread().getName() + " 正在运行");
    }
}
```
### 3 Callable + Future
`Callable` 接口和 `Runnable` 接口类似，但是 `Callable` 接口的 `call()` 方法可以有返回值，并且可以抛出异常。可以使用 `ExecutorService` 来提交 `Callable` 任务，并通过 `Future` 对象获取任务的执行结果。
```java
import java.util.concurrent.*;

// 定义一个实现 Callable 接口的类
class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        // 线程要执行的任务
        System.out.println("线程 " + Thread.currentThread().getName() + " 正在运行");
        return 42;
    }
}

public class ThreadCreationByCallable {
    public static void main(String[] args) {
        // 创建线程池
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        // 创建 MyCallable 类的实例
        MyCallable myCallable = new MyCallable();
        // 提交 Callable 任务，并获取 Future 对象
        Future<Integer> future = executorService.submit(myCallable);
        try {
            // 获取任务的执行结果
            Integer result = future.get();
            System.out.println("任务的执行结果是: " + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        // 关闭线程池
        executorService.shutdown();
    }
}
```