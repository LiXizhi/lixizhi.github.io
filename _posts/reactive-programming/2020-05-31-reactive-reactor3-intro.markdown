---
layout: post
title:  "Reactive Programming & Reactor3 #01 - Reactive Programming과 Reactor3 소개"
date:   2020-05-30 15:14:55
categories: reactive-programming
comments: true
---
* content
{:toc}

최근에 `Reactive`에 관한 기술을 다시 공부할 필요성을 느끼게 되었다. `Reactive`의 이점을 제대로 누리기 위해서는 어플리케이션의 로직 및 I/O가 발생하는 부분에서 `Async & Nonblocking`이란 조건을 만족해야 한다. 기존에는 Reactive하게 개발한다고 하더라도 JDBC가 Blocking하게 동작하기 때문에 Reactive의 이점을 누리기에는 한계가 있었다.
하지만 최근 들어 이러한 한계점을 극복하는 `R2DBC(Reactive Relational Database Connectivity)` 등의 기술이 발전하고 있기 때문에 `Reactive Programming`에 관해 다시 살펴볼 좋은 타이밍이라고 생각하고 이에 관한 포스트를 작성하기로 결심하였다.

`Spring Webflux`의 베이스가 되는 `Reactor3`의 레퍼런스 문서를 참고하면서 앞으로 포스트를 작성할 예정이다.

<br>

### 1. Reactive Programming 소개
레퍼런스에는 Reactor3가 구현한 `Reactive Programming`을 다음과 같이 소개하고 있다.
<br/><br/>
*Reactive Programming은 Data Stream 및 변경에 대한 전파와 관련된 Async Programming이다.*
<br/><br/>

무슨 말인고 하니 Data Stream은 `Iterable & Iterator` 패턴, 변경에 대한 전파는 `Observer` 패턴에 비유해 보자.
`Iterable & Iterator` 패턴에서 
- `Iterable`은 리스트나 배열의 특정 원소 값들에 접근할 수 있는 `Iterator`를 반환하는 책임이 있다.
- `Iterator`는 원소 값에 대한 접근을 가능하도록 하는 책임이 있다.

즉, 데이터 관점에서 보면 `Iterable`은 처리할 데이터들의 묶음을 반환하는 역할을 맡고, `Iterator`는 그 데이터에 접근하기 위한 역할을 제공한다.

Reactor3에서는 `Publisher-Subscriber` 모델을 사용한다. `Event-Driven Architecture`와 유사한데, `Publisher`는 데이터를 발행한 후 `Subscriber`에 던져주고 `Subscriber`는 그 데이터에 접근해 처리하게 된다.
이러한 관점에서 보면 `Iterable`은 `Publisher`, `Iterator`는 `Subscriber`에 비유가 가능하다.

`Observer` 패턴은 대상 오브젝트의 상태가 변하거나 이벤트가 발생했을 경우 이를 감지해 적절한 로직을 수행할 수 있도록 하는 디자인 패턴이다. Reactor3에서는 `Publisher`가 `Subscriber`에게 데이터를 전달해주면 `Subscriber`는 이를 감지하여 그 데이터를 바탕으로 적절한 로직을 수행한다. (아직은 설명하지 않았지만 Reactive의 주요 특징인 Backpressure도 이와 관련이 있다.)

그리고 `Reactive`는 명령형이 아닌 선언적인 특징을 가지고 있다. 예를들면 `Iterable & Iterator` 패턴에서는 `next()` 메소드를 통해 데이터에 대한 접근을 개발자가 원하는 시기에 수행하는데 반해, Reactor3의 `Publisher & Subscriber`은 자바8의 `Stream`처럼 데이터 접근 메소드를 직접 호출하지 않고 데이터 처리에 대한 로직을 선언만 한다.

`Async Programming`에 대한 설명을 하지 않았는데, 이는 간단하게 위에서 말한것을 모두 `Async`하게 처리하는 것이 Reactor3가 구현한 `Reactive Programming`이라고 생각하면 된다.
<br/><br/>

### 2. Reactive Programming을 구현한 Reactor3의 특징 및 장점
위에서 Reactor3가 구현한 Reactive Programming의 기본 개념을 알아보았지만, 이러한 패러다임이 우리에게 주는 이점이 무엇이지? 라는 생각이 들것이다. 궁금증을 해소하기 위해  Reactor3에 반영된 Reactive Programming의 특징 및 장점에 대해서 알아보겠다.

#### 1. Asynchronous & Non-Blocking 이다.
`Non-Blocking`은 어떠한 로직이 멈추지 않고 계속 수행된다. 라는 정도로 이해하면 되겠다. 비동기적으로 처리하지 않는다라는 뜻인 `Asynchronous`와는 다른 뜻인데 이 차이점에 대해 많은 사람들이 명확하게 차이점을 인지하지 못하고 있어, 간단한 예를들어 설명해보면 아래와 같다.
- Asynchronous인데 Blocking인 경우
```java
        // 비동기로 데이터를 가져온다. (Asynchronous)
        Future<Data> future = executorService.submit(() -> {
            Data data = dataService.getData();
            return data;
        });

        // 데이터를 가져오는것은 다른 스레드에 위임하여 비동기 방식으로 동작하지만,
        // 아래의 비동기 작업이 완료될 때까지 현재 스레드는 blocking 된다. 
        Data data = future.get();

        // future.get()의 동작이 완료되어야 아래의 로직이 수행된다.
        service.processLogic2();
```
`Non-Blocking & Async` 방식으로 동작해 I/O와 어플리케이션 로직에서의 유휴 리소스를 최소화하므로 적은 리소스로도 많은 퍼포먼스를 발휘할 수 있다.

#### 2. Asynchronous & Non-Blocking한 코드를 작성하기 쉽다.
`Asynchronous & Non-Blocking`한 코드를 작성할때, 자바에서는 콜백 패턴을 주로 사용하게 된다. 복잡한 비즈니스 로직에 이를 적용하게 되면 Callback 패턴이 중첩된 `Callback Hell`을 마주할 수 있는데 이는 코드의 가독성과 유지 보수성을 현저히 떨어뜨린다. `CompletableFuture`로 Callback Hell을 어느정도 방지할 수는 있지만, Reactor3는 데이터에 대한 `Asynchronous & Non-Blocking` 처리 과정을 추상화하여 Java8의 Stream과 비슷하게 메소드 체이닝 방식으로 코드를 더 쉽게 작성하게 해준다.

#### 3. 풍부한 Operator의 제공
Java8의 스트림의 `map()`, `filter()` 등 데이터를 가공 및 처리하는 연산자에 대응되는 풍부한 Operator를 제공한다. `Operator`는 변형된 새로운 데이터를 다시 제공한다는 관점에서 중간 `Publisher`가 생겨난다.

#### 4. Subscribe 될 때까지 로직을 수행하지 않는다.
기본적으로 Reactor3를 이용하여 메소드 체이닝 방식으로 `Publisher`가 제공하는 데이터를 처리 및 가공하는 로직을 작성하더라도 `subscribe()` 메소드가 호출되지 않으면 아무런 동작을 하지 않는다. 이는 재사용성 및 다른 `Publisher`간의 합성 기능을 지원하기 위함인것 같다.

#### 5. Backpressure
`Backpressure`는 `Subscriber`가 `Publisher`에게 피드백을 주어 `Publisher`가 어떠한 조치를 취하도록 하는것이다. 예를 들어, `Subscriber`가 30개의 데이터를 처리한 결과 많은 시간이 걸렸다면, `Subscriber`는 `Publisher`에게 앞으로는 30개보다 더 적은 데이터를 주도록 요청을 하는 경우이다. 무한히 데이터가 흐를수 있다라고 가정을 하는 `Data Stream` 관점에서는 중요한 특징인 것 같다.
<br/><br/>

위의 내용들을 토대로 Reactor3가 동작하는 그림을 간단히 묘사하면 아래와 같다.
![reactor4-diagram-01](https://user-images.githubusercontent.com/19832483/83354217-54b62f80-a392-11ea-822b-5b3999944ec9.png)

<br/><br/><br/>
***참고 리소스*** <br/>
***https://projectreactor.io/docs/core/release/reference/*** <br/>
***https://tech.io/playgrounds/929/reactive-programming-with-reactor-3/Intro***