
# CompleteableFuture

书上说: 下一代网络应用都采用混搭方式 (mash-up): 它会使用来自多个来源的内容, 将这些内容聚合在一起...

如果你的目标是在同一个 CPU 上执行几个松耦合的任务, 充分利用 CPU 的核心, 让其足够忙碌, 从而最大化程序的吞吐量, 那么你其实真正想做的是避免因为等待远程服务的返回或者对数据库的查询,  而阻塞线程的执行, 浪费宝贵的计算资源, 因为这种等待的时间可能相当长.

`Future` 接口,尤其是他的新版实现 `CompletableFuture` 是处理这种情况的利器.


## `Future` 接口

Future 接口在 Java5 中被引入, 设计初衷是对将来某个时刻会发生的结果进行建模. 他建模了一种异步计算, 返回一个执行运算结果的引用, 当运算结束后, 这个引用被返回给调用方.

在 Future 中触发那些潜在耗时的操作把调用线程解放出来, 让它能继续执行其他有价值的工作, 不需要呆呆第等耗时的操作完成.

要使用 Future, 通常只需要将耗时的操作封装到一个 Callable 对象中, 再将它提交给 ExecutorService 即可.


```java

ExecutorService executor = Executors.newCachedThreadPool();
// 想 ExecutorService 提交一个 Callable 对象
Future<Double> future = executor.submit(new Callable<Double>() {

	public Double call() {
		// 以异步方式在新的线程中执行耗时的操作.
		return doSomeLongComputation();
	}
});

// 异步操作进行的同时, 可以做其他事情.
doSumethingElse(); 

// 获取异步操作的结, 最多等待 1 秒钟
try {
	Double result = future.get(1, TimeUnit.SECONDS);
} catch (ExecutionException ee) {
	// 计算抛出一个异常
} catch (InterruptedException ie) {
	// 当前线程在等待过程中被中断
} catch (TimeoutException te) {
	// 在 Future 对象完成之前超时
}
```


### Future 接口的局限性

`Future` 接口中提供了方法来检测异步计算是否已经结束( `isDone()`), 等待异步操作结束, 以及获取计算结果. 但是还是有些场景:

- 将两个异步计算合并为一个 -- 这两个异步计算之间相互独立, 同时第二个又依赖第一个个的结果.
- 等待 `Future` 集合中的所有任务都完成.
- 仅仅等待 `Future` 集合中最快结束的任务完成(有可能因为他们试图通过不同的凡是计算同一个值), 并返回结果.
- 通过编程方式完成一个 `Future` 任务的执行(即手工设定异步操作结果的方式).
- 应对 `Future` 的完成事件(即当 `Future` 的完成事件发生时会收到通知, 并能使用 `Future`计算的结果进行下一步操作, 不只是简单地等待操作的结果)


## 使用 `CompletableFuture` 构建异步应用

```java

public Future<Double> getPriceAsync(String product) {
	// 创建 CompletableFuture 对象,
	// 它包含计算的结果
	CompletableFuture<Double> futurePrice = new CompletableFuture<>();
	new Thread(() -> {
		try {
			// 在另一个线程中异步方式执行计算
			double price = calculatePrice(product);
			// 计算出结果时, 设置 Future 的返回值
			futurePrice.complete(price);
		} catch (Exception ex) {
			// 出现异常就抛出去
			futurePrice.completeExceptionally(ex);
		}
		
	}).start();
	// 无需等待, 直接返回 Future 对象.
	return futurePrice;
}

/*
 * 调用的方法
 */
public void run() {
	Shop shop = new Shop();
	long start = System.nanoTime();
	Future<Double> futurePrice = shop.getPriceAsync("my products");
	long invocationTime = ((System.nanoTime() - start) / 1_000_000);
	doSimethingElse();
	try {
		double price = futurePrice.get();
		System.out.printf("Price is %.2f%n", price);
	} catch (Exception e) {
		throw new RuntimeException(e);
	}
	long retrievalTime = ((System.nanoTime() - start) / 1_000_000);

}
```

### 使用工厂方法 `supplyAsync` 创建 `CompletableFuture`

CompletableFuture 类自带了大量精巧的工厂方法, 可以更容易地完成真个流程.

`supplyAsync` 方法接受一个生产者(`Supplier`)作为参数. 返回一个额 `CompletableFuture` 对象, 该对象完成一部执行后悔读取生产者方法的返回值. 生产者会交友 `ForkJoinPool` 池中的某个执行线程 (Executor) 运行.


```java
// 替换上面的额方法 ...
public Future<Double> getPriceAsync(String product) {
	return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
```


### 使用并行流

```java
public List<String> findPrices(String product) {
	return shops.parallelStream()
	        .map(shop -> String.format("%s price is %.2f"
	        	, shop.getName()
	        	, shop.getPrices(product)))
	        .collect(toList());
}

```

### 使用 `CompletableFuture` 发起异步请求


```java
/*
 * 使用两个不同的 Stream 流水线
 */
public List<String> findPrices(String product) {
	List<CompletableFuture<String>> priceFutures = shops
		.stream()
    	.map(shop -> CompletableFuture.supplyAsync(() -> String.format("%s price is %.2f", shop.getname(), shop.getPrice(product))))
    	.collect(toList());

    return priceFutures.stream()
    	.map(CompletableFuture::join)
    	.collect(toList());
}



```


### 使用定制的执行器


```java
private final Executor executor = 
        Executors.newFixedThreadPool(Math.min(shops.size(), 100), new ThreadFactory() {
        	public Thread newThread(Runtable r) {
        		Thread t = new Thread(r);
        		// 使用守护线程
        		// 这种凡是不会阻止程序的关停.
        		t.setDaemon(true);
        		return t;
        	}
        })

CompletableFuture.supplyAsync(() -> shop.getName() + " price is "+ shop.getPrice(product),
	executor);
```


## 使用 Stream 还是 CompletableFuture ?

- 如果是计算密集型的操作, 并且没有 I/O, 那么推荐使用 Stream 接口.
- 反之, 使用 CompletableFuture .


## 构造同步和异步操作


