---
layout: post
title:  "자바의 동시성 #2 - 동시성 프로그래밍에서 발생할 수 있는 문제점과 volatile, synchrozied 키워드"
date:   2019-01-14 01:11:00
categories: java
comments: true
---
* content
{:toc}

[이전의 포스트][Pre-Post]에서는 자바 스레드가 하드웨어 수준에서 어떻게 연관되어 동작하는지 살펴보았다. 이번 포스트에서는 앞에서 설명했던 내용을 기반으로, 동시성 프로그래밍에서 발생할 수 있는 문제점과 자바에서 이를 해결하기 위해 지원하는 키워드인 `volatile`, `synchrozied`의 동작 방식에 대해 살펴볼 것이다.
<br><br>

### CPU-RAM 아키텍쳐와 동시성 프로그래밍에서 발생할 수 있는 문제점
현대 컴퓨터의 CPU와 RAM의 관계도를 그려보면 다음과 같은 그림이 될것이다. 설명의 편의성을 위해 2CPU 2코어 2스레드 모델을 예로 들겠다.
<br>
![computer-CPU-RAM-01](https://user-images.githubusercontent.com/19832483/51120169-dc5fd480-1857-11e9-909e-4eb8201a2f44.png){: .u-mid-img}
<br><br>
`CPU`가 어떤 작업을 처리하기 위해 데이터가 필요할 때, `CPU`는 `RAM`의 일부분을 고속의 저장 징치인 `CPU Cache Memory`로 읽어들인다. 이 읽어들인 데이터로 명령을 수행하고 이 데이터를 다시 `RAM`에 저장하기 위해서는 데이터를 읽어들일 때의 과정을 역순으로 밟는다. 즉, 적절한 시점에 `CPU Cache Memory`에서 `RAM`으로 쓰기 작업을 하게 되는데, 이 의미는 `CPU`가 캐시에 쓰기 작업을 수행했다고 해서 바로 `RAM`으로 쓰기 작업을 수행할 필요가 없다는 것이다. 반대의 과정인 읽기 작업도 마찬가지이다.

동시성 프로그래밍에서는 `CPU`와 `RAM`의 중간에 위치하는 `CPU Cache Memory`와 `병렬성`이라는 특징 때문에 다수의 스레드가 공유 자원에 접근할 때 두 가지 문제가 발생할 수 있다. 하나는 `가시성`의 문제이고, 다른 하나는 `동시 접근`의 문제이다. 사실 위의 두 문제는 동시성보다는 `병렬성` 때문에 발생하는 문제이지만, 자바 스레드는 동시성의 성질을 가지고 있으므로, 자바에서는 동시성 프로그래밍에서 발생하는 문제점이라고 부르는듯 하다.
<br><br>

### 가시성(visibility)의 문제와 volatile 키워드
하나의 스레드에서 공유 자원(변수 및 객체)을 수정한 결과가 다른 스레드에게는 보이지 않을수 있다. 이 뜻이 무엇일까?? 다음의 자바 코드를 살펴보자.
```java
public class StopThread {

    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });

        backgroundThread.start();

        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```
메인 스레드가 1초 후 stopRequest를 true로 설정하면 `backgroundThread`는 반복문을 빠져나올 것처럼 보일 것이다. 하지만 실행시켜보면 다음의 코드는 영원히 수행될 수도 있다. 왜 이런 일이 발생한 것일까? (사실 위 코드는 jvm의 `hoisting`이라는 최적화 기법과도 연관이 되어 있지만, 이 포스트의 주제를 벗어나므로 이에 대해 설명하지 않겠다.)

다음의 그림을 살펴보자.
<br>
![concurrency-visibility-01](https://user-images.githubusercontent.com/19832483/51120188-e41f7900-1857-11e9-9178-02204167f598.png){: .u-mid-img}
<br><br>
CPU1에서 수행된 스레드를 `backgroundThread`, CPU2에서 수행된 스레드를 `mainThread`라고 하자. `mainThread`는 `CPU Cache Memory 2`와 `RAM`에 공유 변수인 `stopRequested`를 `true`로 쓰기 작업을 완료했으나, `backgroundThread`는 `CPU Cache Memory 1`에서 읽은 여전히 업데이트 되지 않은 `stopRequested`값을 사용한다. 이 값은 `false`이므로 무한 루프를 수행하게 된다. 즉, `mainThread`가 수정한 값을 `backgroundThread`가 언제쯤에나 보게 될지 보증할 수 없다. 이러한 문제점을 `가시성`의 문제라고 한다.

이 문제를 해결하기 위해서는 `stopRequested` 변수를 `volatile`로 선언하면 된다. `volatile`로 선언된 변수에 대해서는 다음 그림과 같이 `CPU Cache Memory`를 거치지 않고 `RAM`으로 직접 읽고 쓰는 작업을 수행하게 된다. 
<br>
![concurrency-visibility-02](https://user-images.githubusercontent.com/19832483/51120190-e5e93c80-1857-11e9-9d30-29077ef281bb.png){: .u-mid-img}
<br><br>

### 동시 접근의 문제와 synchronized 키워드
여러 스레드에서 공유 자원에 동시에 접근하여 변경했을 때 문제가 발생할 수 있다. 다음의 자바 코드를 살펴보자.
```java
public class IncremantThread {

    private static int count;

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            for (int i = 0; i < 10000000; ++i) {
                ++count;
            }
        });
        backgroundThread.start();

        for (int i = 0; i < 10000000; ++i) {
            ++count;
        }

        TimeUnit.SECONDS.sleep(5);
        System.out.println(count);

    }
}
```
다음의 코드를 실행하면 5초뒤 결과값으로 `20000000`이 출력이 될까? 아마도 그렇지 않을 것이다. 이 코드에 대한 동작 과정을 다음의 그림으로 살펴보자.
<br>
![concurrency-race-condition-01](https://user-images.githubusercontent.com/19832483/51125190-72e5c300-1863-11e9-9765-23ddb62c0600.png){: .u-mid-img}
<br><br>
CPU1에서 수행된 스레드를 `backgroundThread`, CPU2에서 수행된 스레드를 `mainThread`라고 하자. 그리고 어느 순간에 `RAM`에 저장된 `count` 변수의 값이 `2`라고 가정해보자. 공유 자원에 대해서 `backgroundThread`와 `mainThread`는 동시에 `CPU Cache Memory`로 count 값을 읽어오게 한다. 두 스레드에서 `count` 값을 `1` 증가시키고, `3`이란 값을 각각의 `CPU Cache Memory`에 저장하게 한다. 그 후 두 `CPU Cache Memory`에 있는 `count` 값이 `RAM`에 저장이 된다면 `3`이란 값이 연속으로 중복 저장되게 될것이다. 무언가 이상하지 않은가? 순차적으로 실행했으면, `4`라는 정상적인 값이 `RAM`에 저장이 되었을텐데, 동시에 공유 자원에 접근하다보니 아이러니한 일이 발생해버렸다. 이러한 문제점을 `동시 접근`의 문제라고 한다.

이 문제를 해결하기 위해서는, `synchronized` 키워드를 사용하면 된다. `synchronized`는 `lock`을 이용하여 스레드가 공유 자원에 접근시 하나의 스레드만 공유 자원에 접근할 수 있도록 한다. 다음의 코드는 `synchrozied`를 이용해 위의 문제를 해결한 수정된 코드이다.
```java
public class IncremantThread {

    private static int count;

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            for (int i = 0; i < 10000000; ++i) {
                increment();
            }
        });
        backgroundThread.start();

        for (int i = 0; i < 10000000; ++i) {
            increment();
        }

        TimeUnit.SECONDS.sleep(5);
        System.out.println(count);

    }

    private static synchronized void increment() {
        ++count;
    }
}
```
`synchrozied`는 `가시성`의 문제도 해결한다. 하지만 `volatile`은 `동시 접근`의 문제를 해결하지 못 한다. 추가로 `synchrozied`와 `volatile`은 위에서 잠깐 언급한 jvm의 최적화 기법을 방지하는 역할을 한다.

[Pre-Post]:https://badcandy.github.io/2019/01/14/concurrency-01