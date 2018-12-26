---
layout: post
title:  "Effective Java 3rd - Chapter08. 메서드"
date:   2018-12-26 11:07:00
categories: java
comments: true
---
Effective Java를 공부하는 도중에 블로그 개설을 해서 챕터 앞부분은 추후에 복습겸 포스팅을 할 예정이다. 같은 이유로 Item 53부터 이 글을 시작하겠다.

### Item 53. 가변인수는 신중히 사용하라
가변인수는 인수 개수가 정해지지 않았을 때 아주 유용하다.
그러나 가변인수 메서드는 호출될때 마다 배열을 새로 하나 할당하고 초기화하기 때문에, 성능에 민감한 상황인 경우 다음과 같은 패턴을 주로 사용한다.

	public void foo() {}
    public void foo(int a1) {}
    public void foo(int a1, int a2) {}
    public void foo(int a1, int a2, int a3) {}
    public void foo(int a1, int a2, int a3, int... rest) {}

주로 사용하는 메서드의 인자 개수를 파악한후에, 마지막 다중정의 메서드가 주로 사용하는 인자 개수 이상인 메서드 호출을 담당한다.

### Item 54. null이 아닌, 빈 컬렉션이나 빈 배열을 반환하라
컬렉션이나 배열 반환시에 size() 또는 length가 0이면 null을 반환하지 말아라. null을 반환하게 된다면 null에 대한 방어코드를 클라이언트에서 처리해야한다.
이를 방지하기 위해서 null이 아닌, 빈 컬렉션이나 빈 배열을 반환하자.
혹시나 성능이 우려된다면(대부분의 경우 그렇지 않겠지만), 불변 객체를 미리 할당하고, 이를 재사용하자.

	public List<Cheese> getCheeses() {

		// Collections.emptyList()는 불변 리스트 객체를 리턴한다.
		return cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheessesInStock);
	}