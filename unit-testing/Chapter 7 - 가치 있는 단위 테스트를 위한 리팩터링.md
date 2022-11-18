# Chapter 7 - 가치 있는 단위 테스트를 위한 리팩터링

단위 테스트와 기반 코드는 서로 얽혀 있기 때문에 가치 있는 테스트를 위해 코드 베이스에 노력을 기울여야한다.

## 7.1 리팩터링할 코드 식별하기

테스트 스위트를 크게 개선하려면 기반 코드를 **리팩터링**해야 한다. 리팩터링 방향을 설명하고자 코드를 네 가지 유형으로 분류하는 방법을 소개한다.

### 7.1.1 코드의 네 가지 유형

모든 제품 코드는 2차원으로 분류할 수 있다.

* 복잡도 또는 도메인 유의성
* 협력자 수

코드 복잡도는 코드의 **의사 결정**(분기) 지점 수로 정의한다.

> 순환 복잡도란 컴퓨터 공학에서 코드 복잡도를 설명하는 용어이며, 1 + <분기점 수> 로 계산한다.

도메인 유의성은 코드가 문제 도메인에 얼마나 **의미** 있는지를 나타낸다. 일반적으로 **도메인** 계층은 도메인 유의성이 높다. 복잡한 코드와 도메인 유의성을 갖는 코드는 **회귀 방지**에 뛰어나기 때문에 단위 테스트에서 가장 이롭다.

두 번째는 클래스 또는 메서드가 가진 **협력자 수**다. 협력자는 가변 의존성이거나 프로세스 외부 의존성 또는 둘 다이다. 협력자가 많을수록 테스트도 커지며, 유지 보수 비용이 많이 든다.

코드 복잡도, 도메인 유의성, 협력자 수의 조합으로 네 가지 코드 유형을 볼 수 있다.

* 도메인 모델과 알고리즘
  * 보통 복잡한 코드는 도메인 모델이지만 100%는 아니며, 문제 도메인과 직접적으로 관련이 없는 복잡한 알고리즘이 있을 수 있다.
* 간단한 코드
  * 매개변수가 없는 생성자와 한 줄 속성 등이 있다. 협력자가 있는 경우가 거의 없고 복잡도나 도메인 유의성도 거의 없다.
* 컨트롤러
  * 복잡하거나 비즈니스에 중요한 작업을 하는 것이 아니라 도메인 클래스와 외부 애플리케이션 같은 다른 구성 요소의 작업을 조정한다.
* 지나치게 복잡한 코드
  * 이러한 코드는 협력자가 많으며 복잡하거나 중요하다. 한 가지 예로 덩치가 큰(복잡한 작업을 위임하지 않고 모든 것을 **스스로**하는) 컨트롤러가 있다.

**도메인** 모델 및 **알고리즘**을 단위 테스트하면 노력 대비 **가장 이롭다**. 간단한 코드는 테스트 가치가 0에 가깝기 때문에 테스트할 필요가 **전혀** 없으며, 컨트롤러의 경우 포괄적인 **통합 테스트의 일부**로서 **간단히** 테스트해야 한다.

가장 문제가 되는 유형은 지나치게 복잡한 코드다. 이 장에서는 어떻게 문제를 **우회**할 수 있는지에 초점을 맞춘다. 때때로 실제 구현이 까다로울 수 있지만, **알고리즘**과 **컨트롤러**라는 두 부분으로 나누는 것이 일반적이다.

지나치게 복잡한 코드를 피하고 **도메인 모델**과 **알고리즘**만 단위 테스트 하는 것은 가치 있고 유지 보수가 쉬운 테스트 스위트로 가는 길이지만 이를 **목표**로 해서는 **안되며**, 목표는 각각의 테스트가 **프로젝트 가치**를 높이는 테스트 스위트다. 테스트 스위트 크기를 부풀리지 말고 모든 테스트를 **리팩터링**하거나 **제거**하라. **좋지 않은 테스트를 작성하는 것보다는 전혀 작성하지 않는게 낫다.**

### 7.1.2 험블 객체 패턴을 사용해 지나치게 복잡한 코드 분할하기

험블 객체 패턴은 지나치게 **복잡한** 코드에서 **로직**을 **추출**해 코드를 테스트할 필요가 없도록 간단하게 만든다. 추출된 로직은 테스트하기 어려운 의존성에서 **분리**된 다른 클래스로 이동한다.

테스트 대상 코드의 로직을 테스트하려면, **테스트 가능한 부분**을 **추출**해야 한다. 결과적으로 코드는 테스트 가능한 부분을 둘러싼 **얇은 험블 래퍼**가 된다. 험블 래퍼가 테스트하기 어려운 의존성과 새로 추출된 구성 요소를 **붙이지만**, **자체적인 로직**은 거의 또는 **전혀 없기** 때문에 테스트할 필요가 없다.

육각형과 함수형 아키텍처 모두 이 패턴을 구현한다. **비즈니스 로직**과 **프로세스 외부 의존성**과의 통신을 **분리**한다. **함수형 코어**와 **도메인 계층**은 **도메인 모델** 및 **알고리즘**에 속하며, 협력자가 거의 없고 복잡도와 도메인 유의성이 높다. **가변 셸** 및 **애플리케이션 서비스** 계층은 **컨트롤러**에 속한다.

험블 객체 패턴을 보는 다른 방법은 **SRP**를 지키는 것이다. 이는 각 클래스가 **단일한 책임**만 가져야 한다는 원칙이며, 그러한 책임 중 하나로 늘 비즈니스 로직이 있는데, 이 패턴을 적용하면 비즈니스 로직을 **거의 모든 것**과 **분리**할 수 있다.

## 7.2 가치 있는 단위 테스트를 위한 리팩터링하기

이 절에서 험블 객체 패턴을 통해 복잡한 코드를 알고리즘과 컨트롤러로 나누는 예제를 살펴본다.

### 7.2.1 고객 관리 시스템 소개

사용자 등록을 처리하는 고객 관리 시스템(CRM)이며, 사용자는 데이터베이스에 저장된다. 시스템은 사용자 이메일 변경이라는 하나의 유스케이스만 지원하며, 세 가지 비즈니스 규칙이 있다.

* 사용자 이메일이 회사 도메인에 속한 경우 해당 사용자는 직원으로 표시된다. 그렇지 않으면 고객이다.
* 시스템은 회사의 직원 수를 추적해야 한다. 사용자는 유형이 직원에서 고객으로, 또는 그 반대로 변경되면 직원 수도 변경돼야 한다.
* 이메일이 변경되면 메시지버스로 메시지를 보내 외부 시스템에 알려야 한다.

```cs
// CRM 시스템의 초기 구현

public class User
{
    public int UserId { get; private set;}
    public string Email { get; private set;}
    public UserType type { get; private set;}
    
    public void ChangeEmail(int userId, string newEmail)
    {
        object[] data = Database.GetUserById(userId);
        UserId = userId;
        Email = (string)data[1];
        Type = (UserType)data[2];
        
        if (Email == newEmail)
            return;
            
        object[] companyData = Database.GetCompany(); // 도메인과 직원 수 검색
        string companyDomainName = (string)companyData[0];
        int numberOfEmployees = (int)companyData[1];
        
        string emailDomain = newEmail.Split('@')[1];
        bool isEmailCorparate = emailDomain == companyDomainName;
        UserType newType = isEmailCorporate // 도메인에 따라 유형 설정
            ? UserType.Employee
            : UserType.Customer;
        
        if (Type != newType)
        {
            int delta = newType == UserType.Employee ? 1 : -1;
            int newNumber = numberOfEmployees + delta;
            Database.SaveCompany(newNumber); // 직원 수 업데이트
        }
        
        Email = newEmail;
        Type = newType;
        
        Database.SaveUser(this); // 저장
        MessageBus.SendEmailChangeMessage(UserId, newEmail); // 알림 전송
    }
}

public enum UserType
{
    Customer = 1,
    Employee = 2
}
```

`ChangeEmail()`은 애플리케이션의 **핵심 비즈니스 로직**이므로, 이 클래스는 복잡도와 도메인 유의성 측면에서 점수가 높다.

반면 User 클래스는 네 개의 의존성이 있으며 그 중 두 개는 **명시적**이고 나머지는 **암시적**이다. 명시적 의존성은 `userId`와 `newEmail` 인수다. 이 둘은 **값**이므로 협력자 수에는 포함되지 않는다. 암시적인 것은 `Database`와 `MessageBus`이며, 이 둘은 **프로세스 외부 협력자**다. 도메인 유의성이 높은 코드에서 프로세스 외부 협력자는 사용하면 **안된다**. 따라서 User 클래스는 **지나치게 복잡한 코드**다.

**도메인** 클래스 **스스로** 데이터베이스를 검색하고 다시 저장하는 방식을 **활성 레코드 패턴**이라고 한다. **비즈니스 로직**과 **프로세스 외부 의존성**과의 통신 사이에 **분리**가 **없기 때문**에 코드 베이스가 커지면 **확장**하지 **못하는** 경우가 많다.

### 7.2.2 1단계: 암시적 의존성을 명시적으로 만들기

일반적인 방법이다. 즉, 데이터베이스와 메시지 버스에 대한 **인터페이스**를 두고, User에 주입한 후 테스트에서 **목**으로 **처리**한다. 그러나 이 방법은 도움이 되지만 **충분하지는 않다**.

프로세스 외부 의존성이든 인터페이스든 상관 없다. 해당 의존성은 **여전히 프로세스 외부**에 있으며, 아직 메모리에 데이터가 없는 **프록시 형태**이다. 목 사용은 테스트 유지비가 증가하며, 목을 데이터베이스 의존성에 사용하면 **테스트 취약성**을 야기한다.

결국 도메인 모델은 **직접적이든 간접적(인터페이스)이든** 프로세스 **외부 협력자**에게 **의존하지 않는 것**이 훨씬 깔끔하다. 도메인 모델은 **외부 시스템**과의 **통신**을 **책임지지 않아야** 한다.

### 7.2.3 2단계: 애플리케이션 서비스 계층 도입

도메인 모델이 외부 시스템과 **직접 통신**하는 문제를 극복하려면 **험블 컨트롤러**(애플리케이션 서비스)로 **책임**을 **옮겨**야 한다. 도메인 모델은 일반적으로 다른 도메인 클래스나 단순 값과 같은 프로세스 내부에만 의존해야한다.

```cs
// 애플리케이션 서비스, 버전 1

public class UserController
{
    private readonly Database _database = new Database();
    private readonly MessageBus _messageBus = new MessageBus();
    
    public void ChangeEmail(int userId, string newEmail)
    {
        object[] data = _database.GetUserById(userId);
        string email = (string)data[1];
        UserType type = (UserType)data[2];
        var user = new User(userId, email, type);
        
        object[] companyData = _database.GetCompany();
        string companyDomainName = (string)companyData[0];
        int numberOfEmployees = (int)companyData[1];
        
        int newNumberOfEmployees = user.ChangeEmail(
            newEmail, companyDomainName, numberOfEmoloyees);
            
        _database.SaveCompany(newNumberOfEmployees);
        _database.SaveUser(user);
        _messageBus.SendEmailChangeMessage(userId, newEmail);
    }
}
```

User 클래스로부터 프로세스 외부 의존성과의 작업을 줄이는 데 도움이 됐다. 그러나 몇 가지 문제가 있다.

* 프로세스 외부 의존성이 주입되지 않고 **직접 인스턴스화** 한다. 이는 통합 테스트에서 문제가 된다.
* 컨트롤러는 데이터베이스에서 받은 원시 데이터를 User 인스턴스로 재구성한다. 이는 복잡한 로직이므로 애플리케이션 서비스에 속하면 안된다.
* 회사 데이터도 마찬가지로 User는 이제 업데이트된 직원 수를 반환하는데, 회사 직원 수는 특정 사용자와 관련 없다. 이 **책임**은 다른 곳에 있어야 한다.
* 컨트롤러는 새로운 이메일을 **무조건** 데이터를 수정해서 저장하고 메시지 버스에 알림을 보낸다.

```cs
// ChangeEmail 메서드 버전 1

public int ChangeEmail(string newEmail,
    string companyDomainName, int numberOfEmployees)
{
    if (Email == new Email)
        return numberOfEmployees;
        
    string emailDomain = new Email.Split('@')[1];
    bool isEmailCorporate = emailDomain == companyDomainName;
    UserType newType = isEmailCorporate
        ? UserType.Employee
        : UserType.Customer;
    
    if (Type != newType)
    {
        int delta = newType == UserType.Employee ? 1 : -1;
        int newNumber = numberOfEmployees + delta;
        numberOfEmplyees = newNumber;
    }
    
    Email = newEmail;
    Type = newType;
    
    return numberOfEmployees;
}
```

### 7.2.4 3단계: 애플리케이션 서비스 복잡도 낮추기

**ORM** 라이브러리를 사용해 데이터베이스를 도메인 모델에 매핑하면, UserController에서 재구성 로직을 옮기기에 적절한 위치가 될 수 있으며, ORM 을 사용하지 않거나 사용할 수 없다면 도메인 클래스를 인스턴스화하는 **팩토리 클래스**를 만들어 복잡도를 낮춰라.

```cs
// User 팩토리 클래스

public class UserFactory
{
    public static User Create(object[] data)
    {
        Precondition.Requires(data.length >= 3);
        
        int id = (int)data[0];
        string email = (string)data[1];
        UserType type = (UserType)data[2];
        
        return new User(id, email, type);
    }
}
```

이제 모든 협력자와 **완전히 격리**돼 있으므로 테스트가 쉬워졌다. 이는 도메인 유의성이 없으며, 사용자 이메일을 변경하려는 클라이언트의 목표와 직접적인 관련이 없다.

### 7.2.5 4단계: 새 Company 클래스 소개

컨트롤러 코드의 User에서 업데이트된 직원 수를 반환하는 부분은 **책임**을 잘못 뒀다는 신호이자 **추상화**가 없다는 신호다. 이를 해결하려면 **회사 관련 로직**과 **데이터**를 함께 **묶는** 또 다른 **도메인** 클래스인 Company가 있어야 한다.

```cs
// 도메인 계층의 새로운 클래스

public class Company
{
    public string DomainName { get; private set;}
    public int numberOfEmployees { get; private set;}
    
    public void ChangeNumberOfEmployees(int data)
    {
        Precondition.Requires(NumberOfEmployees + delta >= 0);
        NumberOfEmployees += delta;
    }
    
    public bool IsEmailCorporate(string email)
    {
        string emailDomain = email.Split('@')[1];
        return emailDomain == DomainName;
    }
}
```

이 클래스에서 `ChangeNumberOfEmployees()`와 `IsEmailCorporate()` 두 가지 메서드는 '**묻지 말고 말하라**'라는 원칙에 준수하는데 도움이 된다. User 인스턴스는 직원 수를 변경하거나 특정 이메일이 회사 이메일인지 **여부**를 파악하도록 회사에 **말하며**, 원시 데이터를 **묻지 않고** 모든 작업을 **자체적으로 수행**한다.

UserFactory 와 유사한 Company 객체의 재구성을 담당하는 CompanyFactory 클래스도 있다.

```cs
// 리팩터링 후 컨트롤러

public class UserController
{
    private readonly Database _database = new Database();
    private readonly MessageBus _messageBus = new MessageBus();
    
    public void ChangeEmail(int userId, string newEmail)
    {
        // object[] data = _database.GetUserById(userId);
        // string email = (string)data[1];
        // UserType type = (UserType)data[2];
        // var user = new User(userId, email, type);
        
        object[] userData = _database.GetUserById(userId);
        User user = UserFactory.Create(userData);
        
        // object[] companyData = _database.GetCompany();
        // string companyDomainName = (string)companyData[0];
        // int numberOfEmployees = (int)companyData[1];
        
        object[] companyData = _database.GetCompany();
        Company company = CompanyFactory.Create(companyData);
        
        // int newNumberOfEmployees = user.ChangeEmail(
        //    newEmail, companyDomainName, numberOfEmoloyees);
        
        user.ChangeEmail(newEmail, company);
            
        _database.SaveCompany(newNumberOfEmployees);
        _database.SaveUser(user);
        _messageBus.SendEmailChangeMessage(userId, newEmail);
    }
}
```

```cs
// 리팩터링 후 User

public class User
{
    public int UserId { get; private set; }
    public string Email { get; private set; }
    public UserType type { get; private set; }
    
    public void ChangeEmail(string newEmail, Company company)
    {
        if (Email == newEmail)
            return;
            
        UserType newType = company.IsEmailCorporate(newEmail)
            ? UserType.Employee
            : UserType.Customer;
        
        if (Type != newType)
        {
            int delta = newType == UserType.Employee ? 1 : -1;
            company.ChangeNumberOfEmployees(delta);
        }
        
        Email = newEmail;
        Type = newtype;
    }
}
```

회사 데이터를 처리하는 대신 Company 인스턴스를 받아, 회사 이메일인지의 결정과, 직원 수를 변경하는 작업들을 해당 인스턴스에 **위임**한다.

CRM의 **도메인** 계층(User, Company)은 **프로세스 외부 의존성**과 통신하지 않는다. **애플리케이션 서비스 계층**(UserController)이 해당 통신을 담당한다. 함수형 아키텍처와의 차이는 CRM의 도메인 모델은 **부작용**을 일으킨다. 부작용은 변경된 사용자 이메일과 직원 수의 형태로 도메인 **모델 내부**에 남아있으며, 컨트롤러가 객체를 **저장할 때만** 부작용이 도메인 모델의 경계를 넘는다.

마지막 순간까지 **부작용**이 **메모리에 남아있다는 사실**로 인해 테스트 용이성이 크게 향상 된다. 통신 기반 테스트가 아닌 **메모리에 있는** 객체의 **출력 기반 테스트**와 **상태 기반 테스트**로 모든 검증을 수행할 수 있다.

## 7.3 최적의 단위 테스트 커버리지 분석

험블 객체 패턴을 사용해 리팩터링을 마쳤으니 프로젝트의 코드들의 유형과 어떻게 테스트해야 하는지 분석해보자.

|                  | 협력자가 거의 없음                                                                                                                              | 협력자가 많음                                       |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| 복잡도와 도메인 유의성이 높음 | User - ChangeEmail(newEmail, company) </br> Company - ChangeNumberOfEmployees(delta), IsEmailCorporate(email) </br> CompanyFactory - Create(data) |                                               |
| 복잡도와 도메인 유의성이 낮음 | User 와 Company 생성자                                                                                                                      | UserController - ChangeEmail(userId, newMail)  |

### 7.3.1 도메인 계층과 유틸리티 코드 테스트하기

표 좌측 상단 테스트 메서드는 비용 편익 측면에서 **최상의 결과**를 가져다준다. 회귀 방지가 뛰어나고, 협력자가 거의 없어 유지비도 가장 낮다.

다음은 User 를 어떻게 테스트하는지에 대한 예다.

```cs
// User 테스트

[Fact]
public void Changing_email_from_non_corporate_to_corporate()
{
    var company = new Company("mycorp.com", 1);
    var sut = new User(1, "user@gmail.com", UserType.Customer);
    
    sut.ChangeEmail("new@mycorp.com", company);
    
    Assert.Equal(2, company.NumberOfEmployees);
    Assert.Equal("new@mycorp.com", sut.Email);
    Assert.Equal(UserType.Employee, sut.Type);
}
```

전체 커버리지를 달성하려면 테스트 세 개가 더 필요하다.

* `public void Changing_email_from_corporate_to_non_corporate()`
* `public void Changing_email_without_changing_user_type()`
* `public void Changing_email_to_the_same_one()`

다른 세 가지 테스트는 훨씬 짧을 것이며, **매개변수화된 테스트**로 여러 케이스를 묶을 수 있다.

```cs
// User 매개변수화 테스트

[InlineData("mycorp.com", "email@mycorp.com", true)]
[InlineData("mycorp.com", "email@gmail.com", false)]
[Theory]
public void Differentiates_a_corporate_email_from_non_corporate(
    string domain, string email, bool expectedResult)
{
    var sut = new Company(domain, 0);
    
    bool isEmailCorporate - sut.IsEmailCorporate(email);
    
    Assert.Equal(expectedResult, isEmailCorporate);
}
```

### 7.3.2 나머지 세 사분면에 대한 코드 테스트하기

User와 Company의 생성자는 복잡도가 낮고 협력자가 거의 없다. 이러한 코드는 단순해서 **노력을 들일 필요가 없으며**, 테스트는 회귀 방지가 떨어질 것이다.

### 7.3.3 전체 조건을 테스트해야 하는가?

특별한 종류의 분기점을 살펴보고 이를 테스트해야 하는지 확인해보자.

```cs
// Company 분기 메서드

public void ChangeNumberOfEmployees(int data)
{
    Precondition.Requires(NumberOfEmployees + delta >= 0);
    
    NumberOfEmployees += delta;
}
```

회사의 직원 수가 음수가 돼서는 안된다는 조건은 **예외 상황**에서만 활성화되는 **보조 장치**다. 보호 장치는 **빠르게 실패**하고 데이터베이스에서 오류가 확산하고 지속되는 것을 **방지**하기 위한 메커니즘을 제공한다. 그래서 **전제 조건**에 대한 테스트가 충분히 가치가 있는가?

일반적으로 도메인 유의성에 있는 **모든 전제 조건**을 테스트해야 한다. 음수가 되면 안된다는 것은 **요구 사항이 아닌 전제 조건**이다. 이는 Company 클래스의 **불변성**에 해당한다. 그러나 **도메인 유의성이 없는 전제 조건**을 테스트하는 데 **시간을 들이지 말라**.

```cs
// 또 다른 도메인 유의성이 없는 전제 조건

public static User Create(object[] data)
{
    Precondition.Requires(data.Length >= 3);
}
```

## 7.4 컨트롤러에서 조건부 로직 처리

조건부 로직을 처리하면서 동시에 **프로세스 외부 협력자** 없이 도메인 계층을 유지 보수하는 것은 까다롭고 **절충**이 있다. 그 절충을 살펴본다.

**비즈니스 로직**과 **오케스트레이션**의 **분리**는 다음과 같이 비즈니스 연산이 세 단계로 있을 때 가장 효과적이다.

* 저장소에서 데이터 검색
* 비즈니스 로직 실행
* 데이터를 다시 저장소에 저장

그러나 비즈니스 연산 중에 **프로세스 외부 의존성**을 참조하는 등 단계가 **명확하지 않은 경우**가 많으며, 이러한 상황에서는 세 가지 방법이 있다.

* 외부에 대한 모든 읽기와 쓰기를 **가장 자리**로 밀어낸다. 이 방법은 '읽고-결정하고-실행하기' 구조를 유지하지만 **성능**이 **저하**된다. **필요 없는 경우**에도 컨트롤러가 의존성을 호출한다.
* 도메인 모델에 **프로세스 외부 의존성**을 **주입**하고 **비즈니스 로직**이 해당 의존성을 **호출할 시점**을 **직접 결정**할 수 있게 한다.
* **의사 결정 프로세스 단계**를 더 세분화하고, 각 단계별로 **컨트롤러**가 실행하도록 한다.

문제는 다음 세 가지 특성의 균형을 맞추는 것이다.

* 도메인 모델 테스트 유의성
  * 도메인 클래스의 협력자 수와 유형에 따른 함수
* 컨트롤러 단순성
  * 의사 결정 지점(분기)의 유무
* 성능
  * 프로세스 외부 의존성에 대한 호출 수

방법들은 세 가지 특성 중 두 가지 특성만 갖는다.

* 외부에 대한 모든 읽기와 쓰기를 가장 자리로 밀어내기
  * 컨트롤러를 단순하게 하고 프로세스 외부 의존성과 도메인 모델을 분리(테스트 할 수 있도록)하지만, **성능**이 **저하**된다.
* 도메인 모델에 프로세스 외부 의존성 주입하기
  * 성능을 유지하면서 **컨트롤러**를 단순하게 하지만, 도메인 모델의 **테스트 유의성**이 떨어진다.
* 의사 결정 프로세스 단계를 더 세분화하기
  * 성능과 도메인 모델 테스트 유의성에 도움을 주지만, **컨트롤러**가 단순하지 않아 **세부 단계**를 **관리**하려면 **의사 결정 지점**이 있어야 한다.

세 가지 특성을 모두 충족하는 해법은 없다. 따라서 세 가지 중 두 가지를 선택해야 한다.

대부분 **성능**이 **매우 중요**하므로 첫 번째 방법(외부에 대한 읽기와 쓰기를 비즈니스 연산 가장 자리로 **밀어내기**)은 고려할 필요가 **없다**.

두 번째 방법(도메인 모델에 프로세스 외부 의존성 주입하기)은 대부분 코드를 지나치게 복잡한 사분면에 넣는다. 이는 CRM 초기 구현이며, 비즈니스 로직과 프로세스 외부 의존성과의 통신을 분리하지 않으면 테스트와 유지 보수가 훨씬 어려워진다.

그러면 세 번째 방법(의사 결정 프로세스 단계를 더 세분화하기)만 남게된다. 이 방식은 컨트롤러를 더 복잡하게 만들기 때문에 지나치게 복잡한 사분면에 가까워지지만, 그러나 **완화**할 수 있는 방법이 있다. 컨트롤러를 제외한 모든 복잡도를 **고려**할 수는 없지만 **관리**할 수는 있다.

### 7.4.1 CanExecute/Execute 패턴 사용

컨트롤러의 복잡도가 커지는 것을 완화하는 첫 번째 방법은 CanExecute/Execute 패턴을 사용해 **비즈니스 로직**이 **도메인 모델**에서 **컨트롤러**로 **유출**되는 것을 방지하는 것이다. CRM을 확장해보자.

이메일은 사용자가 확인할 때만 변경할 수 있고 이후에 변경을 시도하면 오류가 표시돼야 한다.

```cs
// 새 속성이 추가 된 User

public class User
{
    public int UserId { get; private set; }
    public string Email { get; private set; }
    public UserType type { get; private set; }
    // 새로 추가
    public bool IsEmailConfirmed { get; private set; }
    
    public void ChangeEmail(string newEmail, Company company)
    {
        // 검증
        if (IsEmailConfirmed)
            return "Can't change a confirmed email"
        ...
    }
}
```

이 메서드의 출력에 따라 **컨트롤러**는 오류를 반환하거나 필요한 **모든 부작용**을 낼 수 있다.

```cs
// 모든 의사 결정을 제거한 컨트롤러

public class UserController
{
    private readonly Database _database = new Database();
    private readonly MessageBus _messageBus = new MessageBus();
    
    public void ChangeEmail(int userId, string newEmail)
    {
        object[] userData = _database.GetUserById(userId);
        User user = UserFactory.Create(userData);
        
        object[] companyData = _database.GetCompany();
        Company company = CompanyFactory.Create(companyData);
        
        // 의사 결정
        string error = user.ChangeEmail(newEmail, company);
        
        // 이는 의사 결정의 프로세스가 아니므로 복잡도 증가로 간주하지 않음.
        if (error != null)
            return error;
            
        _database.SaveCompany(newNumberOfEmployees);
        _database.SaveUser(user);
        _messageBus.SendEmailChangeMessage(userId, newEmail);
    }
}
```

이 구현은 **컨트롤러**가 **의사 결정**을 하지 않지만, **성능 저하**를 **감수**해야 한다. 이메일을 확인해 변경할 수 없는 경우에도 **무조건** 데이터베이스에서 Company 인스턴스를 **검색**한다. 이는 모든 외부 읽기와 쓰기를 비즈니스 연산 끝으로 **밀어내는** 예다.

두 번째 방법은 `IsEmailConfirmed` 확인을 User(**도메인 모델**)에서 **컨트롤러**로 옮기는 것이다.

```cs
// 사용자의 이메일을 변경할지 여부를 결정하는 컨트롤러

    public void ChangeEmail(int userId, string newEmail)
    {
        object[] userData = _database.GetUserById(userId);
        User user = UserFactory.Create(userData);
        
        // User에서 이곳으로 옮긴 의사 결정
        if (user.IsEmailConfirmed)
            return "Can't change a confirmed email";        
        ...
    }
```

이러한 구현은 **성능**은 그대로 **유지** 된다. 그러나 **의사 결정 프로세스**는 두 부분으로 **나뉜다**.

* 이메일 변경 진행 여부(컨트롤러에서 수행)
* 변경 시 해야할 일(User에서 수행)

이제 **플래그**를 먼저 확인하지 않고 이메일을 변경하지만, **도메인 모델**의 **캡슐화**가 떨어진다. 이러한 **파편화**로 비즈니스 로직의 분리가 방해되고 지나치게 복잡한 위험 영역에 더 가까워진다. **파편화**를 **방지**하기 위해 User에 새 메서드(`CanChangeEmail()`)를 둬서, **전제 조건**을 둔다. 다음 예제는 CanExecute/Execute 패턴을 따르게끔 수정한 버전이다.

```cs
// CanExecute/Execute 패턴을 사용한 이메일 변경

    public string CanChangeEmail()
    {
        if (user.IsEmailConfirmed)
            return "Can't change a confirmed email";
            
       return null;
    }
    
    public void ChangeEmail(int userId, string newEmail)
    {
       Precondition.Requires(CanChangeEmail() == null);
       ...
    }
```

이 방법에는 두 가지 이점이 있다.

* **컨트롤러**는 이메일 변경 프로세스를 **알 필요**가 **없다**. 메서드를 호출해서 **연산 수행** 가능 **여부**만 확인하면 된다. 유효성 검사 모두 컨트롤러부터 **캡슐화**돼 있다.
* 메서드의 **전제 조건**이 추가돼도 **먼저 확인**하지 않으면 이메일을 **변경**할 수 없도록 **보장**한다.

이 패턴을 사용하면 도메인 계층의 **모든 결정**을 **통합**할 수 있으며, **컨트롤러**에 **의사 결정 지점**은 **없다**. 따라서 컨트롤러에 if 문이 있어도 if 문을 테스트할 필요는 **없다**. User 클래스의 **전제 조건**을 단위 테스트하면 된다.

### 7.4.2 도메인 이벤트를 사용해 도메인 모델 변경 사항 추적

애플리케이션에서 무슨 일이 일어나는지 **외부 시스템**에 **알릴 때** 도메인 모델을 현재 상태로 만든 **단계**를 **추적**하기 어렵다. 컨트롤러에 이러한 책임도 있으면 더 **복잡해진다**. 이를 피하기위해 **도메인 모델**에서 중요한 **변경 사항**을 **추적**하고 비즈니스 연산이 완료된 후 해당 **변경 사항**을 **프로세스 외부 의존성 호출**로 **변환**한다. _도메인 이벤트_로 이러한 추적을 구현할 수 있다.

> 도메인 이벤트란 애플리케이션 내에서 **도메인 전문가**에게 중요한 이벤트를 말하며, 도메인 전문가는 무엇으로 도메인 이벤트와 일반 이벤트(예: 버튼 클릭)를 구별하는지가 중요하다.

CRM에는 **메시지 버스**에 메시지를 보내 추적 사항을 알려줘야 한다. 현재 구현에는 이메일이 변경되지 않은 경우에도 메시지를 보낸다.

`ChangeEmail()`에서 이메일이 같은지 검사하는 부분을 **컨트롤러**로 옮겨서 해결하더라도, **비즈니스 로직**이 **파편화**되는 문제가 있다. 새 이메일이 이전 이메일과 동일하다면 `CanChangeEmail()`에서 오류를 반환해서는 안되기 때문에 검사하는 부분을 넣을 수도 없다.

**비즈니스 로직**을 너무 많이 **파편화**하지 않기 때문에 검사가 포함돼도 컨트롤러가 지나치게 복잡하지 않다고 생각할 수 있다. 그러나 애플리케이션이 **프로세스 외부 의존성**을 **불필요**하게 **호출**하는 것보다 **도메인 이벤트**를 사용하는 것이 복잡하게 만들지 않는 방법이다.

구현 관점에서 도메인 이벤트는 사용자의 ID와 이메일을 들 수 있다.

```cs
// 외부 시스템에 통보하는 데 필요한 데이터를 가진 클래스

// 도메인 이벤트는 이미 일어난 일들이기 때문에 과거 시제로 명명
public class EmailChangedEvent
{
    public int UserId { get; }
    public string NewEmail { get; }
}
```

User는 이메일이 변경될 때 이벤트 컬렉션을 갖게 된다.

```cs
// 이메일이 변경될 때 이벤트를 추가하는 User

public void ChangeEmail(string newEmail, Company company)
{
    ...
    EmailChangedEvents.Add(
        new EmailChangedEvent(UserId, newEmail));
}
```

컨트롤러는 이벤트를 메시지 버스의 메시지로 변환한다.

```cs
// 도메인 이벤트를 처리하는 컨트롤러

public string ChangeEmail(int userId, string newEmail)
{
    ...
    foreach(var ev in user.EmailChangedEvents) {
        _messageBus.SendEmailChangedMessage(
            ev.UserId, ev.NewEmail);
    }
}
```

저장 로직이 **도메인 이벤트**에 의존하지 않기 때문에 여전히 Company와 User 인스턴스는 **무조건** 데이터베이스에 저장된다. 이는 데이터베이스 **변경 사항**과 메시지버스의 **메시지**가 **다르기 때문**이다.

CRM을 제외한 다른 애플리케이션이 데이터베이스에 **접근 권한이 없다면** 데이터베이스와의 통신은 CRM의 **식별할 수 있는 동작**이 아니라 **구현 세부 사항**이다. 데이터베이스는 **최종 상태**만 정확하면 **호출 횟수**는 중요하지 않지만, 메시지 버스와의 통신은 CRM이 **식별할 수 있는 동작**이다. **외부 시스템**과의 **계약**을 지키려면 CRM 은 이메일이 **변경될 때만** 메시지 버스에 넣어야 한다.

추가로 컨트롤러에서 도메인 이벤트를 수동으로 발송하는 대신, 별도의 **이벤트 디스패처**를 작성할 수도 있다.

**도메인 이벤트**는 **컨트롤러**에서 **의사 결정 책임**을 **제거**하고 해당 책임을 **도메인 모델**에 적용함으로써 **외부 시스템**과의 통신에 대한 단위 테스트를 **간결**하게 한다.

```cs
// 도메인 이벤트 생성 테스트

[Fact]
public void Change_email_from_corporate_to_non_coporate()
{
    var company = new Company("mycorp.com", 1);
    var sut = new User(1, "user@mycorp.com", UserType.Employee, false);
    
    sut.ChangeEmail("new@gmail.com", company);
    
    company.NumberOfEmployees.Should().Be(0);
    sut.Email.Should().Be("new@gmail.com");
    sut.Type.Should().Be(UserType.Customer);
    
    // 컬렉션 크기와 요소를 동시에 검증
    sut.EmailChangedEvents.Should().Equal(new EmailChangedEvent(1, "new@gmail.com"))    
}
```

### 7.5 결론

이 장에서 다뤘던 주제는 **외부 시스템**에 대한 애플리케이션의 **부작용**을 **추상화**하는 것이었다. **비즈니스 연산**이 **끝날 때까지** 부작용을 **메모리**에 둬서 **추상화**하고, **프로세스 외부 의존성** 없이 **단순한** 단위 테스트로 테스트할 수 있다. **추상화할 것**을 테스트하기보다는 **추상화**를 테스트하는 것이 더 쉽다. 도메인 이벤트는 **프로세스 외부 의존성 호출** 위의 **추상화**에 해당하며, **도메인 클래스**의 **변경**은 데이터 저장소의 **향후 수정**에 대한 **추상화**다.

도메인 이벤트와 CanExecute/Execute 패턴을 사용해 도메인 모델에 모든 **의사 결정**을 잘 담을 수 있었지만, **비즈니스 로직**의 **파편화**가 불가피한 상황의 경우 담을 수 **없다**. 그러나 **잠재적인 파편화**가 있더라도 **비즈니스 로직**을 **오케스트레이션**에서 **분리**하는 것은 단위 테스트 프로세스가 **간소화**되기 때문에 **많은 가치**가 있다.

**컨트롤러**에 **비즈니스 로직**이 있는 것을 피할 수 없는 것처럼, 도메인 클래스에서 모든 협력자를 제거할 수 있는 경우는 거의 없을 것이다. 하지만 **협력자**가 있더라도 **프로세스 외부 의존성**을 **참조하지 않는 한**, 도메인 클래스는 지나치게 복잡한 코드가 아닐 것이다.

그러나 이러한 **협력자**와의 **상호 작용**을 검증하려고 **목을 사용하지 마라**. 이러한 **상호 작용**은 **도메인 모델**의 **식별할 수 있는 동작**과 아무런 관련이 **없다**. **컨트롤러**에서 **도메인** 클래스로 가는 **첫 번째 호출**만 컨트롤러의 **목표**와 **연관**이 있고, **같은 연산 내**에서 인접 도메인 클래스에 대해 수행하는 **후속 호출**은 모두 **구현 세부 사항**이다.

메서드가 클래스의 **식별할 수 있는 동작**인지 여부는 **클라이언트**가 **누구**인지와 **목표**가 무엇인지에 달려있다. 다음 두 가지 기준 중 하나를 충족해야 한다.

* 클라이언트 **목표** 중 하나에 **직접적인 연관**이 있다.
* 외부 애플리케이션에서 **볼 수 있는 프로세스 외부 의존성**에서 부작용이 발생한다.

컨트롤러의 `ChangeEmail()`와 메시지 버스에 대한 호출은 **식별할 수 있는 동작**이다. `ChangeEmail()`은 외부 클라이언트의 **진입점**이기 때문에 **첫 번째 기준**을 충족하며, 메시지 버스에 대한 호출은 **외부 애플리케이션**으로 메시지를 보내기 때문에 두 번째 기준을 충족한다.

이 두 메서드의 호출을 모두 확인해야 한다. 그러나 컨트롤러에서 User로 가는 **후속 호출**은 외부 클라이언트의 **목표와 연관이 없다**. 클라이언트는 시스템의 올바른 **최종 상태**와 메시지 버스 호출이 잘 되는 한, **구현 세부 사항**은 상관하지 않는다. 따라서 컨트롤러의 **동작**을 테스트할 때 컨트롤러가 User에 수행하는 **호출을 검증해서는 안 된다**.

호출 스택을 한 단계 내려도, 클라이언트는 컨트롤러이고 User의 `ChangeEmail()`은 사용자 이메일을 변경하는 **클라이언트의 목표**에 **직접적인 연관**이 있기 때문에 테스트해야 한다. 그러나 User에서 Company로 가는 **후속 호출**은 **구현 세부 사항**이다. 따라서 User의 `ChangeEmail()`을 다루는 테스트는 User가 어떤 Company에 어떤 메서드를 호출하는지 검증해서는 **안 된다**.

한 단계 더 내려가서 Company의 두 가지 메서드를 User 관점에서 테스트할 때도 마찬가지다.

**식별할 수 있는 동작**과 **구현 세부 사항**을 양파의 여러 겹으로 생각하라. 외부 계층의 관점에서 각 계층을 테스트하고, 해당 계층이 기저 계층과 어떻게 통신하는지 **무시**하며, 이러한 계층을 하나씩 벗겨가면서 **관점**을 바꿔라. 이전에 **구현 세부 사항**이었던 것이 이제는 **식별할 수 있는 동작**이 되며 이는 **또 다른** 테스트로 다루게 된다.
