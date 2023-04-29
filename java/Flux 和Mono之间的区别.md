# 1.概述
在本教程中将了解Reactor Core库的Flux和Mono之间的区别。

# 2.什么是Mono？
Mono是一种特殊类型的Publisher。Mono对象表示单个或空值。这意味着它最多只能为onNext() 请求发出一个值，然后以onComplete()信号终止。如果失败，它只会发出一个onError()信号。

看一个带有完成信号的Mono示例：
```
@Test
public void givenMonoPublisher_whenSubscribeThenReturnSingleValue() {
    Mono<String> helloMono = Mono.just("Hello");
    StepVerifier.create(helloMono)
      .expectNext("Hello")
      .expectComplete()
      .verify();
}
```
在这里可以看到，当helloMono被订阅时，它只发出一个值，然后发送完成信号。

# 3.什么是Flux？
Flux是一个标准的Publisher，代表 0 到 N 个异步序列值。这意味着它可以发出 0 到多个值，对于onNext()请求可能是无限值，然后以完成或错误信号终止。

让我们看一个带有完成信号的Flux示例：
```
@Test
public void givenFluxPublisher_whenSubscribedThenReturnMultipleValues() {
    Flux<String> stringFlux = Flux.just("Hello", "Baeldung");
    StepVerifier.create(stringFlux)
      .expectNext("Hello")
      .expectNext("Baeldung")
      .expectComplete()
      .verify();
}
```
现在，看一个带有错误信号的Flux示例：
```
@Test
public void givenFluxPublisher_whenSubscribeThenReturnMultipleValuesWithError() {
    Flux<String> stringFlux = Flux.just("Hello", "Baeldung", "Error")
      .map(str -> {
          if (str.equals("Error"))
              throw new RuntimeException("Throwing Error");
          return str;
      });
    StepVerifier.create(stringFlux)
      .expectNext("Hello")
      .expectNext("Baeldung")
      .expectError()
      .verify();
}
```
在这里可以看到，在从 Flux 获取两个值后，我们得到了一个错误。

# 4.Mono Vs Flux
Mono和Flux都是Publisher接口的实现。简单来说，我们可以说，当我们在做计算或向数据库或外部服务发出请求，并期望最多一个结果时，我们应该使用Mono。

当期望从我们的计算、数据库或外部服务调用中获得多个结果时，应该使用Flux。

Mono有点类似于 Java 中的Optional类，因为它包含 0 或 1 个值；而Flux与List更相似，因为它可以有 N 个值。

# 5.结论
在本文中了解了Mono和Flux之间的区别。

与往常一样，示例的完整源代码可在 GitHub 上获得。

```
https://github.com/eugenp/tutorials/tree/master/reactor-core
```