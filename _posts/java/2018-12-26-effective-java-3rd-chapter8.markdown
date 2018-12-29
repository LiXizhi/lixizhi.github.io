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
```java
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
```java
public List<Cheese> getCheeses() {

	// Collections.emptyList()는 불변 리스트 객체를 리턴한다.
	return cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheessesInStock);
}
```
### Item 55. 옵셔널 반환은 신중히 하라
결과가 없을 수 있으며, 클라이언트가 이 상황을 특별하게 처리해야 한다면 Optional<T>를 반환하라. 아래는 클라이언트에서 Optional의 활용 예제이다.
```java
// 옵셔널 활용1 - null일 경우, 기본값을 정해둘 수 있다.
String lastWordInLexicon = max(words).orElse("단어 없음..");

// 옵셔널 활용2 - 원하는 예외를 던질 수 있다.
Toy myToy = max(toys).orElseThrow(ToysException::new)
```
컬렉션, 스트림, 배열, 옵셔널 같은 컨테이너 타입은 옵셔널로 감싸면 안된다. 또한 컬렉션의 키, 값, 원소나 배열의 원소로 사용하는게 적절한 상황은 거의 없다.
Optional 자체도 초기화해야하는 객체이므로 성능이 중요한 상황에서는 적절하지 않을수 있다.
 
### Item 56. 공개된 API 요소에는 항상 문서화 주석을 작성하라
API를 문서화하려면 공개된 모든 클래스, 인터페이스, 메서드, 필드 선언에 문서화 주석을 달아야 한다. 직렬화할 수 있는 클래스라면 직렬화 형태에 관해서도 적어야 한다.

다음은 메소드 주석화에 관한 설명이다.

- 상속용으로 설계된 클래스의 메서드가 아니라면 how가 아닌 what을 기술해야한다.
- 클라이언트가 해당 메서드를 호출하기 위한 전제조건을 모두 나열해야한다. 전제조건은 `@throws` 태그로 비검사 예외를 선언하여 암시적으로 기술한다.
- 메서드가 성공적으로 수행된 후에 만족해야 하는 사후조건도 모두 나열해야 한다.
- 시스템의 상태에 어떠한 변화를 가져올 것이라는 부작용도 명시해야한다.
	- 백그라운드 스레드를 시작시키는 메서드라면 그 사실을 문서에 밝혀야 한다.
- 모든 매개변수에 `@param` 태그를 붙인다. (명사구로 쓴다.)
- 발생할 가능성이 있는(검사든 비검사든) 모든 예외에 `@throws` 태그를 붙인다.
- 반환 타입이 void가 아니라면 `@return` 태그를 붙인다. (명사구로 쓴다.)

다음은 위의 규칙을 모두 반영한 문서화 주석의 예이다.

```java
/**
* Returns the element at the specified position in this list.
*
* <p>This method is <i>not</i> guaranteed to run in constant
* time. In some implementations it may run in time proportional
* to the element position.
*
* @param  index index of element to return; must be
*         non-negative and less than the size of this list
* @return the element at the specified position in this list
* @throws IndexOutOfBoundsException if the index is out of range
*         ({@code index < 0 || index >= this.size()})
*/
E get(int index) {
	return null;
}
```

자바독 유틸리티는 문서화 주석을 HTML로 변환하므로 HTML 태그를 사용할수 있다.
`{@code}` 태그는 태그로 감싼 내용을 코드용 폰트로 렌더링 한다. 여러 줄로 된 코드 예시를 넣으려면 `{@code}` 태그를 다시 `<pre>` 태그로 감싸면 된다.

클래스를 상속용으로 설계했을 때에는 자기사용 패턴에 대해서 주석화 해야한다.
`@implSpec` 태그로 하위 클래스들이 그 메서드를 상속하거나, super 키워드를 이용해 호출할 때 어떻게 동작하는지 설명해야 한다.

다음은 `@implSpec`을 반영한 문서화 주석의 예이다.

```java
/**
* Returns true if this collection is empty.
*
* @implSpec This implementation returns {@code this.size() == 0}.
*
* @return true if this collection is empty
*/
public boolean isEmpty() {
	return false;
}
```

API 설명에 <,>,& 등의 HTML 메타문자를 포함시키려면 `{@literal}` 태그를 사용하자.

문서화 주석의 첫 번째 문장은 해당 요소의 요약 설명으로 간주된다.
메서드와 생성자의 요약 설명은 해당 메서드와 생성자의 동작을 설명하는 주어가 없는 동사구여야 한다.
한편 클래스, 인터페이스, 필드의 요약 설명은 대상을 설명하는 명사절이여야 한다.

제네릭 타입이나 제네릭 메서드를 문서화할 때는 모든 타입 매개변수에 주석을 달아야 한다.
다음은 자바 Map의 예제이다.

```java
/**
* An object that maps keys to values.  A map cannot contain duplicate keys;
* each key can map to at most one value. 
*
* @param <K> the type of keys maintained by this map
* @param <V> the type of mapped values
*/
public interface Map<K,V> {
	...
}
```

열거 타입을 문서화할 때는 상수들에도 주석을 달아야 한다.

```java
/**
* An instrument section of a symphony orchestra.
*/
public enum OrchestraSection {
	/** Woodwinds, such as flute, clarinet, and oboe. */
	WOODWIND,

	/** Brass instruments, such as french horn and trumpet. */
	BRASS,

	/** Percussion instruments, such as timpani and cymbals. */
	PERCUSSION,

	/** Stringed instruments, such as violin and cello. */
	STRING;
}
```

또한, 어노테이션 타입을 문서화할 때는 멤버들에도 모두 주석을 달아야 한다. 어노테이션 타입의 요약 설명은 프로그램 요소에 이 어노테이션을 단다는 것이 어떤 의미를 설명하는 동사구로 쓴다.

```java
/**
* Indicates that the annotated method is a test method that
* must throw the designated exception to pass.
*/
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
	/**
	* The exception that the annotated test method must throw
	* in order to pass. (The test is permitted to throw any
	* subtype of the type described by this class object.)
	*/
	Class<? extends Throwable> value();
}
```

패키지를 설명하는 문서화 주석은 `package-info.java` 파일에 작성한다. 모듈 시스템을 사용한다면 모듈 관련 설명은 `module-info.java` 파일에 작성하면 된다.

클래스 혹은 정적 메서드가 스레드 안전하든 그렇지 않든, 스레드 안전 수준을 반드시 API 설명에 포함해야 한다.
또한 직렬화할 수 있는 클래스라면 직렬화 형태도 API 설명에 기술해야 한다.

`{@inheritDoc}` 태그를 사용해 상위 타입의 문서화 주석 일부를 상속할 수 있다.