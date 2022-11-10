# Chapter 11 - 단위 테스트 안티 패턴

안티 패턴은 겉으로 적절한 것처럼 보이지만, 나중에 더 큰 문제로 이어지는 반복적인 문제에 대한 일반적인 해결책이다. 테스트에서 시간 처리, 비공개 메서드 단위 테스트, 코드 오염, 구체 클래스 목 처리 등 안티 패턴을 설명하며 피하는 방법을 알아본다.

## 11.1 비공개 메서드 단위 테스트

자주 나오는 주제 중 하나인데, 짧게 대답하면 '**전혀** 하지 말아야 한다.'

### 11.1.1 비공개 메서드와 테스트 취약성

테스트를 위해 비공개 메서드를 노출하는 경우 5장에서 다룬 기본 원칙 중 하나인 **식별할 수 있는 동작**만 테스트하는 것을 위반한다. 이는 테스트가 구현 세부 사항과 결합되고 리팩터링 내성이 떨어진다. 직접 테스트하는 대신 포괄적인 식별할 수 있는 동작으로 간접적으로 테스트 하는 것이 좋다.

### 11.1.2 비공개 메서드와 불필요한 커버리지

때로는 너무 복잡해서 식별할 수 있는 동작으로 테스트하기에 충분히 커버리지를 얻을 수 없는 경우가 있다. 식별할 수 있는 동작에 이미 합리적인 커버리지가 있다고 가정한다면 다음 두 가지 문제가 발생할 수 있다.

* 죽은 코드다. 테스트에서 벗어난 코드가 어디에도 사용되지 않는다면 리팩토링 후 남은 코드일 수 있다. 삭제하는 것이 좋다.
* 추상화가 누락돼 있다. 비공개 메서드가 너무 복잡하면(그래서 클래스의 공개 API를 통해 테스트하기 어렵다면) 별도의 클래스로 도출해야 하는 추상화가 누락됐다는 징후다.

#### 복잡한 비공개 메서드가 있는 클래스

```cs
public class Order
{
    private Customer _customer;
    private List<Product> _products;
    
    public string GenerateDescription()
    {
        return $"Customer name: {_customer.Name}, " +
        $"total number of products: {_products.Count}" +
        $"total price: {GetPrice()}";
    }
    
    private decimal GetPrice() // 복잡한 비공개 메서드
    {
        decimal basePrice = ...;
        decimal discounts = ...;
        decimal taxes = ...;
        return basePrice - discounts + taxes;
    }
}
```

GetPrice()는 중요한 비즈니스 로직이 있기 때문에 테스트를 철저히 해야 한다. 이 로직은 추상화가 누락됐다. 다음 예제와 같이 추상화를 별도의 클래스로 도출해서 명시적으로 작성하는 것이 좋다.

#### 복잡한 비공개 메서드 추출

```cs
public class Order
{
    private Customer _customer;
    private List<Product> _products;
    
    public string GenerateDescription()
    {
        var calc = new PriceCalculator();
        
        return $"Customer name: {_customer.Name}, " +
        $"total number of products: {_products.Count}" +
        $"total price: {calc.GetPrice(_customer, _products)}";
    }
}
```

이제 Order와 별개로 PriceCalculator를 테스트할 수 있다. PriceCalculator에는 숨은 입출력이 없기 때문에 출력 기반 스타일의 단위 테스트를 사용할 수 있다.

### 11.1.3 비공개 메서드 테스트가 타당한 경우

5장에서 다룬 코드의 공개 여부와 목적 간의 관계를 다시 살펴봐야 한다.

|        | 식별할 수 있는 동작 | 구현 세부 사항 |
| ------ | ------------------- | -------------- |
| 공개   | 좋음                | 나쁨           |
| 비공개 | 해당 없음           | 좋음           |

식별할 수 있는 동작을 공개로 하고 구현 세부 사항을 비공개로 하면 API가 잘 설계됐다고 할 수 있다. 반면, 구현 세부 사항이 유충되면 코드 캡슐화를 해친다. 메서드가 식별할 수 있는 동작이 되려면 클라이언트 코드에서 사용돼야 하므로 해당 메서드가 비공개인 경우에는 불가능하다.

비공개 메서드를 테스트 하는 것 자체는 나쁘지 않지만, 비공개 메서드가 구현 세부 사항의 **프록시**에 해당하기 때문에 나쁜 것이다. 메서드가 비공개이면서 식별할 수 있는 동작인 경우는 드물다.

신용 조회를 관리하는 시스템을 예로 들어보자. 하루에 한 번 데이터 데이터베이스에 직접 대량의 새로운 조회를 로드한다. 관리자는 그 조회를 하나씩 검토하고 승인 여부를 결정한다.

#### 비공개 생성자가 있는 클래스

```cs
public class Inquiry
{
    public bool IsApproved { get; private set; }
    public DateTime? TimeApproved { get; private set; }
    
    private Inquiry(
        bool isApproved, DateTime? timeApproved)
    {
        if (isApproved && !timeApproved.HasValue)
            throw new Exception();
        
        IsApproved = isApproved;
        TimeApproved = timeApproved;
    }
    
    public void Approve(DateTime now)
    {
        if (isApproved)
            return;
        
        IsApproved = true;
        TimeApproved = now;
    }
}
```

ORM을 사용하기 때문에 공개 생성자가 필요하지 않으며, 비공개 생성자로 작동한다. 그리고 시스템은 이러한 조회를 만들어낼 책임도 없기 때문에 생성자가 필요하지 않다.

객체를 인스턴스화할 수 없다면 Iquiry 클래스를 어떻게 테스트 할까? 승인 로직은 분명히 중요하기 때문에 단위 테스트를 거쳐야 한다. 그러나 다른 한편으로 생성자를 공개하는 것은 규칙을 위반하게 된다.

Iquiry 생성자는 비공개이면서 식별할 수 있는 동작인 메서드다. 이 생성자는 ORM과의 계약을 지키며, ORM은 생성자 없이 데이터베이스 조회를 복원할 수 없기 때문에 비공개라도 중요하다.

따라서 이 경우 Inquiry 생성자를 공개한다고 해서 **테스트가 쉽게 깨지지 않는다**. 생성자가 캡슐화를 지키는 데 필요한 전제 조건이 모두 포함돼 있는지 확인하라. 예제에서의 전제 조건은 승인된 모든 조회 승인 시간이 있도록 요구하는 것이다. 또한 클래스의 공개 API 노출 영역을 작게 하려면 ORM처럼 테스트에서 **리플렉션**을 통해 Inquiry를 인스턴스화할 수 있다.

## 11.2 비공개 상태 노출

또 다른 안티 패턴으로 단위 테스트 목적으로만 비공개 상태를 노출하는 것이다. 이는 비공개로 지켜야하는 상태를 노출하지 말고 식별할 수 있는 동작만 테스트하라는 비공개 메서드 지침과 같다. 다음 예제를 보자.

#### 11.4 비공개 상태가 있는 클래스

```cs
public class Customer
{
    private CUstomerStatus _status = 
        CustomerStatus.Regular;
    
    public void Promote()
    {
        _status = CustomerStatus.Preferred;
    }

    public decimal GetDiscount()
    {
        return _status == CustomerStatus.Preferred ? 0.05m : 0m;
    }
}

public enum CustomerStatus
{
    Regular,
    Preferred
}
```

고객은 각각 Regular 상태로 생성된 후에 Preferred로 업그레이드 할 수 있으며, 업그레이드하면 모든 항목에 5% 할인을 받는다. Promote() 메서드는 어떻게 테스트 하겠는가? 이 메서드의 부작용은 _status 필드의 변경이지만, 필드는 **비공개**이기 때문에 테스트할 수 없다. 결국 Promote() 호출의 궁극적인 목표는 상태 변경이기 때문에 필드를 공개하는것은 어떤가?

그러나 이는 안티 패턴일 것이다. 특별한 권한이 따로 있어서는 안된다. _status 필드는 제품 코드에 **숨어있기 때문에** SUT의 식별할 수 있는 동작이 아니다. 해당 필드를 공개하면 테스트가 구현 세부 사항에 결합된다.

Promote() 메서드를 테스트하는 방법은 제품 코드가 이 클래스를 어떻게 사용하는지 대신 살펴보는 것이다. 제품 코드는 고객의 상태를 신경쓰지 않는다. 관심을 갖는 정보는 승격 후 고객이 받는 할인뿐이며, 이것이 테스트에서 확인해야 할 사항이다. 그리고 다음 사항을 확인해야 한다.

* 새로 생성된 고객은 할인이 없다.
* 업그레이드 시 5% 할인율을 적용한다.

제품 코드가 고객 상태 필드를 사용하기 시작하면 공식적으로 SUT의 식별할 수 있는 동작이 되기 때문에 테스트에서 해당 필드를 결합할 수도 있다.

## 11.3 테스트로 유출된 도메인 지식

도메인 지식을 테스트로 유출하는 것은 또 하나의 흔한 안티 패턴이며, 보통 복잡한 알고리즘을 다루는 테스트에서 일어난다. 계산 알고리즘을 예로 들어보자.

```cs
public static class Calculator
{
    public static int Add(int value1, int value2)
    {
        return value1 + value2;
    }
}
```

다음 예제는 잘못된 테스트 방법을 보여준다.

#### 알고리즘 구현 유출

```cs
public class CalculatorTests
{
    [Fact]
    public void Adding_two_numbers()
    {
        int value1 = 1;
        int value2 = 3;
        int expected = value1 + value2; // 유출

        int actual = Calculator.Add(value1, value2);

        Assert.Equal(expected, actual);
    }
}
```

추가 비용 없이 몇 가지 테스트 사례를 추가로 처리하도록 테스트를 매개변수화할 수도 있다.

#### 같은 테스트의 매개변수화 버전

```cs
public class CalculatorTests
{
    [Theory]
    [InlineData(1, 3)]
    [InlineData(11, 33)]
    [InlineData(100, 500)]
    public void Adding_two_numbers(int value1, int value2)
    {
        int expected = value1 + value2; // 유출

        int actual = Calculator.Add(value1, value2);

        Assert.Equal(expected, actual);
    } 
}
```

두 예제 다 처음에는 괜찮아 보이지만, 사실은 안티 패턴이다. 이러한 테스트는 제품 코드에서 알고리즘 구현을 **복사**했다. 준비 부분에 해당 알고리즘을 다시 구현하는 것 외에는 아무것도 하지 않는다. 단순히 제품 코드에서 **복사-붙여넣기**를 할 뿐이다. 이러한 테스트는 구현 세부 사항과 결합되는 또 다른 예이며, 리팩터링 내성 지표에서 거의 0점을 받게 되고 결국 가치가 없다. 알고리즘 변경으로 인해 테스트가 실패하면 개발 팀은 원인을 파악하려고 노력하지 않으며, 해당 알고리즘의 새 버전을 테스트에 복사할 가능성이 높다.

테스트를 작성할 때 특정 구현을 암시하지 마라. 알고리즘을 복제하는 대신 다음 예제와 같이 결과를 하드코딩하라.

#### 도메인 지식이 없는 테스트

```cs
public class CalculatorTests
{
    [Theory]
    [InlineData(1, 3, 4)]
    [InlineData(11, 33, 44)]
    [InlineData(100, 500, 600)]
    public void Adding_two_numbers(int value1, int value2, int expected)
    {
        int actual = Calculator.Add(value1, value2);

        Assert.Equal(expected, actual);
    }
}
```

**단위 테스트에서는 예상 결과를 하드코딩하는 것이 좋다**. 하드코딩된 값의 중요한 부분은 SUT가 아닌 다른 것을 사용해 미리 계산하는 것이다. 또는 레거시 애플리케이션을 리팩터링할 경우 레거시 코드가 이러한 결과를 생성하도록 한 후 테스트 예상 값으로 사용할 수 있다.

## 11.4 코드 오염

다음 안티 패턴은 *코드 오염*이다. 코드 오염은 종종 다양한 유형의 스위치 형태를 취한다. 로거를 예로 들어보자.

> 코드 오염이란 테스트에만 필요한 제품 코드를 추가하는 것이다.

#### 불 스위치가 있는 로거

```cs
public class Logger
{
    private readonly bool _isTestEnvironment;

    public Logger(bool isTestEnvironment) // 스위치
    {
        _isTestEnvironment = isTestEnvironment;
    }

    pubilc void Log(string text)
    {
        if (_isTestEnvironment) // 스위치
            return;

        ...
    }
}

public class Controller
{
    public void SomeMethod(Logger logger)
    {
        logger.Log("someMethod 호출");
    }
}
```

이 예제의 Logger는 실행되는 환경의 여부를 나타내는 생성자 매개변수가 있으며, 환경에 따라 다르게 동작한다. 스위치를 사용하면 다음 예제와 같이 테스트 실행 중에 로거를 비활성화할 수 있다.

#### 불 스위치를 사용한 테스트

```cs
[Fact]
public void Some_test()
{
    var logger = new Logger(true); // 테스트 환경 스위치
    var sut = new Controller();

    sut.SomeMethod(logger);
}
```

코드 오염의 문제는 테스트와 제품 코드가 혼재돼 유지비가 증가하는 것이다. 이를 방지하려면 테스트 코드를 제품 코드베이스와 분리해야 한다.

Logger의 예제에서는 ILogger 인터페이스를 도입해 두 가지 구현을 생성한다. 하나는 운영을 위한 진짜 구현체이고, 다른 하나는 테스트를 목적으로 한 가짜 구현체다. 그리고 다음 예제와 같이 구체 클래스 대신 인터페이스를 받도록 컨트롤러에서 대상을 다시 지정한다.

#### 스위치가 없는 버전

```cs
public interface ILogger
{
    void Log(string text);
}

public class Logger : ILogger // 제품 코드
{
    ...
}

public class FakeLogger : ILogger // 테스트 코드
{
    ...
}

public class Controller
{
    public void SomeMethod(ILogger logger)
    {
        logger.log("SomeMethod 호출");
    }
}
```

ILogger는 제품 코드베이스에 있지만 테스트에만 필요한 코드 오염의 한 형태이다. 하지만 처음 Logger를 구현한 것과 달리 운영 목적으로 사용하지 않는 코드 경로를 잘못 호출할 일이 없다. 인터페이스에는 ~~코드가 없기 때문에 버그가 있을 수 없으며~~, 불 스위치와 달리 잠재적인 버그에 대한 노출 영역을 늘리지 않는다.

## 11.5 구체 클래스를 목으로 처리하기

인터페이스가 아닌 구체 클래스를 대신 목으로 처리해서 본래 클래스의 기능 일부를 보존하며, 이는 때때로 유용할 수 있다. 그러나 이는 단일 책임 원칙을 위배하는 중대한 단점이 있다. 다음 예제를 통해 알아보자.

#### 통계를 계산하는 클래스

```cs
public class StatisticsCalculator
{
    public (double totalWeight, double totalCost) Calculator(
        int customerId)
    {
        List<DeliveryRecord> records = GetDeliveries(customerId);
        double totalWeight = records.Sum(x => x.Weight);
        double totalCost = records.Sum(x => x.Cost);

        return (totalWeight * totalCost);
    }

    public List<DeliveryRecord> GetDeliveries(int customerId)
    {
        // 프로세스 외부 의존성 호출
    }
}
```

이 클래스는 외부 서비스(GetDeliveries)에서 검색한 배달 목록을 기반으로 계산한다. 다음 예제와 같이 StatisticsCalculator를 사용하는 컨트롤러가 있다고 하자.

#### StatisticsCalculator를 사용하는 컨트롤러

```cs
public class CustomerController 
{
    private readonly StatisticsCalculator _calculator;

    public CustomerController(StatisticsCalculator calculator)
    {
        _calculator = calculator;
    }

    public string GetStatistics(int customerId)
    {
        (double totalWeight, double totalCost) = _calculator
            .Calculator(customerId);

        return 
            $"Total weight delivered: {totalWeight}. " +
            $"Total cost: {totalCost}";
    }
}
```

이 컨트롤러의 테스트는 실제 StatisticsCalculator 인스턴스를 넣을 수 없다. 이 인스턴스는 **비관리 프로세스 외부 의존성**을 참조하기 때문이다. 비관리 의존성은 스텁으로 대체해야하며, 동시에 StatisticsCalculator 클래스는 중요한 계산 기능이 있으므로 교체하면 안된다.

이 딜레마를 극복하는 한 가지 방법은 StatisticsCalculator 클래스를 목으로 처리하고 **GetDeliveries() 메서드만 재정의**하는 것이다. 이 다음 예제와 같이 가상으로 만들면 가능하다.

#### 구체 클래스를 목으로 처리하는 테스트

```cs
[Fact]
public void Customer_with_no_deliveries()
{
    // Arrange
    var stub = new Mock<StatisticsCalculator> { CallBase = true };
    stub.Setup(x => x.GetDeliveries(1)) // 반드시 가상으로 돼 있어야 함
        .Returns(new List<DeliveryRecord>());
    var sut = new CustomerController(stub.Object);

    // Act
    string result = sut.GetStatistics(1);

    // Assert
    Assert.Equal("Total weight delivered: 0. Total cost: 0", result);
}
```
CallBase = true 설정은 명시적으로 재정의하지 않는 한 목이 기초(추상) 클래스의 동작을 유지하도록 한다. 이 방식으로 클래스의 일부만 대체하고 나머지는 그대로 유지할 수 있다. 앞서 언급했듯이 이는 안티 패턴(단일 책임 원칙 위반)이다.

StatisticsCalculator에는 비관리 의존성과 통신하는 책임과 통계를 계산하는 책임이 서로 관련이 없음에도 결합돼 있다. StatisticsCalculator를 목으로 처리하는 대신 다음 예제와 같이 이 **클래스를 둘로 나눈다**.

#### StatisticsCalculator를 두 클래스로 나누기

```cs
public class DeliveryGateway : IDeliveryGateway 
{
    public List<DeliveryRecord> GetDeliveries(int customerId)
    {
        // 프로세스 외부 의존성 호출
    }
}

public class StatisticsCalculator
{
    public (double totalWeight, double totalCost) Calculator(
        List<DeliveryRecord> records)
    {
        double totalWeight = records.Sum(x => x.Weight);
        double totalCost = records.Sum(x => x.Cost);

        return (totalWeight + totalCost);
    } 
}
```

다음 예제는 리팩터링 후의 컨트롤러다.

#### 리팩터링 후의 컨트롤러

```cs
public class CustomerController
{
    private readonly StatisticsCalculator _calculator;
    private readonly IDeliveryGateway _gateway;

    public CustomerController(
        StatisticsCalculator calculator,
        IDeliveryGateway gateway)
    {
        _calculator = calculator;
        _gateway = gateway;
    }

    public string GetStatistics(int customerId)
    {
        var records = _gateway.GetDeliveries(customerId);
        (double totalWeight, double totalCost) = _calculator
            .Calculate(records);

        return 
            $"Total weight delivered: {totalWeight}. " +
            $"Total cost: {totalCost}";
    }
}
```

비관리 의존성과 통신하는 책임은 이제 DeliveryGateway로 넘어갔다. 게이트웨이 인터페이스가 있으므로 구체 클래스 대신 인터페이스를 목에 사용할 수 있다. 리팩터리 후의 컨트롤러 예제는 **험블 객체 디자인 패턴**의 실제 예다.

## 11.6 시간 처리하기

시간에 따라 달라지는 기능을 테스트하면 거짓 양성이 발생할 수 있다. 실행 단계의 시간과 검증 단계의 시간이 다를 수 있다. 이 의존성을 안정화하는 데는 세 가지 방법이 있으며, 그 중 하나는 안티 패턴이고, 나머지 두 가지 중 바람직한 방법이 있다.

### 11.6.1 앰비언트 컨텍스트로서의 시간

첫 번째 방법은 앰비언트 컨텍스트 패턴을 사용하는 것이다. 8장의 로거 테스트에서 살펴본 적이 있다. 앰비언트 컨텍스트는 프레임워크의 내장 DateTime.Now 대신 다음 예제와 같이 코드에서 사용할 수 있는 사용자 정의 클래스에 해당한다.

#### 앰비언트 컨텍스트로서의 현재 날짜와 시간

```cs
public static class DateTimeServer
{
    private static Func<DateTime> _func;
    public static DateTime Now => _func();

    public static void Init(Func<DateTime> func)
    {
        _func = func;
    }
}
```

로거 기능과 마찬가지로 시간을 앰비언트 컨텍스트로 사용하는 것도 **안티 패턴**이다. 이는 제품 코드를 오염시키고 테스트를 더 어렵게 한다. 또한 정적 필드는 테스트 간에 공유하는 의존성을 도이배 해당 테스트를 통합 테스트 영역으로 전환한다.

### 11.6.2 명시적 의존성으로서의 시간

더 나은 방법으로 다음 예제와 같이 서비스 또는 일반 값으로 시간 의존성을 명시적으로 주입하는 것이 있다.

#### 명시적 의존성으로서의 현재 날짜와 시간

```cs
public interface IDateTimeServer
{
    DateTime Now { get; }
}

public class DateTimeServer : IDateTimeServer
{
    public DateTime Now => DateTime.Now;
}

public class InquiryController
{
    private readonly IDateTimeServer _dateTimeServer;

    public InquiryController(
        IDateTimeServer dateTimeServer) // 시간을 서비스로 주입
    {
        _dateTimeServer = dateTimeServer;
    }

    public void ApproveInquiry(int id)
    {
        Inquiry inquiry = GetById(id);

        inquiry.Approve(_dateTimeServer.Now); // 시간을 값으로 주입
        SaveInquiry(inquiry);
    }
}
```

시간을 서비스로 주입하는 것보다 값으로 주입하는 것이 더 낫다. 제품 코드에서 일반 값으로 작업하는 것이 더 쉽고, 테스트에서 해당 값을 스텁으로 처리하기도 더 쉽다.

의존성 주입 프레임워크가 값 객체와는 잘 어울리지 않기 때문에 시간을 항상 일반 값으로 주입할 수는 없을 것이다. 컨트롤러가 생성자에 DateTimeServer(서비스)를 받지만, 이후에는 Inquiry 도메인 클래스에 DateTime 값을 전달하는 것처럼 비즈니스 연산을 시작할 때 서비스로 시간을 주입받고, **나머지 연산**에서 값을 전달하는 것이 좋다.
