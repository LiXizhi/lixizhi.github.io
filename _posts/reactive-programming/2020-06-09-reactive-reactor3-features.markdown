---
layout: post
title:  "Reactive Programming & Reactor3 #02 - Flux와 Mono의 기본 개념과 내부 동작"
date:   2020-06-17 15:14:55
categories: reactive-programming
comments: true
---
* content
{:toc}
 
Reactor 프로젝트의 메인 아티펙트는 `reactor-core`이다. `rector-core`를 사용하는 개발자들은 `Publisher`를 구현하여 커스텀한 `Publisher`를 만들수 있다.

하지만 `reactor-core`는 풍부한 연산자를 제공하는 미리 정의된 `Flux` 및 `Mono`와 같은 `Publisher`를 제공한다.
- `Flux`는 0..N개의 항목에 대한 비동기 처리 가능한 시퀀스를 나타낸다.
- `Mono`는 0..1개의 항목에 대한 비동기 처리 가능한 결과를 나타낸다.

`Flux`가 `Mono`의 역할을 대신 할 수 있을것 같은데 왜 굳이 타입을 두개로 나누었을까? 기능 제공과 의미론적인 측면에서 나누는 것이 타당하다고 생각했기 때문이다. 예를들어 HTTP request는 오직 한개의 `HttpResponse`를 생성한다. 이 상황에서 `Flux<HttpResponse>`로 표현하는 것 보다는 `Mono<HttpResponse>`로 표현하여 정상적으로 동작했을 경우 한 개의 `HttpResponse`를 반환한다는 의미를 암시하고, 오직 0..1개의 항목에 대한 연산자만 제공할 수 있기 때문이다.

이제 본격적으로 `Flux`와 `Mono`에 대해 자세히 알아보자.

<br/>
## Flux

`Flux<T>`는 0..N개의 방출된 항목에 대한 비동기 시퀀스를 나타내는 `Publisher<T>`이다. 시퀀스에 대해서 어떠한 행위를 취할것인가는 `Subscriber`의 `onNext(T t)`, `onComplete()`, `onError(Throwable t)`를 이용해 작성할 수 있다.
- `onNext(T t)`: 시퀀스의 항목에 대해 어떠한 조치를 취할것인지 행위를 작성한다.
- `onError(Throwable t)`: 시퀀스 처리 도중 예외가 발생했을 경우 어떠한 조치를 취할것인지 행위를 작성한다.
- `onComplete()`: 시퀀스 처리가 완료되었을때의 행위를 작성한다.

`Flux`에서 제공하는 `count`, `map`과 같은 풍부한 중간 연산자들은 각 용도에 맞게 구현된 `Subscriber`가 이미 내장되어 있다. 즉 `Flux`를 사용할 때에는 우리는 최종 결과에 대한 `Subscriber`를 작성하기만 하면 된다는 의미이다.

예를들면 아래와 같다.

``` java
        // "aaa", "bb", "cc", "dd" 의 항목으로 시퀀스를 만들고
        // 대문자로 변환하여 각 항목을 출력하는 로직
        Flux.just("aaa", "bb", "cc", "dd")  // just에 대한 Subscriber는 내장되어 있으므로 작성할 필요가 없다.
            .map(str -> str.toUpperCase())  // map 대한 Subscriber를 내장되어 있으므로 작성할 필요가 없다.
            .subscribe(new Subscriber<String>() {   // 최종 Subscriber만 작성한다. 인자가 없는 subscribe() 메소드를 사용한다면
                                                    // 기본 Subscriber가 적용된다.
                @Override
                public void onSubscribe(Subscription s) {
                    // 항목들 중 최대 5개만 요청한다.
                    s.request(5);
                }

                @Override
                public void onNext(String s) {
                    System.out.println(s);
                }

                @Override
                public void onError(Throwable t) {
                    System.out.println(t);
                }

                @Override
                public void onComplete() {
                    System.out.println("complete");
                }
            });
```

위 코드가 어떻게 동작하는지 자세히 살펴보기 전에 `Publisher`, `Subscriber`, `SubScription`이 어떤 역할을 하고 어떻게 상호작용하는지 살펴보자.

<br/>
### Publisher
`Publisher`는 `Subscriber`에게 데이터를 제공한다. `Publisher` 인터페이스의 정의는 다음과 같이 되어있다.
``` java
public interface Publisher<T> {

    // 이 메소드는 `Publisher`가 `Subscriber`에게 데이터 스트리밍을 시작하도록 하는 역할을 한다.
    public void subscribe(Subscriber<? super T> s);
}
```
 `Publisher`의 구현체는 `Subscriber`를 이용해 데이터를 `Subscriber`에게 전송하도록 `subscribe()` 메소드를 구현해야한다.

<br/>
### Subscription
`Subscription`은 `Subscriber`가 `Publisher`에게 데이터를 요청할때 사용하는 인터페이스이다. 구독하는 `Subscriber`와 1:1 라이프 사이클을 가진다. 즉, 1개의 `Subscriber`는 1개의 `Subscription`을 사용해야만 한다. `SubScription` 인터페이스의 정의는 다음과 같이 되어있다.
``` java
public interface Subscription {
    
    // Publisher에게 데이터를 n개 요청한다.
    public void request(long n);

    
    // Publisher에게 데이터를 요청을 중단하고, 리소스를 정리하도록 요청한다.
    public void cancel();
}
```

<br/>
### Subscriber
`Subscriber`는 `Publisher`에게 데이터를 요청하고 받는 주체이다. 데이터를 요청할때에는 `Subscription`을 통해 데이터를 요청한다. `Subscriber`의 인터페이스는 다음과 같이 정의되어 있다.
``` java
public interface Subscriber<T> {

    // Publisher#subscribe(Subscriber<? super T> s)가 호출이 되면 내부에서
    // 해당 메소드를 호출한다.
    // 보통 해당 메소드에서 인자로 받은 Subscription을 이용해 Subscription#request(long n)을 호출하여
    // 데이터 스트리밍의 시작을 요청한다.
    // Subscription#request(long n)을 호출하지 않으면, 데이터 스트리밍은 시작되지 않는다.
    public void onSubscribe(Subscription s);

    // Subscription#request(long n) 요청에 대한 Publisher의 응답으로 보내진
    // 데이터를 처리하는 로직을 작성하는 메소드이다.
    public void onNext(T t);

    // 데이터 처리 도중 예외가 발생했을때 로직을 작성하는 메소드이다.
    // Failed State로 종료한다.
    public void onError(Throwable t);

    // 데이터가 모두 정상적으로 처리되었을때의 로직을 작성하는 메소드이다.
    // Successful State로 종료한다.
    public void onComplete();
}
```
Subscriber의 구현체는 Publisher#subscribe(Subscriber<? super T> s) 호출시에 인자로 전달되어 onSubscribe(Subscription s)에 대한 호출을 한 번 받는다.
이 메소드에서 Subscription#request(long n)가 호출되지 않는다면, 데이터 스트리밍은 시작되지 않는다.

<br/>
### Flux의 동작
위의 개념을 바탕으로 위에서 작성한 코드가 내부에서 어떻게 동작하는지 그림으로 살펴보자.
![reactor-flux-01](https://user-images.githubusercontent.com/19832483/85020508-688fcd00-b1ab-11ea-828f-d5f1a4b17225.png)

위의 그림을 볼 때에는 reactor3의 내부 코드를 같이 보면서 어떻게 동작하는지 확인해보는걸 추천한다. 그럼 `Publisher`, `Subscription`, `Subscriber`가 어떻게 상호 작용하는지 쉽게 이해가 될 것이다. 또는 필자처럼 직접 코드를 보면서 위와 같이 그림을 그려간다면 이해하는데 가장 효과적일 것이다. 여기서 특이한 점은 `MapSubscriber`가 `Subscriber`, `Subscription` 두 인터페이스를 구현하고 있다는 점이다.

이제 공식 문서에 나와 있는 문서에 있는 그림을 토대로, `Flux`가 전반적으로 어떻게 동작하는지 살펴보자.
![reactor-flux-02](https://user-images.githubusercontent.com/19832483/85021393-bf49d680-b1ac-11ea-8ba0-d186fd9d1594.png)

위에서 보았듯이 데이터 스트리밍을 시작하면 기본적으로 `Flux`는 데이터를 한 개씩 처리한다.(async하게 사용하지 않는다고 가정한다.) 데이터 처리 도중 예외 상황이 발생하면 나머지 데이터를 처리하지 않고 Fail 상태로 스트리밍을 끝낸다. 데이터 처리가 모두 성공적으로 끝나면 Complete 상태로 스트리밍을 끝낸다.

<br/>
## Mono

`Mono<T>`는 0..1개의 방출된 항목에 대한 시퀀스를 나타내는 `Publisher<T>`이다. 시퀀스에 대해서 어떠한 행위를 취할것인가는 `Subscriber`의 `onNext(T t)`, `onComplete()`, `onError(Throwable t)`를 이용해 작성할 수 있다. `Mono`는 `Flux`에서 사용할 수 있는 연산자 중 일부만 제공한다. (Mono가 제공하는 데이터는 최대 1개이니 count와 같은 연산자를 제공할 이유가 없을 것이다.) 

<br/>
### Mono의 동작
다음은 `Mono`의 동작 방식이다.
![reactor-mono-01](https://user-images.githubusercontent.com/19832483/85026237-67629e00-b1b3-11ea-84c5-d669d7807fba.png)

데이터가 1개 이하를 처리한다는 점을 제외하고는, 위에서 설명한 `Flux`와 동일하게 동작하므로 자세하게 설명하지 않을 것이다.

다음은 `Mono`를 이용한 예제 코드이다. `Flux`와 유사하다. (subscribe() 메소드는 `Flux`와 `Mono`에 여러 메소드가 오버로딩 되어있으니 참고하자. 메소드가 달라도 기본 동작 방식은 위에서 설명했던 것과 비슷하다.)
``` java
        Mono.just("aa")
            .map(String::toUpperCase)
            .subscribe(input -> System.out.println(input));
```


<br/><br/>
여기까지 `Flux`와 `Mono`에 대해 살펴보았다. 실용적인 측면보다는 Reactor3에서 제공하는 `Flux`와 `Mono`가 기본적으로 어떻게 동작하는지에 초점을 맞춰 포스트를 작성하였다. 아무래도 기본적인 지식이 있어야 여러 상황에서 대처 및 응용을 쉽게할 수 있을것 같기 때문이다. 다음 포스트는 `Flux`와 `Mono`를 실용적인 측면에서 작성할 예정이다. (바쁘지만 않다면.) 

<br/><br/><br/>
***참고 리소스*** <br/>
***https://projectreactor.io/docs/core/release/reference/*** <br/>