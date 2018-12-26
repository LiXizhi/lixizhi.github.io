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
```
	public void foo() {}
    public void foo(int a1) {}
    public void foo(int a1, int a2) {}
    public void foo(int a1, int a2, int a3) {}
    public void foo(int a1, int a2, int a3, int... rest) {}
```
주로 사용하는 메서드의 인자 개수를 파악한후에, 마지막 다중정의 메서드가 주로 사용하는 인자 개수 이상인 메서드 호출을 담당한다.
 
### Item 54. null이 아닌, 빈 컬렉션이나 빈 배열을 반환하라
컬렉션이나 배열 반환시에 size() 또는 length가 0이면 null을 반환하지 말아라. null을 반환하게 된다면 null에 대한 방어코드를 클라이언트에서 처리해야한다.
이를 방지하기 위해서 null이 아닌, 빈 컬렉션이나 빈 배열을 반환하자.
혹시나 성능이 우려된다면(대부분의 경우 그렇지 않겠지만), 불변 객체를 미리 할당하고, 이를 재사용하자.
```
	public List<Cheese> getCheeses() {

		// Collections.emptyList()는 불변 리스트 객체를 리턴한다.
		return cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheessesInStock);
	}
 ```
### Item 55. 옵셔널 반환은 신중히 하라
결과가 없을 수 있으며, 클라이언트가 이 상황을 특별하게 처리해야 한다면 Optional<T>를 반환하라. 아래는 클라이언트에서 Optional의 활용 예제이다.
```
	// 옵셔널 활용1 - null일 경우, 기본값을 정해둘 수 있다.
	String lastWordInLexicon = max(words).orElse("단어 없음..");

	// 옵셔널 활용2 - 원하는 예외를 던질 수 있다.
	Toy myToy = max(toys).orElseThrow(ToysException::new)
```
컬렉션, 스트림, 배열, 옵셔널 같은 컨테이너 타입은 옵셔널로 감싸면 안된다. 또한 컬렉션의 키, 값, 원소나 배열의 원소로 사용하는게 적절한 상황은 거의 없다.
Optional 자체도 초기화해야하는 객체이므로 성능이 중요한 상황에서는 적절하지 않을수 있다.
 
### Item 56. 공개된 API 요소에는 항상 문서화 주석을 작성하라
