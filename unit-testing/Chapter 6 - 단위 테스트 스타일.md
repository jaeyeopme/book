# Chapter 6 - 단위 테스트 스타일

단위 테스트 스타일에는 **출력** 기반, **상태** 기반, **통신** 기반 세 가지가 있다. 출력 기반 테스트의 품질이 **가장** 좋고, 상태 기반 테스트는 두 번째로 좋으며, 통신 기반 테스트는 간헐적으로만 사용해야 한다. 하지만 출력 기반 테스트는 아무데서나 사용할 수 없으며, **순수 함수** 방식으로 작성된 코드에만 적용되기 때문에 **함수형 프로그래밍 원칙**을 사용해 기반 코드가 **함수형 아키텍처**를 **지향**하게 **재구성**하면 된다.

함수형 프로그래밍과 출력 기반 테스트의 연관성을 이해하고, 함수형 프로그래밍과 함수형 아키텍처가 지닌 **한계**를 배운다.

## 6.1 단위 테스트의 세 가지 스타일

단위 테스트에는 세 가지 스타일이 있다.

* 출력 기반 테스트
* 상태 기반 테스트
* 통신 기반 테스트

하나의 테스트에서 모든 스타일을 자유롭게 사용할 수 있다.

### 6.1.1 출력 기반 테스트 정의

테스트 대상 시스템(SUT)에 입력을 넣고 **최종 출력**을 점검하는 방식이다. 이 스타일은 **전역 상태**나 **내부 상태**를 변경하지 않는 코드에만 적용되므로 **반환 값만** 검증하기 때문에 부작용이 없고 SUT의 **작업 결과**는 **호출자에게 반환하는 값** 뿐이다.

`PriceEngine` 클래스는 일련의 상품을 받아 할인을 계산한다.

```cs
// 출력 기반 테스트

public class PriceEngine
{
    public decimal CalculateDiscount(params Product[] products)
    {
        decimal discount = products.Length * 0.01m;
        return Math.Min(discount, 0.2m);
    }
}

[Fact]
public void Discount_of_two_products()
{
    var product1 = new Product("Hand wash");
    var product2 = new Product("Shampoo");
    
    decimal discount = sut.CalculateDiscoiunt(product1, product2);
    
    Assert.Equal(0.02m, discount);
}
```

이 클래스는 내부 컬렉션에 상품을 추가하거나 데이터베이스에 저장하지 않는다. `CalculateDiscount()`의 결과는 반환된 할인 즉 출력 값 뿐이다.

출력 기반 테스트는 **함수형**이라고도 한다. 이는 **부작용 없는 코드** 선호를 강조하는 함수형 프로그래밍에 뿌리를 두고 있다.

### 6.1.2 상태 기반 스타일 정의

작업이 완료된 후 시스템의 **최종 상태**를 확인하는 것이다. 상태라는 용어는 SUT가 협력자 중 하나, 또는 데이터베이스와 같은 **프로세스 외부 의존성의 상태** 등을 의미할 수 있다.

`Order` 클래스를 통해 클라이언트가 새로운 상품을 추가할 수 있다.

```cs
// 상태 기반 테스트

public class Order
{
    private readonly List<Product> _products = new List<Product>();
    public IReadOnlyList<Product> Products = _products.ToList();
    
    public void AddProduct(Product product)
    {
        _products.Add(product);
    }
}

[Fact]
public void Adding_a_product_to_an_order()
{
    var product = new Product("Hand wash");
    var sut = new Order();
    
    sut.AddProduct(product);
    
    Assert.Equal(1, sut.Products.Count);
    Assert.Equal(product, sut.Products[0]);
}
```

테스트는 상품 추가 후 `Products` 컬렉션의 상태를 검증한다. 출력 기반과는 달리 `AddProduct()`의 결과는 주문 상태의 변경이다.

### 6.1.3 통신 기반 스타일 정의

**목**을 사용해 **테스트 대상 시스템**과 **협력자** 간의 **통신**을 검증한다. 참고로 고전파는 통신 기반보다 상태 기반을 선호하며, 런던파는 이와 반대이다. 출력 기반은 **두 분파 모두** 사용한다.

```cs
// 통신 기반 테스트

[Fact]
public void Sending_a_greetings_email()
{
    var emailGatewayMock = new Mock<IemailGatewat>();
    var sut = new Controller(emailGatewayMock.Object);
    
    sut.GreetUser("user@email.com");
    
    emailGatewayMock.Verify(
        x => x.SendGreetingsEmail("user@email.com"),
        Times.Once);
}
```

## 6.2 단위 테스트 스타일 비교

좋은 단위 테스트의 4대 요소로 서로 비교하면 재미있을 것이다.

* 회귀 방지
* 리팩터링 내성
* 빠른 피드백
* 유지 보수성

### 6.2.1 회귀 방지와 피드백 속도 지표로 스타일 비교하기

회귀 방지 지표는 특정 스타일에 따라 달라지지 않으며, 회귀 방지 지표는 다음 세 가지 특성으로 결정된다.

* 테스트 중에 실행되는 코드의 양
* 코드 복잡도
* 도메인 유의성

어떤 스타일도 모든 부분들에서 도움이 되지 않으며, 코드 복잡도와 도메인 유의성의 경우 통신 기반 스타일은 **예외**가 하나 있는데, 작은 코드 조각을 검증하고 다른 것은 **모두 목**을 사용하는 등 **피상적인 테스트**가 되지 않도록 주의해야 한다. 하지만 이는 통신 기반 스타일의 특징이 아닌 **기술**을 **남용**하는 극단적인 사례이다.

테스트 스타일과 피드백 속도 사이에는 상관관계가 거의 없다. 테스트가 **프로세스 외부 의존성**과 떨어져 있는 한, 모든 스타일은 테스트 실행 속도가 거의 동일하다.

### 6.2.2 리팩터링 내성 지표로 스타일 비교하기

리팩터링 내성은 리팩터링 중 발생하는 **거짓 양성** 수에 대한 척도다. 거짓 양성은 식별할 수 있는 동작이 아니라 **구현 세부사항**에 결합된 테스트의 결과다.

* 출력 기반 테스트
  * 테스트가 테스트 대상 메서드에만 결합되므로 거짓 양성 방지가 가장 우수하다. 이러한 테스트가 결합되는 경우 테스트 대상 메서드가 **구현 세부 사항**일 때 뿐이다.

* 상태 기반 테스트
  * 일반적으로 거짓 양성이 되기 쉽다. 메서드 외에도 클래스 **상태**와 함께 작동하며, 큰 API 노출 영역에 의존하므로, **구현 세부 사항**과 결합할 가능성도 더 높다.

* 통신 기반 테스트
  * 거짓 양성에 가장 취약하다. 5장에서 살펴봤듯이, 테스트 대역으로 **상호 작용**을 확인하는 테스트는 대부분 깨지기 쉽다. **스텁**과의 **상호 작용**을 확인해서는 안 된다. 애플리케이션 **경계를 넘는 상호 작용**을 확인하고 상호 작용의 **부작용이 외부 환경에 보이는 경우**에만 목이 괜찮다.

테스트 스타일에 따라 필요한 노력은 다르지만 **캡슐화**를 잘 지키고 테스트를 **식별할 수 있는 동작**에만 결합하면 거짓 양성을 **최소**로 줄일 수 있다.

### 6.2.3 유지 보수성 지표로 스타일 비교하기

유지 보수성 지표는 단위 테스트 스타일과 밀접한 관련이 있으며, 완화할 수 있는 방법이 많지 않다. 다음은 단위 테스트의 **유지비 측정**을 위한 두 가지 특성이다.

* 테스트를 **이해**하기 얼마나 어려운가(테스트 크기에 대한 함수)?
* 테스트를 **실행**하기 얼마나 어려운가(테스트에 직접적으로 관련 있는 **프로세스 외부 의존성** 개수에 대한 함수)?

테스트가 크면 파악하기도 변경하기도 어렵다. 하나 이상의 **프로세스 외부 의존성**과 직접 작동하는 테스트는 의존성을 **운영**하는 데 시간이 필요하다.

출력 기반 테스트가 가장 유지 보수하기 용이하다. 메서드로 **입력**을 공급하는 것과 해당 **출력**을 검증하는 **단 몇 줄**로 두 가지를 수행할 수 있다. 거의 항상 짧고 간결하다. 그리고 전역 상태나 내부 상태를 변경할 리 없으며, 프로세스 외부 의존성을 다루지 않는다.

일반적으로 출력 기반 테스트보다 유지 보수가 쉽지 않다. 상태 검증은 종종 출력 검증 보다 **더 많은 공간**을 차지한다.

```cs
// 많은 공간을 차지하는 상태 검증

[Fact]
public void Adding_a_comment_to_an_article()
{
    var sut = new Article();
    var text = "Comment test";
    var authro = "John Doe";
    var now = new DateTime(2019, 4, 1);
    
    sut.AddComment(text, author, now);

    // 글의 상태 검증 (4 라인)    
    Assert.Equal(1, sut.Comments.Count);
    Assert.Equal(text, sut.Comments[0].Text);
    Assert.Equal(author, sut.Comments[0].Author);
    Assert.Equal(now, sut.Comments[0].DateCreated);
}
```

이 테스트는 글에 댓글을 추가 한 후 댓글 목록에 나타나는지 확인하며, 단순하고 댓글이 하나지만 검증부는 네 줄에 걸쳐있다. 대부분 코드를 숨기고 **헬퍼 메서드**로 문제를 완화할 수 있지만 이러한 메서드를 작성하고 **유지**하는 데 상당한 노력이 필요하다.

```cs
// 검증문에 헬퍼 메서드 사용

[Fact]
public void Adding_a_comment_to_an_article()
{
    ...
    // 헬퍼 메서드
    sut.ShouldContainNumberOfComments(1)
        .withComment(text, author, now);
}
```

상태 기반 테스트를 단축하는 또 다른 방법은 검증 대상 클래스의 **동등 멤버**(**참조**가 아니라 **값**으로 비교하는 클래스)를 정의해 검증하는 것이다.

```cs
// 값으로 비교하는 Comment

[Fact]
public void Adding_a_comment_to_an_article()
{
    ...
    // 값으로 비교
    sut.Comments.Should().BeEquivalentTo(comment);
}
```

개별 속성을 지정하지 않고도 전체 값을 비교할 수 있으며, `BeEquivalentTo()`를 통해 전체 컬렉션을 비교할 수 있다. 이는 본질적으로 클래스가 값이고 **값 객체**로 변환할 수 있을 때 효과적이며, 그렇지 않다면 코드 오염으로 이어진다.

헬퍼 메서드 사용과 값 객체로 클래스 변환하기는 가끔만 적용할 수 있다. 또한 적용할 수 있더라도 상태 기반 테스트는 출력 기반 테스트보다 **유지 보수성**이 떨어진다.

통신 기반 테스트의 유지 보수성 지표는 가장 낮다. 테스트 대역과 **상호 작용** 검증을 설정해야 하며, 목이 _사슬 형태_로 있을 때 테스트는 더 커지고 유지 보수하기 어려워진다.

> 목 사슬이란 목이 다른 목을 반환하고, 그 목은 또 다른 목을 반환하는 여러 계층으로 있는 목이나 스텁을 말한다.

### 6.2.4 스타일 비교하기: 결론

출력 기반 테스트가 가장 결과가 좋다. **구현 세부 사항**과 거의 결합되지 않고 간결하며, **프로세스 외부 의존성**이 없기 때문에 유지 보수도 쉽다.

상태와 통신 테스트는 **유출된 구현 세부 사항**에 결합할 가능성이 높고, 크기도 커서 **유지비**가 많이 든다.

그러므로 항상 출력 기반을 선호하는게 좋지만, 출력 기반은 **함수형**으로 작성된 코드에만 적용할 수 있고, 대부분 객체지향 프로그래밍 언어에는 해당하지 않는다. 하지만 테스트를 출력 기반 스타일로 변경하는 **기법**은 존재한다.

이 장의 나머지 부분에서 상태 기반과 통신 기반의 테스트에서 출력 기반 테스트로 바꾸는지 보여준다. 코드를 **순수 함수**로 만들면 출력 기반 테스트가 가능해진다.

## 6.3 함수형 아키텍처 이해

이 절에서는 함수형 프로그래밍과 함수형 아키텍처가 무엇인지 알아보고, 함수형과 육각형 아키텍처의 연관성을 알아본다.

### 6.3.1 함수형 프로그래밍이란?

출력 기반 단위 테스트 스타일은 기반 제품 코드를 함수형 프로그래밍을 이용해 순수 함수 방식으로 작성해야하기 때문에 함수형이라고도 한다. 함수형이란 수학적 함수(순수 함수)를 사용한 프로그래밍이다.

수학적 함수는 **숨은 입출력**이 없는 함수(또는 메서드)이며, 모든 입출력은 메서드 이름, 인수, 반환 타입으로 구성된 **메서드 시그니처**에 명시해야 한다. 그리고 **호출 횟수에 상관 없이 주어진 입력**에 대해 **동일한 출력**을 생성한다. 다음은 출력 기반 테스트에서 살펴봤던 코드이다.

```cs
public decimal CalculateDiscoiunt(Product[] products)
{
    decimal discount = products.Length * 0.01m;
    return Math.Min(discount, 0.2m);
}
```

이 메서드는 **하나의 입력**과 **하나의 출력**이 있으며 둘 다 **메서드 시그니처**에 명시돼 있기 때문에 `CalculateDiscount()`는 **수학적 함수**가 된다.

**숨은 입출력**이 없는 메서드는 수학에서 말하는 함수의 정의를 준수하기 때문에 **수학적 함수**라고 한다. 입출력을 명시한 수학적 함수는 이에 따르는 테스트가 짧고 간결하며 이해하고 유지보수하기 쉽기 때문에 테스트하기 매우 쉽다. 반면 숨은 입출력은 코드를 테스트하기 힘들게 하며, 가독성도 떨어진다. 숨은 입출력의 유형은 다음과 같다.

* 부작용
  * 메서드 시그니처에 표시되지 않는 출력이며, 따라서 숨어있다. 연산은 클래스 인스턴스의 **상태**를 **변경**하고 디스크 파일을 업데이트 하는 등 **부작용**을 발생시킨다.
* 예외
  * 메서드가 예외를 던지면 프로그램 흐름에 메서드 시그니처에 설정된 계약을 **우회**하는 경로를 만든다. 호출된 에외는 호출 스택의 **어디에서나** 발생할 수 있기 때문에 **메서드 시그니처**에 없는 **출력**을 추가한다.
* 내외부 상태에 대한 참조
  * DateTime.Now 와 같은 **정적 속성**을 사용해 현재의 상태를 가져오고, 데이터베이스에 데이터를 질의할 수 있고, 비공개 변경 가능 필드를 참조하는 등 모두 **메서드 시그니처**에 없는 **입력**이며, 따라서 숨어있다.

메서드가 수학적 함수인지 판별하는 좋은 방법은 프로그램의 **동작**을 변경하지 않고 해당 메서드에 대한 반환 값으로 **대체**할 수 있는지 확인하는 것이다. 메서드 호출을 해당 값으로 바꾸는 것을 **참조 투명성**이라고 한다.

```cs
public int Increment(int x)
{
    return x + 1;
}
```

이 메서드는 수학적 함수다. 다음 구문은 서로 동일하다.

```cs
int y = Increment(4);
int y = 5;
```

반면에 다음 메서드는 수학적 함수가 아니다. 반환 값이 메서드의 **출력**을 나타내지 않으므로 반환 값으로 **대체할 수 없다**. 숨은 출력은 x의 변경(부작용)이다.

```cs
int x = 0;
public int Increment()
{
    x++;
    return x;
}
```

부작용은 **숨은 출력**의 가장 일반적인 유형이다. 다음 예제는 수학적 함수처럼 보이지만 실제로 그렇지 않은 `AddComment()`를 보여준다.

```cs
// 내부 상태의 수정

public Comment AddComment(string text)
{
    var comment = new Comment(text);
    _comments.Add(comment); // 부작용
    return comment;
}
```

해당 메서드는 글을 입력하고 댓글을 출력한다. 하지만 댓글 목록에 추가되었고, 이는 **메서드 시그니처**에 표현되어있지 않다.

### 6.3.2 함수형 아키텍처란?

물론 부작용이 없는 애플리케이션은 비현실적이며, 모든 정보 업데이트, 장바구니에 주문 추가 등 모든 애플리케이션이 만들어내는 것이다.

함수형 프로그래밍의 목표는 부작용 제거가 아닌 비즈니스 로직을 처리하는 코드와 **부작용을 일으키는 코드의 분리**이다. 모두 고려하면 복잡도가 배가 되고 장기적으로 유지 보수성을 방해한다. 함수형 아키텍처는 바로 이 곳에 적용된다. 부작용을 **비즈니스 연산 끝**으로 몰아서 **비즈니스 로직**을 **부작용**과 **분리**한다.

> 함수형 아키텍처는 **부작용**을 다루는 코드를 **최소화**하며, 순수 함수(불변) 방식으로 작성한 코드의 양을 **극대화**한다. '불변'이란 변하지 않는 것을 의미하며, 객체가 생성되면 그 **상태**는 바꿀 수 없다.

다음 두 가지 코드 유형을 구분해서 **비즈니스 로직**과 **부작용**을 구분할 수 있다.

* **결정**을 내리는 코드
  * 이 코드는 부작용이 없기 때문에 수학적 함수를 사용할 수 있다.
* 해당 결정에 따라 **작용**하는 코드
  * 수학적 함수에 의해 이뤄진 모든 **결정**을 메시지 버스로 전송된 메시지와 같이 **가시적인 부분**으로 변환한다.

결정을 내리는 코드는 '**함수형 코어**(불변 코어)'라고도 한다. 해당 결정에 따라 작용하는 코드는 **'가변 셸'**이다.

가변 셸은 함수형 코어에 **입력 데이터**를 제공하고 데이터베이스와 같은 **프로세스 외부 의존성**에 대한 부작용을 적용해 그 **결정**을 **해석**한다. 함수형 코어와 가변 셸은 다음과 같은 방식으로 협력한다.

* 가변 셸은 모든 **입력**을 **수집**한다.
* 함수형 코어는 **결정**을 **생성**한다.
* 셸은 **결정**을 **부작용**으로 **변환**한다.

두 계층을 잘 분리하려면, 가변 셸이 **의사 결정**을 추가하지 않게끔 결정을 나타내는 클래스에 **정보**가 충분히 있는지 확인해야 한다. 가변 셸은 가능한 **아무 말도 하지 않아야 하며**, 목표는 **출력 기반 테스트**로 함수형 코어를 두루 다루고 가변 셸을 훨씬 더 적은 수의 **통합 테스트**에 맡기는 것이다.

함수형 프로그래밍에서 상태는 불변이기 때문에 **캡슐화**할 필요가 없으며, 인스턴스를 **만들 때** 클래스의 **상태**를 **한번만 확인**하면 된다. 모든 데이터가 불변일 때 캡슐화가 되지 않아 생긴 문제는 모두 사라진다.

> _객체 지향 프로그래밍은 **작동 부분**을 **캡슐화**해 코드를 이해할 수 있게 한다. 함수형 프로그래밍은 **작동 부분**을 **최소화**해 코드를 이해할 수 있게 한다. - 마이클 페더스_

### 6.3.3 함수형 아키텍처와 육각형 아키텍처 비교

둘 다 **관심사 분리**라는 아이디어를 기반으로하기 때문에 비슷한 점이 많다. 육각형 아키텍처는 **도메인**과 **애플리케이션 서비스** 계층을 구별하며, 도메인은 **비즈니스 로직**에 책임이 있는 반면, 애플리케이션 서비스는 SMTP 서비스와 같이 **외부 애플리케이션과의 통신**에 책임이 있다. 이는 **결정**과 **실행**을 **분리**하는 함수형 아키텍처와 매우 유사하다.

또 다른 유사점은 **의존성 간의 단방향 흐름**이다. 육각형 아키텍처에서 도메인 계층 내 클래스는 서로에게만 의존해야 한다. 마찬가지로 함수형 아키텍처의 불변 코어는 가변 셸에 의존하지 않는다. 자급할 수 있고 외부 계층과 **격리**돼 작동한다. 이로 인해 테스트하기 쉽다. 가변 셸에서 불변 코어를 완전히 떼어내 셸이 제공하는 **입력**을 단순한 값으로 **모방**할 수 있다.

이 둘의 차이점은 **부작용**에 대한 처리이다. 함수형 아키텍처의 모든 부작용은 **불변 코어**에서 **비즈니스 연산 가장자리**로 밀어낸다. 이 가장 자리는 **가변 셸**이 처리한다. 반면 육각형 아키텍처는 **도메인 계층으로 인한 부작용**도 **문제 없다**. 모든 **수정 사항**은 **도메인 계층 내**에 있어야하며, 계층의 **경계**를 넘어선 안된다. 예를 들어 도메인은 데이터베이스에 직접 저장할 수 없지만, 상태는 변경할 수 있다. 애플리케이션 서비스에서 이를 적용한다.

## 6.4 함수형 아키텍처와 출력 기반 테스트로의 전환

함수형 아키텍처로 리팩터링하는 과정에서 두 가지 단계가 있다.

* **프로세스 외부 의존성**에서 **목**으로 변경
* **목**에서 **함수형 아키텍처**로 변경

상태 기반 테스트와 통신 기반 테스트를 출력 기반 테스트로 리팩터링 할 것이다.

### 6.4.1 감사 시스템 소개

조직의 모든 방문자를 추적하는 시스템이다. 텍스트 파일을 기반 저장소로 사용하며, 가장 최근 파일의 마지막 줄에 방문자의 이름과 방문 시간을 추가한다. 파일당 최대 항목을 도달하면 인덱스를 증가시켜 새 파일을 작성한다.

```cs
// 감사 시스템의 초기 구현

public class AuditManager
{
    private readonly int _masEntriesPerFile;
    private readonly string _directoryName;
    
    public AuditManager(int maxEntriesPerFile, string directoryName)
    {
        _maxEntriesPerFile = maxEntriesPerFile;
        _directoryName = directoryName;
    }
    
    public void AddRecord(string visitorName, DateTime timeOfVisit)
    {
        string[] filePaths = Directory.GetFiles(_directoryName);
        (int index, string path)[] sorted = SortByIndex(filePaths);
        
        string newRecord = visitor + ';' + timeOfVisit;
        
        if (sorted.Length == 0)
        {
            string newFile = Path.Combine(_directoryName, "audit_1.txt");
            File.WriteAllText(newFile, newRecord);
            return;
        }
        
        (int currentFileIndex, string currentFilePath) = sorted.Last();
        List<string> lines = File.ReadAllLines(currentFilePath).ToList();
        
        if (lines.Count < _maxEntriesPerFile)
        {
            lines.Add(newRecord);
            string newContent = string.Join("\r\r", lines);
            File.WriteAllText(currentFilePath, newContent);
        } 
        else
        {
            int newIndex = currentFileIndex + 1;
            string newName = $"audit_{newIndex}.txt";
            string newFile = Path.Combine(_directoryName, newName);
            File.WriteallText(newFile, newRecord);    
        }
    }
}
```

생성자는 파일당 최대 항목 수와 작업 디렉터리를 설정 매개변수로 받으며, 이 클래스의 공개 메서드는 `AddRecord()` 뿐이다.

* 작업 디렉터리에서 전체 파일 목록을 검색한다.
* 인덱스 별로 정렬한다. (모든 파일 이름은 인덱스를 이용한 패턴을 따른다.)
* 아직 감사 파일이 없으면 단일 레코드로 첫 번째 파일을 생성한다.
* 감사 파일이 있으면 최신 파일을 가져와서 파일의 항목 수가 한계에 도달햇는지에 따라 새 레코드를 추가하거나 파일을 생성한다.

`AuditManager`클래스는 파일 시스템과 **밀접하게 연결**되어있기 때문에 테스트하기 어렵다. 테스트 전에 파일을 올바른 위치에 배치하고, 끝나면 파일을 읽고 내용을 확인 후 삭제해야 한다. 초기 버전 테스트는 파일 시스템에 **직접적으로** 수행해야 한다.

파일 시스템은 **병목 지점**이자 **공유 의존성**이다. 또한 테스트를 느리게하며, 로컬 시스템과 빌드 서버 모두 디렉터리가 있어야 테스트를 할 수 있기 때문에 **유지 보수성**도 저하된다.

다음은 좋은 단위 테스트를 기준으로 한 감사 시스템의 초기 버전 점수이다.

|               | 초기 버전 |
| ------------- | --------- |
| 회귀 방지     | 좋음      |
| 리팩터링 내성 | 좋음      |
| 빠른 피드백   | 나쁨      |
| 유지 보수성   | 나쁨      |

이는 단위 테스트의 두 번째와 세 번째 특성을 준수하지 않으므로 **통합 테스트** 범주에 속한다.

* 단위 테스트는 단일 동작 단위를 검증하고
* 빠르게 수행하고
* 다른 테스트와 별도로 처리한다.

### 6.4.2 테스트를 파일 시스템에서 분리하기 위한 목 사용

테스트가 결합된 문제는 일반적으로 목으로 해결한다. 파일 시스템을 목으로 처리하고 감사 시스템이 파일을 수행하는 쓰기를 캡처한다.

```cs
// 파일 시스템 작업을 캡슐화한 새로운 사용자 정의 인터페이스

public interface IFileSystem
{
    string[] GetFiles(string directoryName);
    void WriteAllText(string filePath, string content);
    List<string> ReadAllLines(string filePath);
}
```

파일 시스템을 **분리**하여 **공유 의존성**을 없애고 테스트를 **독립적**으로 실행할 수 있다.

```cs
// 목을 이용한 감사 시스템의 동작 확인

[Fact]
public void A_new_file_is_created_when_the_current_file_overflows()
{
    var fileSystemMock = new Mock<IFileSystem>();
    fileSystemMock
        .Setup(x => x.GetFiles("audits"))
        .Returns(new string[]
        {
           @"audits\audit_1.txt",
           @"audits\audit_2.txt" 
        }); 
    ...
    var sut = new AuditManager(3, "audits", fileSystemMock.Object);
    
    sut.AddRecord("Alice", DateTime.Parse("2019-04-06T18:00:00)"));
    
    fileSystemMock.Verify(x => x.WriteAllText(
        @"audits\audit_3.txt",
        "Alice;2019-04-06T18:00:00"));
}
```

이는 목을 타당하게 사용하는 것이다. 애플리케이션은 **최종 사용자**가 볼 수 있는 파일을 생성한다. 따라서 파일 시스템과의 통신과 이러한 부작용(파일 변경)은 애플리케이션의 **식별할 수 있는 동작**이다.

테스트는 더 이상 파일 시스템에 접근하지 않기 때문에 더 빠르며, 유지비도 절감된다. 리팩터링을 해도 회귀 방지와 리팩터링 내성이 나빠지지 않았다.

|               | 초기 버전 | 목 사용 |
| ------------- | --------- | ------- |
| 회귀 방지     | 좋음      | 좋음    |
| 리팩터링 내성 | 좋음      | 좋음    |
| 빠른 피드백   | 나쁨      | 좋음    |
| 유지 보수성   | 나쁨      | 중간    |

하지만 더 개선할 수 있다. 이전 테스트는 복잡한 설정을 포함하며, 유지비에서 이상적이지 않다. 목이 도움을 주지만, 여전히 평이한 입출력에 의존하는 테스트만큼 읽기가 쉽지 않다.

### 6.4.3 함수형 아키텍처로 리팩터링하기

인터페이스 뒤로 **부작용**을 **숨기고** 해당 인터페이스를 주입하는 대신 **부작용**을 **클래스 외부**로 완전히 이동할 수 있다. 그러면 `AuditManager`는 파일에 수행할 작업을 둘러싼 **결정**만 책임지게 된다. 새로운 클래스인 `Persister`는 작업 디렉터리에서 파일과 해당 내용을 수집해 `AuditManager`에 준 다음, 반환 값을 파일 시스템의 **변경 사항**으로 **변환**한다.

```cs
// 리팩터링 후의 AuditManager

public class AuditManager
{
    private readonly int _maxEntriesPerFile;
    // private readonly string _directoryName;
    
    public AuditManager(int maxEntriesPerFile, 
    // string directoryName
    )
    {
        _maxEntriesPerFile = maxEntriesPerFile;
        // _directoryName = directoryName;
    }
    
    public void AddRecord(FileContent[] files, string visitorName, DateTime timeOfVisit)
    {
        // string[] filePaths = Directory.GetFiles(_directoryName);
        (int index, FileContent file)[] sorted = SortByIndex(files);
        
        string newRecord = visitor + ';' + timeOfVisit;
        
        if (sorted.Length == 0)
        {
            // string newFile = Path.Combine(_directoryName, "audit_1.txt");
            // File.WriteAllText(newFile, newRecord);
            // return;
            // 업데이트 명령 반환
            return new FileUpdate(
                "audit_1.txt", newRecord);
        }
        
        (int currentFileIndex, FileContent currentFile) = sorted.Last();
        List<string> lines = currentFile.ToList();
        
        if (lines.Count < _maxEntriesPerFile)
        {
            lines.Add(newRecord);
            string newContent = string.Join("\r\r", lines);
            // File.WriteAllText(currentFilePath, newContent);
            // 업데이트 명령 반환
            return new FileUpdate(
                currentFile.FileName, newContent);
        } 
        else
        {
            int newIndex = currentFileIndex + 1;
            string newName = $"audit_{newIndex}.txt";
            // string newFile = Path.Combine(_directoryName, newName);
            // File.WriteallText(newFile, newRecord);    
            // 업데이트 명령 반환
            return new FileUpdate(
                newName, newRecord);
        }
    }
}
```

`AuditManager`는 작업 디렉터리 경로 대신 `FileContent` 배열을 받는다. 이 클래스는 **결정**을 내리기 위해 파일 시스템에 대해 알아야 할 모든 **정보**들을 포함한다.

```cs
public class FileContent
{
    public readonly string FileName;
    public readonly string[] Lines;
    
    public FileContent(string fileName, string[] lines)
    {
        FileName = fileName;
        Lines = lines;
    }
}
```

또한 디렉터리의 파일 변경이 아닌 수행하려는 부작용에 대한 **명령**을 반환한다.

```cs
public class FileUpdate
{
    public readonly string FileName;
    public readonly string NewContent;
    
    public FileUpdate(string fileName, string newContent)
    {
        FileName = fileName;
        NewContent = newContent;
    }
}
```

```cs
// AuditManager의 결정에 영향을 받는 가변 셸

public class Persister
{
    public FileContent[] ReadDirectory(string directoryName)
    {
        return Directory
            .GetFiles(directoryName)
            .Select(x => new FileContent(
                Path.GetFileName(x),
                File.ReadAllLines(x)))
            .ToArray();
    }
    
    public void ApplyUpdate(string directoryName, FileUpdate update)
    {
        string filePath = Path.Combine(directoryName, update.FileName);
        File.WriteAllText(filePath, update.NewContent);
    }
}
```

작업 디렉터리에서 내용을 읽고 `AuditManager`에서 받은 업데이트 명령을 작업 디렉터리에 다시 수행하기만 하면 된다. 여기에 분기는 없으며 모든 복잡도는 `AuditManager`에 있다. 이것이 **비즈니스 로직**과 **부작용**의 **분리**다.

분리를 유지하려면, `FileContent`와 `FileUpdate`의 인터페이스를 프레임워크 내장된 파일 상호 작용 명령과 최대한 가깝게 둬야 한다. 파싱과 준비 모두 **함수형 코어**에서 수행하므로, **함수형 코어 외부**의 코드는 간결하게 유지된다.

`AuditManager` 와 `Persister`를 붙이려면, 육각형 아키텍처 분류 체계상 **애플리케이션 서비스**라는 또 다른 클래스가 필요하다.

```cs
// 함수형 코어와 가변 셸 붙이기

public class ApplicationService
{
    private readonly string _directoryName;
    private readonly AuditManager _auditManager;
    private readonly Persister _persister;
    
    public ApplicationService(
        string directoryName, int maxEntriesPerFile
    )
    {
        _directoryName = directoryName;
        _auditManager = new AuditManager(maxEntriesPerFile);
        _persister = new Persister();
    }
    
    public void AddRecord(string visitorName, DateTime timeOfVisit)
    {
        FileContent[] files = _persister.ReadDirectory(_directoryName);
        FileUpdate update = _auditManager.AddRecord(
            files, visitorName, timeOfVisit);
        _persister.ApplyUpdate(_directoryName, update);
    }
}
```

`ApplicationService`는 **함수형 코어**(`AuditManager`)와 **가변 셸**(`Persister`)을 붙이고, 클라이언트를 위한 **진입점**을 제공한다. 감사 시스템의 **동작**을 쉽게 확인할 수 있으며, 모든 테스트는 작업 디렉터리의 **가상 상태**를 제공하고 `AuditManager`가 내린 **결정**을 검증하는 것으로 단축됐다. 육각형 아키텍처에서 `ApplicationService`와 `Persister`는 **애플리케이션 서비스** 계층에 해당하고, `AuditManager`는 **도메인 모델**에 속한다.

```cs
// 목 없이 작성된 테스트

[Fact]
public void A_new_file_is_created_when_the_current_file_overflows()
{
    var sut = new AuditManager(3);
    var files = new FileContnet[]
    {
        new FileContent("audit_1.txt", new string[0]),
        new Filecontent("audit_2.txt", new string[]
        {
            "Peter; 2019-04-06T16:30:00",
            "Jane; 2019-04-06T16:30:00",
            "Jack; 2019-04-06T16:30:00",
        })
    };
    
    FileUpdate update = sut.AddRecord(
        files, "Alice", DateTime.Parse("2019-04-06T18:00:00"));
        
    Assert.Equal("audit_3.txt", update.FileName);
    Assert.Equal("Alice; 2019-04-06T18:00:00", update.NewContent);
}
```

이 테스트는 초기 버전의 피드백이 개선됐을 뿐 아니라 유지 보수성 지표도 향상됐다. 복잡한 목 설정 없이, **단순한 입출력**만 비교하기 때문에 가독성도 우수하다.

|               | 초기 버전 | 목 사용 | 출력 기반 |
| ------------- | --------- | ------- | --------- |
| 회귀 방지     | 좋음      | 좋음    | 좋음      |
| 리팩터링 내성 | 좋음      | 좋음    | 좋음      |
| 빠른 피드백   | 나쁨      | 좋음    | 좋음      |
| 유지 보수성   | 나쁨      | 중간    | 좋음      |

함수형 코어가 생성한 **명령**은 항상 **값**이나 **값 집합**이다. 값 내용이 일치하는 한, 두 인스턴스를 서로 **변경 가능**하기 때문에 `FileUpdate`를 **값 객체**로 전환해서 가독성을 더욱 향상시킬 수 있다.

```cs
// 압축된 검증문

Assert.Equal(
    new FileUpdate("audit_3.txt", "Alice; 2019-04-06T18:00:00"),
    update);
```

### 6.4.4 예상되는 추가 개발

감사 시스템은 간단하고 다음 세 가지 분기만 있다.

* 작업 디렉터리가 비어있는 경우 새로운 파일 작성
* 기존 파일에 새 레코드 추가
* 현재 파일의 항목 수가 한도를 초과할 때 다른 파일 작성

유스케이스는 감사 기록에 항목을 추가하는 것 뿐이다. 특정 방문자를 모두 삭제하는 등의 유스케이스가 있다면 여러 파일에 영향을 줄 수 있으므로, 새 메서드는 여러 개의 파일 명령을 반환해야 한다.

```cs
public FileUpdate[] DeleteAllMentions(
    FileContent[] files, string visitorName)
```

또한 작업 디렉터리에 빈 파일을 두지 않도록 요구한다면 삭제 항목이 마지막 항목이면, 해당 파일도 같이 제거해야 한다. 이를 구현하려면 `FileUpdate`를 `FileActions`이라는 이름으로 바꾸고, `ActionType` 열거형 필드를 추가해 **유형**을 나타낼 수 있다.

함수 아키텍처는 **오류 처리**가 더욱 간단하고 명확해진다. `FileUpdate` 클래스나 별도의 구성 요소로 **메서드 시그니처**에 오류를 포함할 수 있다.

```cs
public (FileUpdate update, Error error) AddRecord(
    FileContent[] files,
    string visitorName,
    DateTime timeOfVisit)
```

오류가 있으면 서비스는 업데이트 명령을 `Persister`에 넘기지 않고 **바로** 오류 메시지를 **전달**할 수 있다.

## 6.5 함수형 아키텍처의 단점 이해하기

항상 함수형 아키텍처를 이룰 수 있는 것은 아니며, 또한 함수형이라도 코드베이스가 커지고 **성능**에 **영향**을 미치면서 유지 보수성의 이점이 상쇄된다. 함수형 아키텍처와 관련된 **비용**과 장단점을 살펴본다.

### 6.5.1 함수형 아키텍처 적용 가능성

감사 시스템은 결정을 내리기 전에 모든 입력을 수집할 수 있었다. 하지만 의사 결정 **절차 중간 결과**에 따라 외부 의존성에서 추가 데이터를 질의 하는 경우 **숨은 입력**이 생긴다. 예를 들어 방문 횟수의 임계치를 초과하면 시스템이 데이터베이스에 있는 방문자의 접근 레벨을 확인한다고 해보자.

```cs
public FileUpdat AddRecord(
    FileContent[] files, string visitorName,
    DateTime timeOfVisit, IDatabase database
)
```

이 메서드는 수학적 함수가 될 수 없으며 더 이상 출력 기반 테스트를 적용할 수 없다. 순수 함수가 아니라면 애플리케이션은 더 이상 함수형 아키텍처를 따르지 않는다. 이러한 상황에는 두 가지 해결책이 있다.

* 애플리케이션 서비스 **전면**에서 디렉터리 내용과 방문자 **접근 레벨**을 수집할 수 있다.
* `AuditManager`에서 `IsAccessLevelCheckRequiered` 같은 새로운 메서드를 둔다. `true` 인 경우 레벨을 가져와 `AddRecord`에 전달한다.

첫 번째 방법은 **성능**이 **저하**된다. 접근 레벨이 필요 없는 경우도 **무조건 질의** 한다. 그러나 **비즈니스 로직**과 **외부 시스템**과의 **통신**을 **완전히 분리**할 수 있다.

두 번째 방법은 분리를 **다소 완화** 한다. 질의 여부에 대한 **결정**은 `AuditManager`가 아니라 **애플리케이션 서비스**로 넘어간다.

두 가지 옵션과 달리, 도메인 모델(`AuditManager`)을 데이터베이스에 의존하는 것은 좋은 생각이 아니다.

**메서드 시그니처**에 없는 의존성(`_maxEntriesPerFile`)은 인자에 없을지라도 숨긴 것은 아니다. **클래스 생성자**에 있을 수 있으며, 해당 필드는 **불변**이므로 인스턴스화할 때 **항상 동일**하다. 즉, 해당 필드는 **값**이다.

하지만 `IDatabase` 의존성은 값이 아니라 **협력자**다. 협력자는 다음 중 하나에 해당하는 의존성이다.

* 가변(상태 수정 가능)
* **아직 메모리에 있지 않은** 데이터에 대한 **프록시**(**공유 의존성**)

**프로세스 외부 의존성**에 대한 호출이 더 필요하므로 출력 기반 테스트를 사용할 수 없다.

함수형 코어는 협력자로 작동하면 안 되고, 결과인 **값**으로 작동해야 한다.

### 6.5.2 성능 단점

함수형의 성능 문제는 테스트의 성능이 아니다. 감사 시스템의 초기 버전과 목이 있는 버전 모두 작업 디렉터리에서 모든 파일을 읽지 않았지만, 최종 버전은 읽고-결정하고-실행하기 방식을 따르도록 작업 디렉터리의 **모든 파일**을 읽었다.

함수형 아키텍처의 선택은 **성능**과 코드 **유지 보수성**(제품, 테스트 코드 모두)간의 **절충**이다. 성능 영향이 눈에 띄지 않는 일부 시스템은 함수형 아키텍처를 사용해 유지 보수성을 향상 시키는 편이 낫다.

### 6.5.3 코드베이스 크기 증가

함수형 아키텍처는 **함수형 코어**와 **가변 셸** 사이를 명확하게 **분리**해야 하기 때문에, 궁극적으로 **복잡도**가 낮아지고, **유지 보수성**이 향상 되지만 **초기 코딩**이 더 필요하다. 항상 시스템의 복잡도와 중요성을 고려해 전략적으로 적용하라.

함수형 방식에서 순수성에 많은 비용이 든다면 순수성을 **따르지 말라**. 대부분 모든 도메인 모델을 불변으로 할 수 없기 때문에 출력 기반 테스트에만 의존할 수 없다. 대부분의 경우 **상태 기반과 조합**하게 되며, 통신 기반을 약간 섞어도 괜찮다. 하지만 **가능한 한 많은** 테스트를 **출력 기반 테스트**로 전환하는 것이 좋다.
