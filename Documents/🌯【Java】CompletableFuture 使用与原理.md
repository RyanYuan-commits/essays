### 1 背景与定义
#### 1.1 CompletableFuture 解决的问题
`CompletableFuture` 是 Java 8 引入的，在 Java8 之前我们一般通过 `Future` 实现异步。
- `Future` 用于表示异步计算的结果，只能通过阻塞或者轮询的方式获取结果，而且不支持设置回调方法，Java 8 之前若要设置回调一般会使用 `guava` 的 `ListenableFuture`，回调的引入又会导致臭名昭著的回调地狱（下面的例子会通过ListenableFuture的使用来具体进行展示）。
- `CompletableFuture` 对 `Future` 进行了扩展，可以通过设置回调的方式处理计算结果，同时也支持组合操作，支持进一步的编排，同时一定程度解决了回调地狱的问题。
下面将举例来说明，我们通过 `ListenableFuture`、`CompletableFuture` 来实现异步的差异。假设有三个操作 step1、step2、step3 存在依赖关系，其中step3 的执行依赖 step1 和 step2 的结果。
`Future(ListenableFuture)` 的实现（回调地狱）如下：
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
ListeningExecutorService guavaExecutor = MoreExecutors.listeningDecorator(executor);
ListenableFuture<String> future1 = guavaExecutor.submit(() -> {
    //step 1
    System.out.println("执行step 1");
    return "step1 result";
});
ListenableFuture<String> future2 = guavaExecutor.submit(() -> {
    //step 2
    System.out.println("执行step 2");
    return "step2 result";
});
ListenableFuture<List<String>> future1And2 = Futures.allAsList(future1, future2);
Futures.addCallback(future1And2, new FutureCallback<List<String>>() {
    @Override
    public void onSuccess(List<String> result) {
        System.out.println(result);
        ListenableFuture<String> future3 = guavaExecutor.submit(() -> {
            System.out.println("执行step 3");
            return "step3 result";
        });
        Futures.addCallback(future3, new FutureCallback<String>() {
            @Override
            public void onSuccess(String result) {
                System.out.println(result);
            }        
            @Override
            public void onFailure(Throwable t) {
            }
        }, guavaExecutor);
    }

    @Override
    public void onFailure(Throwable t) {}
}, guavaExecutor);
```
`CompletableFeture` 的实现方式
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
    System.out.println("执行step 1");
    return "step1 result";
}, executor);
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
    System.out.println("执行step 2");
    return "step2 result";
});
cf1.thenCombine(cf2, (result1, result2) -> {
    System.out.println(result1 + " , " + result2);
    System.out.println("执行step 3");
    return "step3 result";
}).thenAccept(result3 -> System.out.println(result3));
```
显然，CompletableFuture的实现更为简洁，可读性更好。
#### 1.2 CompletableFuture 的定义
![[CompletableFuture 的定义.png|center|800]]
`CompletableFuture` 实现了两个接口（如上图所示）：`Future`、`CompletionStage`。`Future` 表示异步计算的结果，`CompletionStage` 用于表示异步执行过程中的一个步骤（Stage），这个步骤可能是由另外一个 `CompletionStage` 触发的，随着当前步骤的完成，也可能会触发其他一系列 `CompletionStage` 的执行。从而我们可以根据实际业务对这些步骤进行多样化的编排组合，`CompletionStage` 接口正是定义了这样的能力，我们可以通过其提供的 `thenAppy`、`thenCompose` 等函数式编程方法来组合编排这些步骤。
### 2 CompletableFuture的使用
下面我们通过一个例子来讲解 CompletableFuture 如何使用，使用 CompletableFuture 也是构建依赖树的过程。一个 CompletableFuture 的完成会触发另外一系列依赖它的 CompletableFuture 的执行：
![[CompletableFuture的使用.png|center]]
如上图所示，这里描绘的是一个业务接口的流程，其中包括 5 个步骤，并描绘了这些步骤之间的依赖关系，每个步骤可以是一次 RPC 调用、一次数据库操作或者是一次本地方法调用等，在使用 CompletableFuture 进行异步化编程时，图中的每个步骤都会产生一个 CompletableFuture 对象，最终结果也会用一个CompletableFuture 来进行表示。
根据 CompletableFuture 依赖数量，可以分为以下几类：零依赖、一元依赖、二元依赖和多元依赖。
#### 2.1 零依赖：CompletableFuture的创建
我们先看下如何不依赖其他 `CompletableFuture` 来创建新的 `CompletableFuture`：
![[零依赖：CompletableFuture的创建.png|center]]
如上图红色链路所示，接口接收到请求后，首先发起两个异步调用CF1、CF2，主要有三种方式：
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
// 1、使用runAsync或supplyAsync发起异步调用
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
  return "result1";
}, executor);

// 2、CompletableFuture.completedFuture() 直接创建一个已完成状态的 CompletableFuture
	CompletableFuture<String> cf2 = CompletableFuture.completedFuture("result2");

// 3、先初始化一个未完成的 CompletableFuture，然后通过 complete()、completeExceptionally()，完成该 CompletableFuture
CompletableFuture<String> cf = new CompletableFuture<>();
cf.complete("success");
```
第三种方式的一个典型使用场景，就是将回调方法转为 `CompletableFuture`，然后再依赖 `CompletableFuture` 的能力进行调用编排，示例如下：
```java
@FunctionalInterface
public interface ThriftAsyncCall {
    void invoke() throws TException;
}
 /**
  * 该方法为美团内部rpc注册监听的封装，可以作为其他实现的参照
  * OctoThriftCallback 为thrift回调方法
  * ThriftAsyncCall 为自定义函数，用来表示一次thrift调用（定义如上）
  */
  public static <T> CompletableFuture<T> toCompletableFuture(final OctoThriftCallback<?,T> callback , 
			  ThriftAsyncCall thriftCall) {
   //新建一个未完成的 CompletableFuture
   CompletableFuture<T> resultFuture = new CompletableFuture<>();
   //监听回调的完成，并且与 CompletableFuture 同步状态
   callback.addObserver(new OctoObserver<T>() {
       @Override
       public void onSuccess(T t) {
           resultFuture.complete(t);
       }
       @Override
       public void onFailure(Throwable throwable) {
           resultFuture.completeExceptionally(throwable);
       }
   });
   if (thriftCall != null) {
       try {
           thriftCall.invoke();
       } catch (TException e) {
           resultFuture.completeExceptionally(e);
       }
   }
   return resultFuture;
  }

```
#### 2.2 一元依赖：依赖一个 CF
![[一元依赖：依赖一个 CF.png|center]]
如上图红色链路所示，CF3，CF5 分别依赖于 CF1 和 CF2，这种对于单个 CompletableFuture 的依赖可以通过 thenApply、thenAccept、thenCompose 等方法来实现，代码如下所示：
```java
CompletableFuture<String> cf3 = cf1.thenApply(result1 -> {
  //result1为CF1的结果
  //......
  return "result3";
});
CompletableFuture<String> cf5 = cf2.thenApply(result2 -> {
  //result2为CF2的结果
  //......
  return "result5";
});
```
#### 2.3 二元依赖：依赖两个 CF
![[二元依赖：依赖两个 CF.png|center]]
如上图红色链路所示，CF4同时依赖于两个CF1和CF2，这种二元依赖可以通过thenCombine等回调来实现，如下代码所示：
```java
CompletableFuture<String> cf4 = cf1.thenCombine(cf2, (result1, result2) -> {
  //result1和result2分别为cf1和cf2的结果
  return "result4";
});
```
#### 2.4 多元依赖：依赖多个CF
![[多元依赖：依赖多个CF.png|center]]
如上图红色链路所示，整个流程的结束依赖于三个步骤 CF3、CF4、CF5，这种多元依赖可以通过 `allOf` 或 `anyOf` 方法来实现，区别是当需要多个依赖全部完成时使用 `allOf`，当多个依赖中的任意一个完成即可时使用 `anyOf`，如下代码所示：
```java
CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf3, cf4, cf5);
CompletableFuture<String> result = cf6.thenApply(v -> {
  //这里的join并不会阻塞，因为传给thenApply的函数是在CF3、CF4、CF5全部完成时，才会执行 。
  result3 = cf3.join();
  result4 = cf4.join();
  result5 = cf5.join();
  //根据result3、result4、result5组装最终result;
  return "result";
});
```
### 3 CompletableFuture 原理
CompletableFuture中包含两个字段：`result` 和 `stack`。`result` 用于存储当前 CF 的结果，`stack` 表示当前CF完成后需要触发的依赖动作，去触发依赖它的 CF 的计算，依赖动作可以有多个，以栈的形式存储，`stack` 表示栈顶元素。
![[CompletableFuture 组成.png|center]]
这种方式类似 “观察者模式”，依赖动作都封装在一个单独 `Completion` 子类中。下面是 `Completion` 类关系结构图。`CompletableFuture` 中的每个方法都对应了图中的一个 `Completion` 的子类，`Completion` 本身是 **观察者** 的基类。
- `UniCompletion` 继承了 `Completion`，是一元依赖的基类，例如 `thenApply` 的实现类 `UniApply` 就继承自 `UniCompletion`。
- `BiCompletion` 继承了 `UniCompletion`，是二元依赖的基类，同时也是多元依赖的基类。例如 `thenCombine` 的实现类 `BiRelay` 就继承自 `BiCompletion`。
![[Completion 类关系结构图.png]]
