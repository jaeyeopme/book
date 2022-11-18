# Chapter 9 - 목 처리에 대한 모범 사례

목은 테스트 대상 시스템과 의존성 간의 **상호 작용**을 모방하고 검사하는 데 도움이 되는 테스트 대역이다. 또한 목은 **비관리 의존성**(**외부**에서 **식별**할 수 있는)에만 적용해야 하며, 다른 것에 적용한다면 깨지기 쉬운(리팩터링 내성이 없는) 테스트가 된다.

이 장에서는 목에 대해 **리팩터링 내성**과 **회귀 방지**를 **최대화**해서 최대 가치의 통합 테스트를 개발하는 데 도움이 되는 지침을 알아본다.

## 9.1 목의 가치를 극대화하기

비관리 의존성에만 목을 사용하게 **제한**하는 것은 목의 가치를 극대화하기 위한 첫 번째 단계이다. 이 주제에 대한 예제로 이전 장의 CRM 시스템을 샘플로 사용할  것이다. 통합 테스트를 살펴보고 목에 관해 테스트를 개선할 수 있는 방법을 알아보자.

```cs
// 사용자 컨트롤러

public class UserController
{
    private readonly Dadabase _database;
    private readonly EnventDispatcher _eventDispatcher;
    
    public UserController(
        Database database,
        IMessageBus messageBus,
        IDomainLogger domainLogger)
    {
        _database = database;
        _eventDispatcher = new EventDispatcher(
            messageBus, domainLogger);
    }
    
    public string ChangeEmail(int userId, string newEmail)
    {
        object[] userData = _database.GetUserById(userId);
        User user = UserFactory.create(userData);
        
        string error = user.CanChangeEmail();
        if (error != null)
            return error;
        
        object[] company = _database.GetCompany();
        Company company = CompanyFactory.Create(companyData);
        
        user.ChangeEmail(newEmail, company);
        
        _database.SaveCompany(company);
        _database.SaveUser(user);
        _eventDispatcher.Dispatch(user.DomainEvents);
        
        return "OK";
    }
}
```

**진단** 로깅은 없고 **지원** 로깅만 남았으며, EventDispatcher라는 새로운 클래스가 도입됐다. 이는 **도메인 모델**에서 생성된 도메인 이벤트를 **비관리** 의존성에 대한 **호출**(이전에는 컨트롤러가 수행)로 변환한다.

```cs
// EventDIspatcher

public class EventDispatcher
{
    private readonly IMessageBus = _messageBus;
    private readonly IDomainLogger = _domainLogger;
    
    public EventDispatcher(
        IMessageBus messageBus,
        IDomainLogger domainLogger)
    {
        _messageBus = messageBus;
        _domainLogger = domainLogger;
    }
    
    public void Dispatch(List<IDomainEvent> events)
    {
        foreach (IDomainEvent ev in events)
        {
            Dispatch(ev);
        }
    }
    
    private void Dispatch(IDomain ev)
    {
        switch (ev) {
            case EmailChangedEvent emailChangedEvent:
                _messageBus.SendEmailChangedMessage(
                    emailChangedEvent.UserId,
                    emailChangedEvent.NewEmail);
                break;
            
            case UseTypeChangedEvent userTypeChangedEvent:
                _domainLogger.UserTypeHasChanged(
                userTypeChangedEvent.UserId,
                userTypeChangedEvent.OldType,
                userTypeChangedEvent.NewType);
                break;
        }
    }
}
```

마지막 예제는 **통합 테스트**다. 이 테스트는 모든(관리와 비관리) **프로세스 외부 의존성**을 거친다.

```cs
// 통합 테스트

[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    // Arrange
    var db = Database(ConnectionString);
    User user = CreateUser("user@mycorp.com", UserType.Employee, db);
    CreateCompany("mycorp.com", 1, db);
    
    // Mock Setup
    var messageBusMock = new Mock<IMessageBus>();
    var loggerMock = new Mock<IDomainLogger>();
    var sut = new UserController(
        db, messageBusMock.Object, loggerMock.Object);
        
    // Act
    string result = sut.ChangeEmail(user.UserId, "new@gmail.com");
    
    // Assert
    Assert.Equal("OK", result);
    
    object[] userData = db.GetUserById(user.UserId);
    User userFromDb = UserFactory.Create(userData);
    Assert.Equal("new@gmail.com", userFromDb.Email);
    Assert.Equal(UserType.Customer, userFromDb.Type);
    
    object[] companyData = db.GetCompany();
    Company companyFromDb = CompanyFactory.Create(companyData);
    Assert.Equal(0, companyFromDb.NumberOfEmployees);
    
    // Mock Verify
    messageBusMock.Verify(
        x => x.SendEmailChangedMessage(
            user.UserId, "new@gmail.com"), Times.Once);
    loggerMock.Verify(
        x => x.UserTypeHasChanged(
            user.UserId, UserType.Employee, UserType.Customer),
         Times.Once);
}
```

먼저 IMessageBus 목에 초점을 맞춰본다.

### 9.1.1 시스템 끝에서 상호 작용 검증하기

통합 테스트에서 사용했던 목이 회귀 방지와 리팩터링 내성 측면이 이상적이지 않은 이유를 알아보자.

messageBusMock 의 문제점은 IMessageBus 인터페이스가 **시스템 끝**에 있지 않다는 것이다. 구현을 살펴보자.

```cs
// 메시지 버스

public interface IMessageBus
{
    void SendEmailChangedMessage(int userId, string newEmail);
}

public class MessageBus : IMessageBus
{
    private readonly IBus _bus;
    
    public void SendEmailChangedMessage(
        _bus.Send("Type: USER EMAIL CHANGED; " + 
        $"Id: {UserId}" + 
        $"NewEmail: {newEmali}");
}

public interface IBus
{
    void Send(string message);
}
```

IBus는 메시지 버스 SDK 라이브러리 위에 있는 **래퍼**다. 이 래퍼는 기술 세부 사항을 **캡슐화**하는 깔끔한 인터페이스이다. IMessageBus는 IBus 위에 있는 래퍼로, 도메인과 관련된 메시지를 정의한다. 이 두 개의 인터페이스를 합칠 수 있지만 어디까지나 차선책이다. 이렇게 두 가지 책임(외부 라이브러리의 **복잡성**을 숨기는 것과 모든 메시지를 두는 것)은 **분리**돼 있는 것이 좋다.

육각형 아키텍처 관점에서 보면 IBus는 컨트롤러와 메시지 버스 사이의 타입 사슬에서 **마지막 고리**이며, IMesageBus는 **중간**이다. IMessageBus 대신 IBus(**시스템 끝**에 있음) 를 **목**으로 처리하면 **회귀 방지**를 극대화할 수 있다. 회귀 방지는 테스트 중에 실행되는 **코드 양**에 대한 함수다. 마지막 타입을 목으로 처리하면 통합 테스트가 **거치는 클래스의 수**가 **증가**(IMessageBus)하기 때문에 **보호**가 향상된다.

다음은 IMessageBus에서 IBus로 대상을 바꾼 후의 통합 테스트다.

```cs
// IBus를 대상으로 한 통합 테스트

[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    var busMock = new Mock<IBus>();
    var messageBus = new MessageBus(busMock.Object);
    var loggerMock = new Mock<IDomainLogger>();
    var sut = new UserController(db, messageBus, loggerMock.Object);   
    ...
    // 실제 메시지 검증
    busMock.Verify(
        x => x.Send(
            "Type: USER EMAIL CHANGED; " + 
            $"Id: {UserId}" + 
            $"NewEmail: {newEmali}"),
        Times.Once);
}
```

이제 IMessageBus는 구현이 **하나**뿐인 인터페이스이며, 이러한 인터페이스를 두기에 **타당한 이유**는 **목**으로 처리하기 위한 것뿐이다. IMessageBus를 더 이상 목으로 처리하지 않기 때문에 이 인터페이스를 **삭제**해야 한다.

또한 테스트가 메시지 버스(외부 시스템)로 전송된 **실제** 텍스트 메시지를 검증하는 것은 단순히 **호출**을 검증하는 것과 큰 차이가 있다. 텍스트 메시지는 **외부에서 식별**할 수 있는 유일한 **부작용**이며, 메시지를 생성하는 데 참여하는 클래스(IMessageBus)는 단지 **구현 세부 사항**일 뿐이다.

결국 테스트는 잠재적인 **거짓 양성**에 노출될 가능성이 낮아진다. 리팩터링을 하더라도 메시지 **구조**를 유지하는 한, 테스트는 실패하지 않는다. 단위 테스트에 비해 통합과 엔드 투 엔드 테스트가 **리팩터링 내성**이 우수한 것과 동일한 매커니즘이다.

비관리 의존성에 대한 호출 중 **마지막** 단계를 선택하는 것이 외부 시스템과의 **하위 호환성**을 보장하는 가장 좋은 방법이다. 하위 호환성은 **목**을 통해 달성할 수 있는 **목표**다.

### 9.1.2 목을 스파이로 대체하기

스파이는 목과 같은 목적을 수행하는 테스트 대역이다. 유일한 차이점은 스파이는 **수동**으로 생성해야 한다는 것이다.

**시스템 끝**에 있는 클래스의 경우 **스파이**가 목보다 **낫다**. 스파이는 검증 단계에서 코드를 **재사용**해 테스트 **크기**를 줄이고 **가독성**을 향상 시킨다.

```cs
// 스파이(직접 작성한 목이라고도 함)

public interface IBus
{
    void Send(string message);
}

public class BusSpy : IBus
{
    private List<string> _sentMessages =
        new List<string>();
        
    public void Send(string message)
    {
        _sentMessages.Add(message);
    }
}

public BusSpy ShouldSendNumberOfMessages(int number)
{
    Assert.Equal(number, _sentMessage,Count);
    return this;
}

public BusSpy WithEmailChangedMessage(int userId, string newEmail)
{
    string message = "Type: USER EMAIL CHANGED; " + 
            $"Id: {UserId}" + 
            $"NewEmail: {newEmali}";
            
    // 전송 메시지 검증
    Assert.Contains(
        _sentMessage, x => x == message);
    return this;
}
```

BusSpy가 제공하는 _플루언트 인터페이스_ 덕분에 메시지 버스와의 **상호 작용**을 검증하는 것이 간결해지고 표현력도 생겼다. 이 인터페이스는 여러 가지 검증을 **묶을 수** 있으므로 **응집도**가 높다.

> 플루언트 인터페이스란 에릭 에반스와 마틴 파울러가 고안한 것으로, **메서드 체이닝**을 기반으로 코드가 쉬운 영어 문장으로 보이게끔 가독성을 향상시키는 API 설계 기법이다.

사실 이는 IMessageBus를 목으로 처리했던 이전 버전과 매우 유사하다.

```cs
messageBusMock.Verify(
    x => x.SendEmailChangedMessage( // WithEmailchangedMessage(user.UserId, "new@gmail.com")과 같음
        user.UserId, "new@gmail.com",
        Times.Once); // ShouldSendNumberOfMessage(1)과 같음
)
```

BusSpy와 MessageBus는 모두 IBus의 **래퍼**이기 때문에 검증은 비슷하지만 둘 사이에는 결정적인 차이가 있다. BusSpy는 **테스트 코드**에 MessageBus는 **제품 코드**에 속한다. 검증문을 작성할 때 **제품 코드**에 **의존**하면 안되기 때문에 이 차이는 중요하다.

테스트를 **감시자**로 생각하라. 감시자는 모든 것을 **재확인**한다. 스파이에서도 마찬가지다. 메시지 **구조**가 변경될 때 알람이 생기게끔 별도의 검사점이 있는 셈이다. 반면 IMessageBus를 **목**으로 처리하면 **제품 코드**를 너무 많이 **신뢰**하게 된다.

### 9.1.3 IDomainLogger는 어떤가?

다음은 현재 통합 테스트에서의 목 검증문이다.

```cs
// 목 검증문

busSpy.ShouldSendNumberOfMessage(1)
    .WithEmailChangedMessage(
        user.UserId, "new@gmail.com");
        
loggerMock.Verify(
    x => x.UserTypeHasChanged(
        user.UserId, UserType.Employee, UserType.Customer),
    Times.Once);
```

MessageBus가 IBus 위의 래퍼인 것처럼 DomainLogger는 ILogger 위의 래퍼다. 이도 애플리케이션 **경계**에 있기 때문에 테스트를 변경해야하지 않을까?

대부분의 프로젝트에서 이렇게 대상을 바꿀 필요는 **없다**. 로거와 메시지버스는 **비관리** 의존성이기 때문에 모두 하위 호환성을 유지해야 하지만, 호환성의 **정확도**가 같은 필요는 없다. 메시지 버스를 사용하면 외부 시스템이 이러한 **변경**에 어떻게 **반응**하는지 알 수 없기 때문에 **구조**를 변경하지 않는 것이 중요하다. 하지만 로그의 정확한 구조는 그다지 중요하지 않다. 중요한 것은 로그의 **존재 유무**와 로그에 있는 **정보**다. 따라서 IDomainLogger만 목으로 처리해도 **보호** 수준은 충분하다.

## 9.2 목 처리에 대한 모범 사례

지금까지 두 가지 주요 모범 사례를 배웠다.

* **비관리** 의존성에만 목 적용하기
* **시스템 끝**에 있는 의존성에 대해 상호 작용 검증하기

이 절에서는 나머지 모범 사례를 설명한다.

* 통합 테스트에서만 목을 사용하고 단위 테스트에서는 하지 않기
* 항상 목 호출 수 확인하기
* 보유 타입만 목으로 처리하기

### 9.2.1 목은 통합 테스트만을 위한 것

**오직** 통합 테스트에서 **목**을 사용해야 한다는 지침은 비즈니스 로직과 오케스트레이션의 **분리**에서 비롯된다. 코드가 복잡하거나 프로세스 외부 의존성과 통신할 수 있지만 둘 다는 아니며, 이는 **도메인 모델**과 **컨트롤러**라는 계층 두 개로 만들어진다.

**도메인 모델**에 대한 테스트는 **단위 테스트** 범주에 속하며, **컨트롤러**는 **통합 테스트**에 속한다.

목은 **비관리** 의존성에만 해당하며 컨트롤러만 이러한 의존성을 **처리**하는 코드이기 때문에 통합 테스트에서 **컨트롤러**를 **테스트할** 때만 **목**을 적용해야 한다.

### 9.2.2 테스트당 목이 하나일 필요는 없음

테스트당 목을 하나만 두라는 지침은 목이 둘 이상인 경우 **한 번에 여러 가지**를 테스트할 가능성이 있기 때문에 2장에서 다뤘던 **기본적인 오해**(단위 테스트에서의 '단위'는 코드 단위이며, 모든 단위는 서로 격리해 테스트해야 한다는 것)에서 비롯된 또 다른 오해다. 오히려 '단위'는 '**코드**'가 아닌 '**동작**' 단위를 의미하며, 필요한 **코드의 양**과는 관계가 없다. 이는 목에도 동일하게 적용된다. **동작 단위**를 검증하는 데 필요한 **목의 수**는 관계가 **없다**.

### 9.2.3 호출 횟수 검증하기

비관리 의존성과의 통신에 관해서 다음 두 가지 모두 확인하는 것이 중요하다.

* 예상하는 호출이 있는가?
* 예상치 못한 호출은 없는가?

이 요구 사항은 다시 **비관리** 의존성과 **하위 호환성**을 지켜야 하는 데서 비롯된다. 호환성은 **양방향**이어야 하며, 테스트 대상 시스템이 다음과 같이 메시지를 전송하는지 **확인**하는 것만으로 **충분하지 않다**.

```cs
messageBusMock.Verify(
    x => x.SendEmailChangedMessage(user.UserId, "new@gmail.com"));
```

메시지가 정확히 **한 번만** 전송되는지 확인하라.

```cs
messageBusMock.Verify(
    x => x.SendEmailChangedMessage(user.UserId, "new@gmail.com"),
    Times.Once);
```

대부분 목 라이브러리는 목에 **다른 호출**이 없는지 명시적으로 **확인**하게 해준다.

```cs
messageBusMock.Verify(
    x => x.SendEmailChangedMessage(user.UserId, "new@gmail.com"),
    Times.Once);
messageBusMock.VerifyNoOtherCalls();
```

BusSpy도 이러한 기능을 구현한다.

```cs
busSpy
    .ShouldSendNumberOfMessage(1)
    .WithEmailChangedMessage(user.UserId, "new@gmail.com");
```

스파이에 있는 ShouldSendNumberOfMessage(1)이라는 확인은 Times.Once와 VerifyNoOtherCalls 검증을 모두 포함한다.

### 9.2.4 보유 타입만 목으로 처리하기

이는 스티브 프리먼과 냇 프라이스가 처음 소개했다. 이 지침에 따르면, 서드파티 라이브러리 위에 항상 **어댑터**를 작성하고 기본 타입 대신 해당 어댑터를 **목**으로 처리해야 한다. 다음은 관련된 몇 가지 주장이다.

* 서드파티 코드의 **작동 방식**에 대해 깊이 **이해**하지 못하는 경우가 많다.
* 해당 코드가 이미 **내장 인터페이스**를 제공하더라도 **목**으로 처리한 **동작**이 실제로 외부 라이브러리와 **일치**하는지 **확인**해야하기 때문에 해당 인터페이스를 **목**으로 처리하는 것은 **위험**하다.
* 서드파티 코드의 **기술 세부 사항**까지는 꼭 필요하지 않기에 어댑터는 이를 **추상화**하고, 애플리케이션 관점에서 라이브러리와의 **관계**를 정의한다.

실제로 어댑터는 코드와 외부 환경 사이의 **손상 방지** 계층으로 작동한다. 다음은 어댑터의 기능이다.

* 기본 라이브러리의 복잡성을 추상화한다.
* 라이브러리에서 필요한 기능만 노출한다.
* 프로젝트 도메인 언어를 사용해 수행할 수 있다.

CRM 프로젝트에서 IBus 인터페이스가 바로 그 목적에 부합한다. 기본 메시지 버스의 라이브러리가 깔끔한 인터페이스를 제공하더라도, 고유의 **래퍼**를 그 위에 두는 것이 좋다. 라이브러리를 업그레이드할 때 서드파티 코드가 어떻게 **변경**될지 알 수 없다.

'보유 타입을 목으로 처리하라'라는 지침은 **프로세스 내부 의존성**에 적용되지 않는다. 따라서 **인메모리 의존성**이나 **관리 의존성**을 추상화할 필요가 **없다**. 예를 들어 ORM이 외부 애플리케이션이 **볼 수 없는** 데이터베이스를 접근하는 데 사용하는 한, ORM을 추상화할 필요는 없다.

모든 라이브러리 위에 고유의 래퍼를 둘 수 있지만, **비관리 의존성** 이외의 다른 용도로는 노력을 들일 만한 가치가 **없다**.
