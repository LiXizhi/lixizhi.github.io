---
layout: post
title:  "Effective Java 3rd - Chapter10. 예외"
date:   2019-01-01 01:11:00
categories: java
comments: true
---
* content
{:toc}

### Item 69. 예외는 진짜 예외 상황에서만 사용하라
배열의 원소를 모두 순회한후 반복문에서 빠져나오기 위해 다음과 깉이 예외 처리한 코드를 보자.

```java
try {
	int i = 0;
	while (true) {
		range[i++].clumb();
	}
} catch (ArrayIndexOutOfBoundsException e) {

}
```

코드의 의도는 배열의 원소를 모두 순회했는지 체크하는 중복 if문을 생략하면 성능이 더 좋아질 것이라고 기대하여 위와 같이 작성했을 것이다. 엄연히 말해서 다음의 이유로 잘못된 추론이다.
1. 코드를 `try-catch` 블록 안에 넣으면 JVM이 적용할 수 있는 최적화가 제한된다.
2. 배열을 순회하는 표준 관용구는 앞서 걱정한 if문 중복 검사를 수행하지 않는다. JVM이 알아서 최적화해 없애준다.
3. `try-catch` 안에 다른 배열을 사용하는 코드가 있고 그 부분에서 `ArrayIndexOutOfBoundsException`이 발생했다면 위의 코드를 사용하는 메서드는 오동작을 일으킬 것이다.

**그러므로 예외는 오직 예외 상황에서만 사용해야 하고, 클라이언트 API에게 정상적인 제어 흐름에서 예외 처리를 강요하는 API 설계를 하지말아야 한다.**
<br><br>

### Item 70. 복구할 수 있는 상황에는 검사 예외를, 프로그래밍 오류에는 런타임 예외를 사용하라
**호출하는 쪽에서 복구하리라 여겨지는 상황이라면 `검사 예외`를 사용하라.** API 설계자는 API 사용자에게 검사 예외를 던져주어 그 상황에서 회복해내라고 요구한 것이다. 검사 예외를 사용한다면 API 사용자가 복구할 수 있도록 복구에 필요한 정보를 알려주는 메서드를 검사 예외 클래스에 제공하자.

**프로그래밍 오류를 나타낼 때는 `런타임 예외(비검사 예외)`를 사용하자.** 런타임 예외의 대부분은 전제조건을 만족하지 못했을 때 발생한다. 이 경우 복구가 불가능하거나, 더 실행해봐야 득보다는 실이 많다는 뜻이다. 예를 들어 `ArrayIndexOutOfBoundsException`이 발생했다는 건 배열의 인덱스 전제조건을 지키지 않았다는 뜻이고 복구 불가능하다는 뜻이다. 복구 가능하다고 확신하기 어렵다면 비검사 예외를 선택하자.

**`Error`, `Throwable`은 사용하지 않도록 한다.** Api 사용자들을 헷갈리게 할 뿐이다.
<br><br>

### Item 71. 필요 없는 검사 예외 사용은 피하라
복구가 가능하고 호출자가 그 처리를 해주길 바란다면, 우선 다음과 같이 `Empty Optional`을 반환해도 될지 고민하자.
```java
public <T> Optional<T> doOpertation(T input) {
	...
	if () {
		// 정상 동작
		...
		return Optional.of(returnValue);
	} else {
		// 예외 케이스인 경우
		...
		return Optional.empty()
	}
}
```
`Optional`만으로는 상황을 처리하기에 충분한 정보를 제공할 수 없을 때만 `검사 예외`를 던지자. 검사 예외를 남용한다면, 그 API를 사용하는 클라이언트는 번거롭게 예외 처리 코드를 추가하는 번거로운 작업을 수행해야 한다. 복구할 방법이 없거나, 복구할 수 있다고 확신이 서지 않는다면 `비검사 예외`를 사용하자.
<br><br>

### Item 72. 표준 예외를 사용하라
자바 라이브러리는 대부분 API에서 쓰기에 충분한 수의 예외를 제공하므로 재사용하는 것이 좋다. 표준 예외를 재사용한다면 다음과 같은 장점이 있다.
1. 많은 프로그래머에게 이미 익숙해진 규약을 그대로 따르므로 작성된 API가 다른 사람이 익히고 사용하기 쉬워진다.
2. 예외 클래스 수가 적을수록 메모리 사용량도 줄고 클래스를 적재하는 시간도 줄어든다.

다음은 널리 재사용되는 예외들이다.
- `IllegalArgumentException`
	- 허용하지 않는 값이 인수로 건네졌을 때
- `IllegalStateException`
	- 객체가 메서드를 수행하기에 적절하지 않은 상태일 때
	- ex) 초기화 하지 않은 객체로 어떤 메서드를 수행하려할 때
- `NullPointerException`
	- null을 허용하지 않는 메서드에 null을 건넸을 때
- `IndexOutOfBoundsException`
	- 인덱스가 범위를 넘어섰을 때
- `ConcurrentModificationException`
	- 허용하지 않는 동시 수정이 발견됐을 때
	- ex) 단일 스레드에서 사용하려고 설계한 객체를 여러 스레드가 동시에 수정하려 할 때
- `UnsupportedOperationException`
	- 호출된 메서드를 지원하지 않을 때
	- ex) 순수 view로써 제공되는 List 구현체의 remove() 메서드에 적용

단, API 문서를 참고해 그 예외가 어떤 상황에서 던져지는지 꼭 확인해야 한다. 예외의 이름뿐 아니라 예외가 던져지는 맥락도 부합할 때만 재사용한다. 더 많은 정보를 제공하길 원한다면 표준 예외를 확장해도 좋다.

Exception, RuntimeException, Throwable, Error는 의미가 너무 포괄적이므로 직접 재사용하지 말고 추상 클래스 취급하라.
<br><br>

### Item 73. 추상화 수준에 맞는 예외를 던져라
아래 계층에서 발생한 예외가 상위 계층에 그대로 노출되기 곤란하다면 `예외 번역`을 사용하라. 다음은 `AbstractSequentialList`에서 수행하는 예외 번역의 예다.
```java
public E get(int index) {
	ListIterator<E> i = listIterator(index);
	try {
		return i.next();
	} catch (NoSuchElementException e) {
		throw new IndexOutOfBoundsException("인덱스: " + index);
	}
}
```
위 코드는 `NoSuchElementException` 예외를 잡아 문맥상 더 알맞은 의미의 `IndexOutOfBoundsException`로 예외 번역 해준다.

예외를 번역할 때, 저수준 예외가 디버깅에 도움이 된다면 저수준 예외를 고수준 예외에 실어 보내는 `예외 연쇄`를 사용하는 게 좋다. 그러면 별도의 접근자 메서드 `Throwable.getCause()`를 이용해 저수준 예외를 꺼내볼 수 있다.
```java
try {
	...
} catch (LowerLevelException cause) {
	throw new HigherLevelException(cause);
}

class HigherLevelException extends Exception {
	HigherLevelException(Throwable cause) {
		super(cause);
	}
}
```
위 코드는 저수준 예외인 `LowerLevelException`을 고수준 예외인 `HigherLevelException`에 실어 보낸다.

하지만 가능하다면 저수준 메서드가 반드시 성공하도록 하여 아래 계층에서는 예외가 발생하지 않도록 하는것이 최선이다. 아래 계층에서 발생하는 예외를 피할 수 없다면 다음의 방법도 고려해보자.
- 상위 계층 메서드의 매개변수 값을 아래 계층 메서드로 건네기 전에 미리 검사하는 방법
- 예외가 발생하면 예외를 전파하지 않고 `logging`하는 방법
	- 클라이언트 코드와 사용자에게 문제를 전파하지 않으면서 프로그래머가 로그를 분석해 추가 조치를 취할 수 있게 해준다.
<br><br>

### Item 74. 메서드가 던지는 모든 예외를 문서화하라
- 메서드에서 예외가 발생하는 상황을 자바독의 `@throws` 태그를 사용하여 정확히 문서화하라.
	- 비검사 예외 같은 경우 현실적으로 불가능할 때가 있다. 작성한 클래스가 사용하는 외부 클래스가 새로운 비검사 예외를 던지게 수정했다면 문서에 언급되지 않은 새로운 비검사 예외를 전파하게 될 것이다.
- `검사 예외`만 메서드 선언부의 `throws`에 일일이 선언하자. API를 사용하는 클라이언트에게 정확한 정보를 주기 위해 Exception이나 Throwable처럼 포괄적으로 선언하면 안 된다.
- 한 클래스에 정의된 많은 메서드가 같은 이유로 같은 예외를 던진다면 그 예외를 각각의 메서드가 아닌 클래스 설명에 추가하는 방법도 있다.
<br><br>

### Item 75. 예외의 상세 메시지에 실패 관련 정보를 담아라
예외가 발생했을 때, **실패 원인을 분석하기 위해 발생한 예외에 관여된 모든 매개변수와 필드 값을 실패 메세지(`toString()`)에 담아야 한다.** 이를 구현하기 위해서 아래 코드와 같이 필요한 정보를 예외 생성자에서 모두 받아 상태 메세지까지 미리 생성해놓는 방법을 사용할 수 있다.
```java
	// 해당 예외의 디버깅에 유용한 정보인 lowerBound, upperBound, index를 예왼 생성자에서 받는다.
    public IndexOutOfBoundsException(int lowerBound, int upperBound,
                                     int index) {
        // Generate a detail message that captures the failure
        super(String.format(
                "Lower bound: %d, Upper bound: %d, Index: %d",
                lowerBound, upperBound, index));

        // Save failure information for programmatic access
        this.lowerBound = lowerBound;
        this.upperBound = upperBound;
        this.index = index;
    }
```
또한 예외는 실패와 관련한 정보를 얻을 수 있는 접근자 메서드를 제공하는 것이 좋다. 접근자 메서드는 예외 상황을 복구하는데 유용할 수 있으므로 검사 예외에서 빛을 발한다.
<br><br>

### Item 76. 가능한 실패 원자적으로 만들어라
호출된 메서드가 실패하더라도 해당 객체는 가능한 메서드 호출 전 상태를 유지해야 한다. 이러한 특성을 `실패 원자적(failure-atomic)`이라고 한다. 다음의 방법으로 메서드를 실패 원자적으로 만들 수 있다.
- `불변 객체`로 설계한다.
- 작업 수행에 앞서 매개변수의 유효성을 검사한다. 객체 내부 상태를 변경하기 전에 잠재적 예외의 가능성 대부분을 걸러낼 수 있다.
- 실패할 가능성이 있는 모든 코드를, 객체의 상태를 바꾸는 코드보다 앞에 배치한다.
	- `TreeMap`은 `key`를 기준으로 하여 원소들을 정렬한다. 엉뚱한 타입의 원소를 추가하려 들면 트리를 변경하기 앞서, 해당 원소가 들어갈 위치를 찾는 과정에서 `ClassCastException`을 던진다. 
- 객체의 임시 복사본에서 작업을 수행한 다음, 작업이 성공적으로 완료되면 원래 객체와 교체한다.
- 작업 도중 발생하는 실패를 가로채는 복구 코드를 작성하여 작업 전 상태로 되돌린다.
	- 주로 디스크 기반의 내구성을 보장하는 자료구조에 쓰이는데, 자주 쓰이는 방법은 아니다.

실패 원자성은 일반적으로 권장되는 덕목이지만 실패 원자성을 달성하기 위한 비용이나 복잡도가 아주 큰 연산일 경우 달성하지 못 할수도 있다. 

메서드 명세에 기술한 예외라면 설혹 예외가 발생하더라도 객체의 상태는 메서드 호출 전과 똑같이 유지돼야 한다는 것이 기본 규칙이다. 이 규칙을 지키지 못한다면 실패 시의 객체 상태를 API 설명에 명시해야 한다.
<br><br>

### Item 77. 예외를 무시하지 말라
API 설계자가 메서드 선언에 예외를 명시하는 까닭은, 그 메서드를 사용할 때 적절한 조치를 취해달라고 말하는 것이다. 그러므로 다음과 같이 예외를 무시하지 말자.
```java
try {
	...
} catch (SomeException e) {
	// 아무것도 하지 않음.
}
```
예외를 무시하지 않고 바깥으로 전파되게만 놔둬도 최소한 디버깅 정보를 남긴 채 프로그램이 신속하게 중단되게 할 수 있다.

어쩔 수 없이 예외를 무시해야 할 때도 있다. 예를 들어 `FileInputStream`의 `close()` 같은 경우 스트림을 닫는다는 건 필요한 정보는 이미 다 읽었다는 뜻이므로 복구하거나 남은 작업을 중단할 이유가 없다. 이처럼 예외를 무시하기로 했다면 catch 블록 안에 그러한 이유를 주석으로 남기고, 예외가 발생했다는 사실을 로그로 남기도록 하고, 예외 변수의 이름도 ignored로 바꾸자.
