---
layout: post
title:  "Effective Java 3rd - Chapter08. 메소드"
date:   2018-12-26 11:07:00
categories: java
comments: true
---
Effective Java를 공부하는 도중에 블로그 개설을 해서 챕터 앞부분은 추후에 복습겸 포스팅을 할 예정이다. 같은 이유로 Item 53부터 이 글을 시작하겠다.

### Item 53. 가변인수는 신중히 사용하라
가변인수는 인수 개수가 정해지지 않았을 때 아주 유용하다.
그러나 가변인수 메서드는 호출될때 마다 배열을 새로 하나 할당하고 초기화하기 때문에, 다음과 같은 패턴을 주로 사용한다.

	public void foo() {}
    public void foo(int a1) {}
    public void foo(int a1, int a2) {}
    public void foo(int a1, int a2, int a3) {}
    public void foo(int a1, int a2, int a3, int... rest) {}

주로 사용하는 메서드의 인자 개수를 파악한후에, 마지막 다중정의 메서드가 주로 사용하는 인자 개수 이상인 메서드 호출을 담당한다.