# Chapter 5 - 목과 테스트 취약성

목과 테스트 취약성 사이에는 깊고 불가피한 관련이 있으며, 리팩터링 내성 저하 없이 목을 사용하는 방법을 알아본다.

## 5.1 목과 스텁 구분

목은 SUT와 그 협력자 사이의 **상호 작용**을 검사할 수 있는 테스트 대역이며, 테스트 대역에는 스텁이라는 또 다른 유형이 존재한다.

### 5.5.1 테스트 대역 유형

테스트 대역은 모든 유형의 **가짜 의존성**을 말하는 포괄적인 용어 이며, **더미**, **스텁**, **스파이**, **목**, **페이크** 라는 다섯가지가 있다. 하지만 실제로는 **목**(목, 스파이)과 **스텁**(스텁, 더미, 페이크)의 두 가지 유형으로 나눌 수 있으며 세부 사항의 차이는 생성 방식에 있다. 이 두 유형의 차이점은 다음과 같다.

* 목은 **외부**로 나가는 **상호 작용**을 모방하고 검사하는 데 도움이 된다. 상호 작용은 SUT가 상태 변경을 위한 의존성을 호출하는 것에 해당한다.
* 스텁은 **내부**로 들어오는 **상호 작용**을 모방하는 데 도움이 된다. 상호 작용은 SUT가 입력 데이터를 얻기 위한 의존성을 호출하는 것에 해당한다.

### 5.1.2 도구로서의 목과 테스트 대역으로서의 목

**도구**로서의 목을 사용해 목과 스텁, 두 가지 유형의 테스트 대역을 생성할 수 있다. 도구의 목과 테스트 대역의 목을 혼동하지 않아야 한다.

### 5.1.3 스텁으로 상호 작용을 검증하지 말라

SUT에서 스텁으로의 호출은 SUT가 생성하는 **최종 결과**가 아닌 최종 결과를 **산출**하기 위한 수단이다. 스텁은 SUT가 출력을 생성하도록 입력을 제공하기 때문에 **스텁**과의 **상호 작용**을 검증하는 것은 **안티 패턴**이다.

테스트에서 거짓 양성을 피하고 리팩터링 내성을 향상 시키는 방법은 **구현 세부 사항**이 아닌 **최종 결과**를 검증하는 것뿐이다. 스텁을 **호출**하는 것은 전혀 **결과**가 아니며, 이는 SUT가 결과를 산출하는 것에 대한 **내부 구현 세부 사항**이기 때문에 호출을 검증하지 않아야 한다. **결과**가 올바르다면 어떻게 생성하는지는 중요하지 않다.

```cs
// 스텁으로 상호 작용 검증

[Fact]
public void Creating_a_report()
{
    var stub = new Mock<Idatabase>();
    stub.Setup(x => x.GetNumberOfUsers()).Returns(10);
    
    var sut = new Controller(stub.Object);
    Report report = sut.CreateReport();
    
    Assert.Equal(10, report.NumberOfUsers);
    
    // 스텁으로 상호 작용 검증
    stub.Verify(
        x => x.GetNumberOfUsers(),
        Times.Once
    );
}
```

최종 결과가 아닌 사항을 검증하는 이러한 관행을 **과잉 명세**라고 하며, 이는 상호 작용을 검사할 때 가장 흔하게 발생한다. 목을 쓰면 무조건 테스트 취약성을 초래하는 것은 아니지만, **대다수**가 그렇다.

### 5.1.4 목과 스텁 함께 쓰기

때로는 목과 스텁의 특성을 모두 나타내는 테스트 대역을 만들 필요가 있다.

```cs
// 목이자 스텁인 storeMock

[Fact]
public void Purchase_failes_when_not_enough_inventory() 
{
    var storeMock = new Mock<IStore>();
    storeMock
        .Setup(x => x.HasEnoughInventory(
            Product.Shampoo, 5))
        .Returnes(false);
    
    var sut = new Customer();
    
    bool success = sut.Purchase(
        storeMock.Object, Product.Shampoo, 5);
        
    Assert.False(success);
    // SUT 에서 수행한 호출을 검사
    storeMock.Verify(
        x => x.RemoveInventory(Product.Shampoo, 5),
        Times.Never);
}
```

이 테스트에서 `storeMock`은 1. 준비된 응답을 반환하고, 2. SUT에서 수행한 메서드 호출을 검증한다. 그러나 이는 서로 **다른** 메서드다. `HasEnoughInventory()`에서 응답을 설정하고, `RemoveInventory()`에 대한 호출을 검증한다. 따라서 **스텁**과의 **상호 작용**을 검증하지 않는다.

### 5.1.5 목과 스텁은 명령과 조회에 어떻게 관련돼 있는가?

CQS 원칙에 따르면 모든 메서드는 **명령**이거나 **조회**여야 하며, 이를 혼용해서는 안된다. 명령은 **부작용(상태 변경)**을 일으키고 **반환 값이 없는** 메서드다. 조회는 그 반대로 부작용이 없고 **반환 값**이 있다. **명령**을 대체하는 것은 **목**이고 **조회**를 대체하는 것은 **스텁**이다.

> CQRS는 CQS에서 확장된 개념이다. CQS는 메서드 단위이지만 CQRS는 시스템 단위에서 분리한다.

항상 CQS원칙을 따를 수 있는 것은 아니며 예외로 `stack.Pop()` 메서드는 스택의 최상위 요소를 제거해 호출자에게 반환한다. 그래도 **가능하면** CQS원칙을 따르는게 좋다.

## 5.2 식별할 수 있는 동작과 구현 세부 사항

리팩터링 내성을 늘리기 위해 거짓 양성을 제거해야하고, 거짓 양성이 있는 주요 이유는 구현 세부 사항과의 **강결합**이다. 강결합을 피하기 위해서는 '**어떻게**'가 아니라 '**무엇**'에 중점을 둬야 한다.

### 5.2.1 식별할 수 있는 동작은 공개 API와 다르다

코드가 시스템의 식별할 수 있는 동작이라면 다음 중 하나를 해야 한다.

* 클라이언트가 목표를 달성하는 데 도움이 되는 **연산**을 노출하라.
  * 연산은 계산을 수행하거나 **부작용**을 초래하거나 둘 다 하는 메서드다.
* 클라이언트가 목표를 달성하는 데 도움이 되는 **상태**를 노출하라.
  * 상태는 시스템의 현재 상태다.

이상적으로 시스템의 **공개 API**는 **식별할 수 있는 동작**과 일치해야 하며, 모든 **구현 세부 사항**은 클라이언트가 **볼 수 없어야 한다**.

### 5.2.2 구현 세부 사항 유출: 연산의 예

해당 코드는 속성과 메서드 둘 다 `public`이다. API를 잘 설계하려면 해당 멤버가 **식별할 수 있는 동작**이어야 한다. `Name`속성만이 **식별할 수 있는 동작**의 요구 사항을 충족하기 때문에 `NormalizeName()`는 클라이언트의 목표에 직결되지 않으며, 공개 API로 유출되는 **구현 세부 사항**이다.

```cs
// 구현 세부 사항을 유출하는 클래스

public class User
{
    public string Name { get; set; }
    
    public string NormalizeName(string name)
    {
        string result = (name ?? "").Trim();
        
        if (result.Length > 50)
            return result.Substring(0, 50);
            
        return result;
    }
}

public class UserController
{
    public void RenameUser(int userId, string newName)
    {
        User user = GetUserFromDatabase(userId);
        // 호출 연산 1
        string normalizedName = user.NormalizeName(newName);
        // 호출 연산 2
        user.Name = normalizedName;
        
        SaveUserToDatabase(user);
    }
}
```

식별할 수 있는 동작(Name 속성)만 공개되있고, 구현 세부 사항(NormalizeName 메서드)은 비공개 API 뒤에 숨겨졌다. 목표 달성을 위한 호출 연산의 수가 1보다 크면 구현 세부 사항을 유출할 가능성이 있으며 리팩터링 후 2에서 1로 감소했다.  

```cs
// 잘 설계된 클래스
public class User
{
    private string _name;
    public string Name
    {
        get => _name;
        set => _name = NormalizeName(value);
    }
   ...
}

public class UserController
{
    public void RenameUser(int userId, string newName)
    {
        User user = GetUserFromDatabase(userId);
        // 호출 연산 1
        user.Name = newName;
        SaveUserToDatabase(user);
    }
}
```

### 5.2.3 잘 설계된 API와 캡슐화

캡슐화는 **불변성 위반**이라고도 하는 **모순**을 방지하는 조치다. **구현 세부 사항**을 노출하면 **불변성 위반**을 가져온다. 코드가 변경됐을 때 모순이 생기지 않도록 실수할 가능성을 최대한 없애기 위한 가장 좋은 방법은 캡슐화를 올바르게 **유지**하는 것이다.

캡슐화는 궁극적으로 단위 테스트와 동일한 목표를 달성한다.

* **구현 세부 사항**을 숨기면 클리언트에게서 **클래스 내부**를 가리기 때문에 내부를 손상시킬 위험이 적다.
* 데이터와 연산을 결합하면 해당 연산이 클래스의 **불변성**을 위반하지 않도록 한다.

### 5.2.4 구현 세부 사항 유출: 상태 예

```cs
// 구현 세부 사항으로서의 상태

public class MessageRenderer : IRenderer
{
    public IReadOnlyList<IRenderer> SubRenderers { get ;}
    
    public MessageRendere()
    {
        ...
    }
    
    public string Render(Message message)
    {
        ...    
    }
}
```

 클라이언트에게 필요한 **동작**은 `SubRenderers` 컬렉션(구현 세부 사항)이 아닌 `Render()` 뿐이다. 모든 **구현 세부 사항**을 비공개로 하면 테스트가 **식별할 수 있는 동작**을 검증하는 것 외에는 다른 선택지가 없으며, 리팩터링 내성도 자동으로 좋아진다.

## 5.3 목과 테스트 취약성 간의관계

### 5.3.1 육각형 아키텍처 정의

각 육각형은 **도메인**과 **애플리케이션 서비스**라는 두 계층으로 구성된다. 애플리케이션이 RESTful API인 경우 **모든 요청**은 **애플리케이션 서비스 계층**에 도달하며, 이 계층은 **도메인** 클래스와 **프로세스 외부 의존성** 간의 작업을 **조정**한다.

다음은 육각형 아키텍처의 세 가지 중요한 지침이다.

* 도메인 계층과 애플리케이션 서비스 계층 간의 **관심사 분리**
  * 도메인 계층은 해당 **비즈니스 로직**에 대해서만 책임을 져야하며, 애플리케이션 서비스는 **도메인** 계층과 **외부 애플리케이션** 간의 작업을 **조정**해야 한다.
* 애플리케이션 **내부 통신**
  * 애플리케이션 계층에서 도메인 계층으로 흐르는 **단방향 의존성 흐름**을 규정한다. 도메인 계층 내 클래스는 **서로에게만 의존**해야하며, **외부 환경**에서 완전히 **격리**돼야 한다.
* 애플리케이션 간의 통신
  * 아무도 도메인 계층에 **직접 접근** 할 수 없으며, 외부 애플리케이션은 애플리케이션 서비스 계층에 있는 **공통 인터페이스**로만 접근 가능하다.

### 5.3.2 시스템 내부 통신과 시스템 간 통신

일반적으로 애플리케이션에는 시스템 **내부**(클래스간) 통신과 시스템(다른 애플리케이션간) 통신이 있다. 시스템 내 통신으로 **도메인 클래스 간의 협력**은 식별할 수 있는 동작이 아니며, **구현 세부 사항**이다. 이러한 협력은 클라이언트의 목표와 직접적인 관계가 없기 때문에 이러한 협력과 결합하면 테스트가 **취약**해진다.

목을 사용하면 **외부 애플리케이션**과의 **통신 패턴**을 확인할 때 좋다. 반면 시스템 내 클래스는 리팩터링 내성이 줄어든다.

### 5.3.3 시스템 내부 통신과 시스템 간 통신의 예

SMTP 서비스에 대한 호출은 **외부 환경에서 확인 가능한 부작용**이므로 애플리케이션 전체적으로 **식별할 수 있는 동작**을 나타내며, 이는 고객의 목표와 **직접적인 연관**이 있다.

```cs
// 외부 애플리케이션과 도메인 모델 연결하기

public class CustomerController
{
    public bool Purchase(int customerId, int productId, int quantity)
    {
        Customer customer = _customerRepository.GetById(customerId);
        Product product = _productRepository.GetById(productId);
        
        bool isSuccess = customer.Purchase(
            _mainStore, product, quantity);
            
        if (isSuccess)
        {
            _emailGateway.SendReceipt(
                customer.Email, product.Name, quantity);
        }
        
        return isSuccess;
    }
}
```

SMTP 서비스에 대한 **호출**은 **리팩터링 후**에도 이러한 통신 유형이 **유지**되기 때문에 목으로 하는 것은 타당하다.

```cs
// 취약한 테스트로 이어지지 않는 목 사용

[Fact]
public void Successful_purchase()
{
    var mock = new Mock<IEmailGateWay>();
    var sut = new CustomerController(mock.Object);
    
    bool isSuccess = sut.Purchase(
        customerId: 1, productId: 2, quantity: 5);
        
    Assert.True(isSuccess);
    mock.Verify(
        x => x.SendReceipt(
            "customer@email.com", "Shampoo", 5),
            Times.Once);
}
```

```cs
// 취약한 테스트로 이어지는 목 사용

[Fact]
public void Purchase_succeed_when_enough_inventory()
{
    var storeMock = new Mock<IStore>();
    storeMock
        .Setup(x => x.HasEnouthInventory(Product.Shampoo, 5))
        .Returns(true);
    var customer = new Customer();
    
    bool success = customer.Purchase(
        storeMock.Object, Product.Shampoo, 5);

    Assert.True(success);
    storeMock.Verify(
        x => x.RemoveInventory(Product.Shampoo, 5),
        Times.Once);
}
```

Customer 클래스에서 Store 클래스의 호출은 모두 **애플리케이션 내**에 있으며, 고객의 목표 달성에 직접적인 관련이 있는 멤버는 `customer.Purchase()`와 `store.GetInventory()`이다. `RemoveInventory()`는 고객의 목표로 가는 **구현 세부 사항**이다.

## 5.4 단위 테스트의 고전파와 런던파 재고

런던파는 시스템 내 통신과 시스템 간 통신을 구분하지 않으며, 애플리케이션과 외부 시스템 간의 통신을 확인하는 것처럼 **클래스 간 통신도 확인한다**. 이는 리팩터링 내성을 떨어뜨린다.

고전파는 테스트 간의 **공유 의존성**만 교체하긴 하지만 역시 **시스템 간** 통신에 대한 처리는 **목** 사용을 **장려**하기 때문에 **이상적이지 않다**.

### 5.4.1 모든 프로세스 외부 의존성을 목으로 해야 하는 것은 아니다

프로세스 외부 의존성과 목을 설명하기 전에 2장에서 살펴봤던 의존성 유형을 살펴보자.

* 공유 의존성
  * 테스트 간에 공유하는 의존성
* 프로세스 외부 의존성
  * 프로그램의 실행 프로세스 외에 다른 프로세스를 점유하는 의존성
* 비공개 의존성
  * 공유하지 않는 모든 의존성

프로세스 외부 의존성이 애플리케이션을 통해서만 접근할 수 있으면, 이러한 의존성과의 통신은 시스템에서 **식별할 수 있는 동작**이 아니다. **외부에서 관찰할 수 없는** 프로세스 외부 의존성은 **애플리케이션의 일부**이며, 이는 **구현 세부 사항**이다. 따라서 리팩터링 후에 그대로 유지할 필요가 없으므로 **목으로 검증해서는 안된다**.

데이터베이스 같은 경우 애플리케이션을 제외하고 어떤 외부 시스템도 이 데이터베이스에 접근할 수 없기 때문에 기존 기능을 손상시키지 않는 한 클라이언트가 모르기 때문에 다른 저장 방식으로 **대체**할 수 있다.

완전한 통제권을 가진 프로세스 외부 의존성이라면 목을 사용하지마라.

### 5.4.2 목을 사용한 동작 검증

목이 **동작**을 검증한다는 사실은 대부분 그렇지 않다. 목표를 달성하고자 각 개별 클래스가 이웃 클래스와 **소통하는 방식**은 **식별할 수 있는 동작**과는 아무런 관계가 없다.

클라이언트는 도움을 청할 때 두뇌의 어떤 뉴런이 켜지는지 신경 쓰지 않는다. 목은 애플리케이션의 **경계**를 넘나드는 **상호 작용**을 검증할 때와 이러한 상호 작용의 부작용이 **외부 환경에서 관찰 가능할 때**만 동작과 관련있다.
