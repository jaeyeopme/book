# Chapter 1 - 인터페이스

## 1.1 리스트가 두 종류인 이유

JCF (Java Collection Framework) 를 사용하다 보면 종종 `ArrayList` 와 `LinkedList` 를 혼동한다.
어떤 동작은 ArrayList 가 빠르거나 저장 공간을 적게 사용하고 다른 상황에서는 LinkedList 가 빠르거나 메모리 사용량이 적다. 어느 것이 더 좋을지는 수행하는 동작에 달려 있다.

## 1.2 자바 interface

interface 는 메서드 집합을 의미하며 이 interface 를 구현하는 클래스는 이러한 메서드를 제공해야 한다.

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

이 interface 는 T 를 사용하여 Comparable 이라는 **제네릭** 타입을 정의한다.

**interface 를 구현하기 위한 클래스의 조건**

* T 타입을 명시해야 한다.
* T 타입을 객체의 인자로 받고 int 를 반환하는 `compareTo()` 메서드를 제공해야 한다.

```java
public final calss Integer extends Number implements Comparable<Integer> {
    @Override
    public int compareTo(Integer anotherInteger) {
        // 주로 삼항 연산자를 사용한다.
        int thisVal = this.value;
        int anotherVal = anotherInteger.value;
        return thisVal < anotherVal ? -1 : (thisVal == anotherVal ? 0 : 1);
    }
}
```
이 클래스는 Number 클래스를 확장하며 Number 클래스의 메서드와 인스턴스 변수를 상속하고 `Comparable<Integer>` 인터페이스를 구현한다. 클래스가 interface 를 구현한다고 선언하면 컴파일러는 interface 가 정의한 모든 메서드를 제공하는지 확인한다. 

## 1.3 List interface

JCF 는 List 라는 interface 를 정의하고 ArrayList 와 LinkedList 라는 두 구현 클래스를 제공한다. interface 를 구현하는 클래스는 `add`, `get`, `remove` 와 약 20가지 메서드를 포함한 특정 메서드 집합을 제공해야 한다. 이 둘은 이러한 메서드를 제공하기 때문에 상호 교환 할 수 있다.

```java
public class ListCliendExample {
    private List list;
    
    public ListCliendExample() {
        this.list = new LinkedList();
    }
    
    private List getList() {
        return this.list;
    }
    
    public static void main(String[] args) {
        ListCliendExample lce = new ListCliendExample();
        List list = lce.getList();
        System.out.println(list);
    }
}
```

생성자만 바꾸면 List 를 구현한 다른 클래스로 변경(인터페이스 프로그래밍)할 수 있다. 특정 구현에 의존해서는 안되며, 구현체가 아닌 인터페이스에 의존한다면 변경이 필요할 때 구현체만 변경하면 된다. 반면, 인터페이스가 변경하면 구현체도 변경되어야 한다. 이것이 개발자가 꼭 필요한게 아니라면 인터페이스를 변경하지 않는 이유다.

## 1.4 실습 1

List 의 구현체를 LinkedList 를 ArrayList 클래스로 교체하라.

```java
public class ListClientExample {

  private final List<Integer> list;

  public ListClientExample() {
    // list = new LinkedList<>();
    list = new ArrayList<>();
  }

  public static void main(String[] args) {
    ListClientExample lce = new ListClientExample();

    List<Integer> list = lce.getList();
    System.out.println(list);
  }

  public List<Integer> getList() {
    return list;
  }
}
```

> 이 테스트는 좋은 테스트 코드가 아니며, 좋은 테스트라면 구현 세부사항에 의존해서는 안된다.

```java
public class ListClientExampleTest {

	/**
	 * Test method for {@link ListClientExample}.
	 */
	@Test
	public void testListClientExample() {
		ListClientExample lce = new ListClientExample();

		List<Integer> list = lce.getList();
		assertThat(list, instanceOf(ArrayList.class) );
	}

}
```
