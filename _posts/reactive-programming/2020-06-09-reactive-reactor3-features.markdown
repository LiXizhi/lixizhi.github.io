---
layout: post
title:  "Reactive Programming & Reactor3 #02 - Reactor3 특징"
date:   2020-06-09 15:14:55
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

<br/><br/><br/>
***참고 리소스*** <br/>
***https://projectreactor.io/docs/core/release/reference/*** <br/>
***https://tech.io/playgrounds/929/reactive-programming-with-reactor-3/Intro***