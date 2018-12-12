
# Java Actor API for Akka 2.4.x


一个例子:

```java
public class JavaDemoActor extends AbstractActor {

	public PartialFunction receive() {
		return ReceiveBuilder
		        .matchEquals("hello", s -> sender().tell("hai", ActorRef.noSender()))
		        .matchAny(x -> sender().tell("what?", ActorRef.noSender()))
		        .build();
	}
}
```


- `AbstractActor` : 首先是继承了 `AbstractActor`, 这是一个 Java 8 特有的 API , 利用了 Java 8 的 Lambda 表达式的特性. 之前还有一个 API 从 `UntypedActor` 来继承, 不过这个有点老了, 使用这个 API 会得到一个对象, 然后必须使用 `if` 语句对条件进行判断.
- `receive()` 方法 : `AbstractActor` 类中有一个 `receive` 方法, 其子类必须实现这个方法或者是通过构造函数调用这个方法. `receive` 方法的返回类型是 `PartialFunction`, 由于 Java 中没有原生的方法来构造 Scala 的 `PartialFunction`, 故此 Akka 提供了一个抽象的构造方法类 `ReceiveBuilder` 来生成.
- `ReceiveBuilder` : 链式调用的对象, 最后的 `build()` 方法来生成 `PartialFunction` 对象.


### `match()` 方法:  

match 方法有多个, 例如:

`match(class, function)` 


描述了对于任何匹配类型的消息该如何响应.

```java
match(String.class, s-> {
	if (s.equals("Ping")) {
		respondToPing(s);
	}
})
```


`match(class, predicate, function)` :

描述了对于 predicate 条件为真的某些特定类型的消息如何响应.

```java
match(string.class, s -> s.equals("Ping"), s-> respondToPing(s))


```

`matchEquals(object, function)` :

描述了对于任何和第一个参数相等的消息, 应该如何响应.

```java
matchEquals("Ping", s -> respondToPing(s))

```

`matchAny(function)` : 

对所有未匹配的消息, 该如何响应, 通常来说, 最佳实践是返回错误消息, 或者记录到日志.

match 函数从上到下进行模式匹配, 因此要先定义特殊情况, 最后定义一版情况.


