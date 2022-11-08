# Chapter 10 - 데이터베이스 테스트

통합 테스트의 마지막 퍼즐은 프로세스 외부 관리 의존성이다. 가장 일반적인 예는 애플리케이션 외의 다른 접근이 없는 데이터베이스이며,  이를 테스트하면 회귀 방지가 아주 뛰어나지만 설정이 복잡하다. 이 장에서는 데이터베이스 테스트를 시작하기 전 거쳐야 할 준비 단계를 살펴본다.

## 10.1 데이터베이스 테스트를 위한 전제 조건

통합 테스트에서는 관리 의존성이 그대로 있어야 한다. 목을 사용하면 안되기 때문에 비관리 의존성보다 작업하기 힘들다. 그러나 테스트 작성 전 통합 테스트가 가능하게끔 준비 단계가 있어야 한다. 이 절에서는 다음과 같은 전제 조건을 살펴본다.

* 형상 관리 시스템에 데이터베이스 유지
* 모든 개발자를 위한 별도의 데이터베이스 인스턴스 사용
* 데이터베이스 배포에 마이그레이션 기반 방식 적용

테스트를 용이하게 하면 보통 데이터베이스 상태도 개선된다.

### 10.1.1 데이터베이스를 형상 관리 시스템에 유지

데이터베이스를 테스트하는 첫 단계를 데이터베이스 스키마를 **일반 코드**로 취급하는 것이다. Git과 같은 **형상 관리 시스템**에 저장하는 것이 최선이다.

모델 데이터베이스 인스턴스에 모든 스키마 변경 사항이 쌓인다면 운영 배포 할 때 팀은 운영 데이터베이스와 모델 데이터베이스를 비교하고, 업그레이드 스크립트를 위한 전문 도구를 사용했으며, 운영 환경에서 해당 스크립트를 실행한다. 이와 같은 전용 인스턴스를 모델 데이터베이스로 사용하는 것은 **안티패턴**이다. 이유는 다음과 같다.

* 변경 내역 부재
    * 스키마를 과거의 특정 시점으로 되돌릴 수 없다. 이는 운영 환경에서 **버그를 재현**할 때 중요하다.
* 복수의 원천 정보
    * 모델 데이터베이스는 개발 상태에 대한 원천 정보를 둘러싸고 경합한다. 기준은 두 가지(Git과 모델 데이터베이스)로 두면 부담이 가중된다.

반면 스키마 업데이트를 형상 관리 시스템에 두면 원천 정보를 하나로 할 수 있고, 일반 코드와 함께 변경을 추적할 수 있다. 형상 관리 외부에서는 데이터베이스 구조를 수정하면 안된다.

### 10.1.2 참조 데이터도 데이터베이스 스키마다

스키마에 관해 유력한 용의자는 테이블, 뷰, 인덱스, 저장 프로시저 그리고 데이터베이스의 청사진을 형성하는 나머지 모든 것이다. 스키마는 SQL 스크립트로 표현되며, 개발 중 언제든지 이러한 스크립트로 기능을 완전히 갖춘 최신 데이터베이스 인스턴스를 만들 수 있어야 한다. 그러나 _참조 데이터_의 경우 스키마에 속하지만, 스키마로 거의 여기지 않는다.

> 참조 데이터란 애플리케이션이 제대로 작동하도록 미리 채워야 하는 데이터다. 애플리케이션이 데이터를 수정할 수 있다면 일반 데이터고, 그렇지 않다면 참조 데이터다.

CRM 시스템의 경우 사용자 유형은 Customer 또는 Employee일 수 있다. 모든 사용자 유형이 포함된 테이블을 만들고 User에서 해당 테이블로 외래 키 제약 조건을 두고 싶다고 가정하면, 이런 제약 조건으로 사용자에게 존재하지 않는 유형을 할당하지 않게끔 **보증**할 수 있다. 이 시나리오에서 사용자를 저장하려면 UserType 테이블에 데이터가 있어야하기 때문에 UserType 테이블의 내용이 **참조 데이터**에 해당한다.

참조 데이터는 필수 사항이기 때문에, 스키마와 함께 `SQL INSERT` 문 형태로 형상 관리 시스템에 저장해야 한다. 참조 데이터는 보통 일반 데이터와 별도로 저장되지만, 두 데이터가 동일한 테이블에 공존하는 경우, 수정할 수 있는 데이터(일반 데이터)와 수정할 수 없는 데이터(참조 데이터)를 **구분**하여, 애플리케이션이 데이터를 변경하지 못하도록 해야 한다.

### 10.1.3 모든 개발자를 위한 별도의 데이터베이스 인스턴스

실제 데이터베이스로 테스트 하는 것은 어렵다. 다른 개발자와 공유하는 데이터베이스를 사용하면 개발 프로세스를 방해하게 된다. 그 이유는 다음과 같다.

* 서로 다른 개발자가 실행한 테스트는 서로 간섭된다.
* 하위 호환성이 없는 변경으로 다른 개발자의 작업을 막는다.

테스트 실행 속도를 극대화하려면 개발자 마다 **별도의 인스턴스**를 사용하라.

### 10.1.4 상태 기반 데이터베이스 배포와 마이그레이션 기반 데이터베이스 배포

데이터베이스 배포에는 상태 기반과 마이그레이션 기반이라는 두 가지 방식이 있다. 마이그레이션 기반 방식은 초기에는 구현하고 유지 보수가 어렵지만 장기적으로 상태 기반 방식보다 좋다.

#### 상태 기반 방식

상태 기반 데이터베이스 배포 방식은 개발 내내 유지 보수하는 모델 데이터베이스가 있다. 배포 중 **비교 도구**가 스크립트를 생성해서 운영 데이터베이스를 모델 데이터베이스와 비교해 최신 상태로 유지한다. 상태 기반 방식에서 모델 데이터베이스에는 원천 데이터가 아닌 SQL 스크립트가 있으며, 이는 형상 관리에 저장된다.

상태 기반 접근 방식에서 비교 도구는 운영 데이터베이스 상태와 관계 없이 불필요한 데이터를 삭제하고 새 테이블을 생성하고 컬럼명을 바꾸는 등 모델 데이터베이스와 **동기화**하는 데 필요한 모든 작업을 수행한다.

#### 마이그레이션 기반 방식

반면 마이그레이션 기반 방식은 데이터베이스를 어떤 버전에서 다른 버전으로 전환하는 **명시적인 마이그레이션**을 의미한다. 이 방식은 비교 도구를 쓸 수 없고, 업그레이드 스크립트를 **직접 작성**해야 한다. 하지만 스키마에서 문서화되지 않은 변경 사항을 발견할 때는 비교 도구를 사용할 수 있다. 

마이그레이션 기반 방식에서 형상 관리에 저장하는 산출물은 **데이터베이스 상태가 아닌** 마이그레이션이다. 마이그레이션은 일반적으로 평이한 SQL 스크립트로 표시하지만, SQL로 변환할 수 있는 DSL 같은 언어를 사용할 수 있다.

#### 상태 기반 방식보다 마이그레이션 기반 방식을 선호하라

* 상태 기반 방식
    * 상태를 형상 관리에 저장하여 **상태를 명시**하고 **비교 도구**가 마이그레이션을 **암묵적으로 제어**할 수 있게 한다.
* 마이그레이션 기반 방식
    * 마이그레이션을 명시적으로 하지만 **상태를 암묵적**으로 둔다. 데이터베이스 상태를 직접 볼 수 없으며, 마이그레이션으로 **조합**해야 한다.

이는 다른 형태의 절충으로 이어진다. 데이터베이스의 상태가 명확하면 **병합 충돌**을 처리하기 수월하며, 명시적 마이그레이션은 _데이터 모션_ 문제를 해결하는 데 도움이 된다.

> 데이터 모션이란 새로운 데이터베이스 스키마를 준수하도록 기존 데이터의 형태를 변경하는 과정이다.

대부분의 프로젝트에서 데이터 모션이 병합 충돌보다 훨씬 더 중요하다. 컬럼을 변경하는 경우 상태 주도 방식을 사용하면 변경을 구현하기 어렵다. 데이터 관리에서 비교 도구는 좋지 않다. 스키마가 아닌 데이터는 상황에 따라 **달라지기 때문이다**. 적절한 변환을 구현하려면 도메인에 특화된 규칙을 적용해야 한다.

결과적으로 상태 기반은 대부분 프로젝트에서 실용적이지 않다. 그러나 아직 릴리스되지 않은 상태라면 일시적으로는 사용할 수 있다. 테스트 데이터는 중요하지 않으며, 데이터베이스가 변경될 때 마다 데이터를 다시 생성할 수 있지만, 릴리스 이후는 마이그레이션 기반으로 전환해서 _데이터 모션_을 처리해야 한다.

형상 관리에 마이그레이션이 커밋된 후에는 수정이 필요하다면(데이터가 손실될 수 있는 경우를 제외) **수정이 아닌 새 마이그레이션**을 생성하라. 

## 10.2 데이터베이스 트랜잭션 관리

제품 코드에서 트랜잭션 관리를 적절히 하면 데이터 모순을 피할 수 있다. 테스트에서는 운영 환경에 근접한 설정으로 데이터베이스 통합을 검증하는 데 도움이 된다. 이 절에서는 CRM 프로젝트를 예시로 제품코드와 통합 테스트에서 트랜잭션을 사용하는 방법을 알아 본다.

### 10.2.1 제품 코드에서 데이터베이스 트랜잭션 관리하기

CRM 프로젝트에서 Database는 각 메서드 호출에서 별도의 SQL을 생성하여, 암묵적으로 트랜잭션을 독립적으로 처리한다.

#### 데이터베이스에 접근할 수 있는 클래스

```cs
public class Database
{
    private readonly string _connectionString;
    
    public Database(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public void SaveUser(User user)
    {
        bool isNewUser = user.UserId == 0;
        using(var connection =
            new SqlConnection(_connectionString)) // Transaction begin
        {
            // 사용자 생성 또는 업데이트
        }
    }
    
    public void SaveCompany(Company company)
    {
        usins(var connection = 
            new SqlConnection(_connectionString)) // Transaction begin
        {
            // 회사 업데이트
        }
    }
}

```

사용자 컨트롤러는 단일 비즈니스 연산 간에 총 네개의 데이터베이스 트랜잭션을 생성한다.

#### 사용자 컨트롤러

```cs
public string ChangeEmail(int userId, string newEmail)
{
    object[] userData = _database.GetUserById(userId); // Transaction begin
    User user = UserFcatory.Create(userData);
    
    string error = user.CanChangeEmail();
    if (error != null)
        return error;
        
    object[] companyData = _database.GetCompanyId(companyId); // Transaction begin
    Company company = CompanyFactory.Create(companyData);
    
    user.ChangeEmail(newEmail, company);
    
    _database.SaveCompany(company); // Transaction begin
    _database.SaveUser(user); // Transaction begin
    _eventDispatcher.Diapatch(user.DomainEvents);
    
    return "OK";
}
```

읽기 전용 연산이라면 여러 트랜잭션을 열어도 괜찮다. 그러나 그게 아니라면, 모순을 피하고자 연산에 포함되는 _모든 업데이트는 원자적_이어야 한다. 컨트롤러가 회사는 잘 저장해도 데이터베이스에 문제가 발생해 사용자를 저장할 때 실패하면 상황의 경우가 발생한다.

> 원자적 업데이트란 모두 수행하거나 전혀 수행하지 않는 것이다.

#### 데이터베이스 트랜잭션에서 데이터베이스 연결 분리하기

잠재적인 모순을 피하려면 결정 유형을 두 가지로 나눠야 한다.

* 업데이트할 데이터
* 업데이트 유지 또는 롤백 여부

컨트롤러가 이러한 결정들을 동시에 내릴 수 없기 때문에 이렇게 분리하는 것이 중요하며, 비즈니스 연산이 모두 성공했을 때만 업데이트를 수행할 수 있는지 여부를 알 수 있어야 한다. 또한 데이터베이스에 접근하고 업데이트를 시도해야만 하기 때문에 리포지터리와 트랜잭션으로 나눠서 이러한 **책임을 구분**할 수 있다.

* 리포지터리
    * 데이터베이스의 데이터에 대한 접근과 수정을 가능하게 하는 클래스이며, 도메인 별로 있어야 한다.
* 트랜잭션
    * 데이터 업데이트를 완전히 커밋하거나 롤백하며, 데이터 수정의 **원자성 확보**를 위해 기본 데이터베이스의 트랜잭션에 의존하는 사용자 정의 클래스다.

이 둘은 서로 책임 뿐만 아니라 수명도 다르다. 리포지터리는 항상 현재 트랜잭션 위에서 작동한다. 데이터베이스에 연결할 때는 리포지터리가 트랜잭션에 등록해서 연결 중에 이뤄지는 모든 데이터 수정 사항이 나중에 트랜잭션에 의해 롤백될 수 있도록 한다. 

**명시적 트랜잭션**을 도입한다면, 트랜잭션은 컨트롤러와 데이터베이스 간의 상호작용을 조정한다. 네 개의 데이터베이스 호출은 그대로지만, 데이터 수정은 커밋되거나 완전히 롤백된다. 다음은 트랜잭션과 리포지터리를 도입한 후의 컨트롤러다.

#### 사용자 컨트롤러, 리포지터리, 트랜잭션

```cs
public class UserController
{
    private readonly Transaction _transaction;
    private readonly UserRepository _userRepository;
    private readonly CompanyRepository _companyRepository;
    private readonly EventDispatcher _eventDispatcher;
    
    public UserController(
        Transaction transaction, // 트랜잭션 인자
        MessageBus messageBus,
        IDomainLogger domainLogger)
    {
        _transaction = transaction;
        _userRepository = new UserRepository(transaction);
        _companyRepository = new CompanyRepository(transaction);
        _eventDispatcher = new EventDispatcher(
            messageBus, domainLogger);
    }
    
    public string ChangeEmail(int userId, string newEmail)
    {
        object[] userData = _userRepository.GetUserById(userId); // Transaction begin
        User user = UserFcatory.Create(userData);
        
        string error = user.CanChangeEmail();
        if (error != null)
            return error;
            
        object[] companyData = _companyRepository.GetCompanyId(companyId); // Transaction begin
        Company company = CompanyFactory.Create(companyData);
        
        user.ChangeEmail(newEmail, company);
        
        _companyRepository.SaveCompany(company); // Transaction begin
        _userRepository.SaveUser(user); // Transaction begin
        _eventDispatcher.Diapatch(user.DomainEvents);
        
        _transaction.Commit(); // 성공 시 커밋
        return "OK";
    }
}

public class UserRepository
{
    private readonly Transaction _transaction;
    
    public UserRepository(Transaction transaction)
    {
        _transaction = transaction;
    }
    
    ...
}

public class Transaction : IDisposable
{
    public void Commit() { ... }
    public void Dispose() { ... }
}
```

Transaction에서는 다음 두 가지 중요한 메서드가 있다.

* Commit()
    * 트랜잭션을 성공으로 표시한다. 비즈니스 연산이 성공하고 모든 데이터의 수정 준비가 완료 된 경우에만 호출된다.
* Dispose()
    * 트랜잭션을 종료한다. 비즈니스 연산이 끝날 때 항상 호출되며, 이전에 Commit()이 호출된 경우 Dispose()는 모든 업데이트를 저장하고, 그렇지 않으면 롤백한다.

트랜잭션은 주요 흐름(비즈니스 시나리오의 성공적인 실행)동안에만 데이터베이스가 변경되도록 한다. 오류가 발생하면 실행 흐름이 일찍 반환돼서 커밋되지 않는다.

Commit() 호출은 **의사 결정**이 필요하기 때문에 컨트롤러가 한다. Dispose() 호출은 인프라 계층의 클래스에 메서드 호출을 위임할 수 있다. 컨트롤러는 인스턴스화하고 필요한 의존성을 제공하는 클래스는 모두 컨트롤러가 작업을 수행한 후에 트랜잭션을 폐기해야 한다.

UserRepository에서 생성자 매개 변수로 트랜잭션을 요구하는 것을 볼 수 있다. 즉, 리포지터리 **스스로 호출 할 수 없다**.

#### 작업 단위로 트랜잭션 업그레이드하기

Transaction 클래스를 _작업 단위_로 업그레이드 한다면 비즈니스 연산 종료 시점에 모든 업데이트를 실행하기 때문에 데이터베이스 트랜잭션의 기간을 단축하고 데이터 혼잡을 줄일 수 있다. 이 패턴은 종종 데이터베이스 호출 수를 줄이는 데 도움이 된다.

> 작업 단위에는 비즈니스 연산의 영향을 받는 객체 목록이 있으며, 작업이 완료되면 데이터베이스를 변경하기 위한 업데이트를 모두 파악하고 업데이트를 **하나의 단위**로 실행한다. (작업 단위 패턴이라고 함)

수정된 객체 목록을 관리하고 생성하기 위해 SQL 스크립트를 파악할 필요 없이 대부분 ORM 라이브러리가 작업 단위 패턴을 구현한다. 엔티티 프레임 워크는 원시 데이터베이스 데이터와 도메인 객체 사이의 **매퍼 역할**을 하기 때문에 Factory 클래스는 더 이상 필요하지 않다.

#### 비관계형 데이터베이스에서의 데이터 모순

대부분의 비관계형 데이터베이스가 갖는 문제점은 고전적인 의미에서 트랜잭션이 없다는 것이다. 원자적 업데이트는 단일 도큐먼트 내에서만 보장되며, 비즈니스 연산이 여러 문서에 영향을 주는 경우 모순이 생기기 쉽다.

비관계형에서는 한 번에 둘 이상의 도큐먼트를 수정하는 비즈니스 연산이 없도록 도큐먼트를 설계해야 한다. 이는 관계형 데이터베이스의 행 보다 도큐먼트가 더 유연하기 때문에 가능한 일이다. 단일 도큐먼트는 정해진 형태가 없으며, 데이터가 복잡하더라도 저장할 수 있기 때문에 복잡한 비즈니스 연산의 부작용도 포착할 수 있다.

도메인 주도 설계에서는 비즈니스 연산 당 둘 이상의 집계를 수정하면 안 된다는 지침이 있으며, 각 도큐먼트가 **하나의 집계**에 해당하는 도큐먼트 데이터베이스를 사용하는 시스템에만 적용된다.

### 10.2.2 통합 테스트에서 데이터베이스 트랜잭션 관리하기

통합 테스트에서 트랜잭션을 관리하는 경우 다음 지침을 준수하라. 테스트 구절 간에 데이터베이스 트랜잭션이나 작업 단위를 재사용하지 마라. 다음은 통합 테스트를 엔티티 프레임워크로 전환해서 CrmContext를 재사용하는 예다.

#### CrmContext를 재사용하는 통합 테스트

```cs
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    using (var context 
        = new CrmContext(ConnectionString))
        
    // Assert
    var userRepository 
        = new UserRepository(context);
    var companyRepository 
        = new CompanyRepository(context);
    var user = new User(0, "user@mycorp.com", 
        UserType.Employee, false);
    userRepository.SaveUser(user);
    var company = new Company("mycorp.com", 1);
    companyRepository.SaveCompany(company);
    context.SaveChanges();
    
    var busSpy = new BusSpy();
    var messageBus = new MessageBus(busSpy);
    var loggerMock = new Mock<IDomainLogger>();
    var sut = new UserController(
        context, // 공유
        messageBus,
        loggerMock.Object);
    
    // Act
    string result = sut.ChangeEmail(user.UserId, "new@gmail.com");
    
    // Assert
    Assert.Equal("OK", result);
    
    User userFromDb = userRepository // 재사용
        .GetUserById(user.UserId);
    Assert.Equal("new@gmail.com", userFromDb.Email)
    Assert.Equal(UserType.Customer, userFromDb.Type);
    
    Company companyFromDb = companyRepository // 재사용
        .GetCompany();
        
    Assert.Equal(0, companyFromDb.NumberOfEmployees);
    
    busSpy.ShouldSendNumberOfMessage(1)
        .WithEmailChangedMessage(user.UserId, "new@gmail.com");
    loggerMock.Verify(
        x => x.USerTypeHasChanged(
            user.UserId, UserType.Employee, UserType.Customer),
         Times.Once);
}
```

동일한 CrmContext 인스턴스를 사용하는 것은 컨트롤러가 운영 환경에서 하는 것과 다른 환경을 만들기 때문에 문제가 된다. 운영 환경에서는 각 비즈니스 연산에 CrmContext의 전용 인스턴스가 있고, 전용 인스턴스는 컨트롤러 메서드 호출 직전에 생성되고 폐기된다.

동작 모순에 빠지지 않으려면 통합 테스트를 가능한 운영 환경과 유사하게 만들어야 한다. 즉, 실행 구절에서 CrmContext를 다른 것과 공유하면 안된다.

데이터베이스의 상태를 확인하는 것은 중요하기 때문에 준비와 검증 구절에서 CrmContext 인스턴스를 가져와야 한다. 많은 ORM 라이브러리는 성능 향상을 위해 요청된 데이터를 캐싱할 수 있기 때문에, 검증 구절에서 같은 컨텍스트를 사용하면 안된다.

통합 테스트에서 **적어도 세 개**의(준비, 실행, 검증) 트랜잭션 또는 작업 단위를 사용하라.

## 10.3 테스트 데이터 생명 주기

공유 데이터베이스를 사용하면 통합 테스트를 서로 분리할 수 없다. 이 문제를 해결하려면,

* 통합 테스트를 **순차적**으로 실행하라.
* 테스트 실행 간에 남은 데이터를 제거하라.

전체적으로 테스트는 데이터베이스의 상태에 따라 달라지면 안된다. 

### 10.3.1 병렬 테스트 실행과 순차적 테스트 실행

통합 테스트를 병렬로 실행하려면 모든 테스트 데이터가 고유한지 확인해야 데이터베이스 제약 조건을 위반하지 않고 테스트가 다른 테스트 후에 입력 데이터를 잘못 수집하는 일이 없다. 성능 향상을 위해 시간을 허비하지말고 순차적으로 통합 테스트를 실행하는 것이 더 실용적이다.

대부분의 단위 테스트 프레임워크는 병렬 처리를 비활성화할 수 있다. 대안으로 컨테이너를 사용할 수 있으나, 이러한 방식은 실제로 유지 보수 부담이 너무 커지게 된다. 도커를 사용하면 데이터베이스 추적 뿐만 아니라,

* 도커 이미지를 유지보수 해야 한다.
* 각 테스트마다 컨테이너 인스턴스가 있는지 확인해야 한다.
* 통합 테스트를 일괄처리해야 한다. (모든 컨테이너 인스턴스를 한 번에 만들 수 없기 때문)
* 다 사용한 컨테이너는 폐기해야 한다.

데이터베이스는 컨테이너를 사용하는 것이 아닌 개발자당 하나의 인스턴스를 갖는 것이 더 실용적이다.

### 10.3.2 테스트 실행 간 데이터 정리

남은 데이터를 정리하는 방법은 네 가지가 있다.

* 각 테스트 전에 데이터베이스 백업 복원
    * 이는 다른 세 가지 방법보다 훨씬 느리다. 
* 테스트 종료 시점에 데이터 정리
    * 이는 빠르지만 정리 단계를 건너뛰기 쉽다. 테스트 중 빌드 서버가 중단하거나 디버거에서 테스트를 종료하면 데이터는 그대로 남아있고 이후 테스트에 영향을 준다.
* 데이터베이스 트랜잭션에 각 테스트를 래핑하고 커밋하지 않기
    * 이 경우 테스트와 SUT에서 변경한 모든 내용이 자동으로 롤백된다. 하지만 이는 작업 단위를 재사용할 때와 같은 문제인데, 추가 트랜잭션으로 인해 운영 환경과 다른 설정이 생성되는 것이다.
* 테스트 시작 시점(준비 구절)에 데이터 정리하기
    * 이 방법이 가장 좋다. 빠르고, 일관성 없는 동작을 일으키지 않으며, 정리 단계를 실수로 건너뛰지 않는다.

데이터베이스 외래 키 제약 조건을 준수하려면 특정 순서에 따라 데이터를 제거해야 한다. 정교한 알고리즘을 이용해 SQL 스크립트를 생성하거나, 무결성 제약 조건을 비활성화하는 대신 SQL 스크립트를 **수동으로 작성**하면 간단하고 삭제 프로세스를 세밀하게 제어할 수 있다.

모든 통합 테스트는 기초 클래스를 두고, 기초 클래스에 삭제 스크립트를 작성하라.

#### 통합 테스트를 위한 기초 클래스

```cs
public abstract class IntegrationTests
{
    private const string ConnectionString = " ... ";
    
    protected Integration()
    {
        ClearDatabase();
    }
    
    private void ClearDatabase()
    {
        string query = 
            "DELETE FROM dbo.[User];" + // 삭제 스크립트
            "DELETE FROM dbo.Company";
            
        using(var connection = new SqlConnection(ConnectionString))
        {
            var commane = new SqlCommand(query, connection);
            {
                CommandType = CommandType.Text
            };
            
            connection.Open();
            command.ExecuteNonQuery();
        }
    }
}
```

삭제 스크립트는 일반 데이터를 모두 제거하며, 데이터베이스 스키마와 참조 데이터는 마이그레이션으로만 제어되어야 한다.

### 10.3.3 인메모리 데이터베이스 피하기

통합 테스트를 서로 분리하는 또 다른 방법으로 인 메모리 데이터베이스를 사용하는 것인데, 다음과 같은 장점이 있다.

* 테스트 데이터를 제거할 필요 없음
* 작업 속도 향상
* 테스트가 실행될 때마다 인스턴스화 가능

인메모리는 공유 의존성이 아니기 때문에 통합 테스트는 실제로 컨테이너 접근 방식과 유사한 단위 테스트가 된다. 하지만 일반 데이터베이스와 기능적으로 일관성이 없기 때문에 사용하지 않는 것이 좋다. 이는 운영 환경의 불일치 문제와 데이터베이스 간의 차이로 인해 거짓 양성 또는 거짓 음성이 발생하기 쉬우며, 수동으로 회귀 테스트를 많이 수행해야 할 것이다.

## 10.4 테스트 구절에서 코드 재사용하기

통합 테스트는 가능한 짧으면서 서로 결합하거나 가독성에 영향을 주지 않아야 한다. 서로 의존해서는 안되고 시나리오의 전체 컨텍스트를 유지해야 하며, 진행 상황을 이해하려고 테스트 클래스의 다른 부분을 검사하면 안된다.

통합 테스트를 짧게 하기에 가장 좋은 방법은 비즈니스와 관련이 없는 기술적인 부분을 비공개 메서드나 **헬퍼 클래스**로 추출하여 재사용하는 것이다. 이 절에서는 테스트의 세 가지 구절을 어떻게 줄여야 하는지 알아본다.

### 10.4.1 준비 구절에서 코드 재사용하기

다음 예제는 별도의 데이터베이스 컨텍스트를 두고 나서의 통합 테스트다.

#### 세 개의 데이터베이스 컨텍스트가 있는 통합 테스트

```cs
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    // 준비
    User user;
    using (var context = new CrmContext(ConnectionString))
    {
        var userRepositor = new UserRepository(context);
        var companyRepository = new CompanyRepository(context);
        var user = new User(0, "user@mycorp.com", 
            UserType.Employee, false);
        userRepository.SaveUser(user);
        var company = new Company("mycorp.com", 1);
        companyRepository.SaveCompany(company);
        
        context.SaveChanges();
        
    }
    
    var busSpy = new BusSpy();
    var messageBus = new MessageBus(busSpy);
    var loggerMock = new Mock<IDomainLogger>();
    
    string result;
    using (var context = new CrmContext(ConnectionString))
    {
        var sut = new UserController(
        context, messageBus,loggerMock.Object);
    
        // 실행
        string result = sut.ChangeEmail(user.UserId, "new@gmail.com");
    }
    
    // 검증
    Assert.Equal("OK", result);
    using (var context = new CrmContext(ConnectionString))
    {
        var userRepositor = new UserRepository(context);
        var companyRepository = new CompanyRepository(context);
        
        User userFromDb = userRepository.GetUserById(user.UserId);
        Assert.Equal("new@gmail.com", userFromDb.Email)
        Assert.Equal(UserType.Customer, userFromDb.Type);
        
        Company companyFromDb = companyRepository.GetCompany();
        Assert.Equal(0, companyFromDb.NumberOfEmployees);
        
        busSpy.ShouldSendNumberOfMessage(1)
            .WithEmailChangedMessage(user.UserId, "new@gmail.com");
        loggerMock.Verify(
            x => x.USerTypeHasChanged(
                user.UserId, UserType.Employee, UserType.Customer),
             Times.Once);
    }
}
```

테스트 준비 구절 간에 코드를 재사용하기 가장 좋은 방법은 비공개 팩토리 메서드를 도입하는 것이다. 예를 들어 다음 예제처럼 사용자를 생성하는 것이다.

#### 사용자를 생성하는 별도의 메서드

```cs
private User CreateUser(
    string email, UserType type, bool isemailConfirmed)
{
    using (var context = new CrmContext(ConnectionString))
    {
        var user = new User(0, email, type, isEmailConfirmed);
        var repository = new UserRepository(context);
        repository.SaveUser(user);
        
        context.SaveChanges();
        
        return user;
    }
}
```

메서드 인수에 대한 기본 값을 정의할 수도 있다.

#### 팩토리 메서드에 기본 값 추가

```cs
private User CreateUser(
    string email = "user@mycorp.com", 
    UserType type = UserType.Employee, 
    bool isemailConfirmed = false)
```

기본 값을 사용하면 인수를 선택적으로 지정할 수 있다.

#### 오브젝트 마더와 테스트 데이터 빌더

기본 값을 추가하는 패턴을 **오브젝트 마더**라고 한다. 오브젝트 마더는 테스트 픽스처를 만드는 데 도움이 되는 클래스 또는 메서드다. 준비 구절에서 코드의 재사용에 도움이 되는 패턴으로 **테스트 데이터 빌더**도 있다. 이는 오브젝트 마더와 유사하지만 일반 메서드 대신 플루언트 인터페이스를 제공한다. 테스트 데이터 빌더는 가독성을 약간 향상시키는 반면, 상용구가 너무 많이 필요하기 때문에 오브젝트 마더를 사용하는 것이 좋다.

#### 팩토리 메서드를 배치할 위치

단순하게 시작하라. 기본적으로 팩토리 메서드를 동일한 클래스에 배치하고, 코드 복제가 중요한 문제가 될 경우 별도의 헬퍼 클래스로 이동하라. 기초(추상) 클래스에 팩토리 메서드를 넣지마라.  기초 클래스는 데이터 정리와 같은 모든 테스트에서 실행해야 하는 코드를 위한 클래스다.

### 10.4.2 실행 구절에서 코드 재사용하기

모든 통합 테스트의 실행 구절에서 데이터베이스 트랜잭션이나 작업 단위를 만든다. 실행 구절도 줄일 수 있다. 어떤 컨트롤러 기능을 호출해야 하는지에 대한 정보가 있는 **대리자**를 받는 메서드를 도입할 수 있다. 다음 예제와 같이 메서드가 데이터 베이스 컨텍스트를 생성해서 컨트롤러의 호출을 감싼다.

#### 데코레이터 메서드

```cs
private string Execute(
    Func<UserController, string> func,
    MessageBus messageBus,
    IDomainLogger logger)
{
    using(var context = new CrmContext(ConnectionString))
    {
        var controller = new UserController(
            context, messageBus, logger);
        return func(controller);
    }    
}
```

**데코레이터 메서드**를 사용하면 테스트의 실행 구절은 몇 줄만으로 충분하다.

```cs
string result = Execute(
    x => x.ChangeEmail(user.UserId, "new@gmail.com"),
     messageBus, loggerMock.Object);
```

### 10.4.3 검증 구절에서 코드 재사용하기

가장 쉬운 방법은 다음 예제와 같이 CreateUser나 CreateCompany와 유사한 헬퍼 메서드를 두는 것이다.

#### 조회 로직 추출 후 데이터 검증

```cs
User userFromDb = QueryUser(user.UserId);
Assert.Equal("new@gmail.com", userFromDb.Email);
Assert.Equal(UserType.Customer, userFromDb.Type);

Company companyFromDb = QueryCompany();
Assert.Equal(0, companyFormDb.NumberOfEmployees);
```

한 발짝 더 나아가, BusSpy와 같이 데이터 검증을 위한 플루언트 인터페이스를 확장 메서드로 구현할 수 있다.

#### 데이터 검증을 위한 플루언트 인터페이스

```cs
public static class UserExtensions
{
    public static User ShouldExist(this User user)
    {
        Assert.NotNull(user);
        return user;
    }
    
    public static User WithEmail(this User user, string email)
    {
        Assert.Euqal(email, user.Email);
        return user;
    }
}
```

플루언트 인터페이스를 사용하면 가독성이 향상된다.

```cs
User userFormDb = QueryUser(user.UserId);
userFromDb
    .ShouldExist()
    .WithEmail("new@gmail.com")
    .WithType(UserType.Customer);
    
...
```

### 10.4.4 테스트가 데이터베이스 트랜잭션을 너무 많이 생성하는가?

통합 테스트를 간결하게 하면 트랜잭션이 많이 생성된다는 단점이 있다. 데이터베이스 컨텍스트를 추가하면 테스트가 느려지기 때문에 어느 정도 문제가 되진 하지만, 할 수 있는건 많지 않다. 빠른 피드백과 유지 보수성 간의 절충을 해야 한다. 이러한 경우 유지 보수성을 위해 **성능을 양보함**으로써 절충하는 것이 좋다.

## 10.5 데이터베이스 테스트에 대한 일반적인 질문

### 10.5.1 읽기 테스트를 해야 하는가?

쓰기를 철저히 테스트하는 것은 매우 중요하다. 잘못되면 데이터가 손상돼 데이터베이스 뿐만 아니라 외부 애플리케이션에도 영향을 미칠 수 있다. 쓰기를 다루는 테스트는 이에 대한 보호책이기 때문에 매우 가치 있다.

그러나 읽기 작업의 버그는 보통 해로운 문제가 없다. 가장 복잡하거나 중요한 읽기 작업만 테스트하고, 나머지는 무시하라. 또한 읽기에는 도메인 모델도 필요하지 않다. 도메인 모델링의 주요 목표 중 하나는 캡슐화이며, 캡슐화는 변경 사항에 비춰 **데이터 일관성을 유지**하는 것이기 때문에 변경이 없으면 읽기 캡슐화는 의미가 없다. 불필요한 추상화 계층을 피해서 성능 면에서 ORM보다 우수한 일반 SQL을 사용하는 것이 좋다. 읽기에는 추상화 계층이 거의 없기 때문에 **단위 테스트가 아무 소용이 없으므로** 읽기를 테스트하기로 결정한 경우 실제 데이터베이스에서 **통합 테스트하라**.

### 10.5.2 리포지터리 테스트를 해야 하는가?

리포지터리가 도메인 객체를 어떻게 매핑하는지를 테스트하는 것은 유익할지 모른다. 그러나 이러한 테스트는 유지비가 높고 회귀 방지가 떨어져서 테스트 스위트에 손실이 된다. 이 두 가지 단점을 알아보자.

#### 높은 유지비

리포지터리는 복잡도가 거의 없고 프로세스 외부 의존성인 데이터베이스와 통신한다. 프로세스 외부 의존성이 있으면 테스트의 유지비가 증가한다. 리포지터리와 일반 통합 테스트는 그 부담 정도가 비슷하지만 그 대가로 유익한 것은 비슷하지 않다.

#### 낮은 회귀 방지

리포지터리는 복잡하지 않으며 회귀 방지에서 일반적인 통합 테스트가 주는 이점과 **겹친다**. 따라서 테스트의 가치를 더 주지 못한다. 리포지터리를 테스트하기 가장 좋은 방법은 UserFactory와 CompanyFactory와 같이 리포지터리가 갖고 있는 약간의 복잡도를 별도의 알고리즘으로 추출하고 이를 테스트 하는 것이다. 이 두 클래스는 다른 의존성을 두지 않고 모든 매핑을 수행했으며, 리포지터리에는 간단한 SQL 쿼리만 있다.

안타깝게도 ORM을 사용할 때 데이터 매핑과 데이터베이스 상호작용 간의 분리는 불가능하다. 따라서 다음 지침을 준수하라. 리포지터리는 직접 테스트하지 말고, **포괄적인 통합 테스트 스위트의 일부**로 취급하라.

EventDispatcher도 별도로 테스트하지 말라. 목 체계가 복잡해서 유지비가 너무 많이 들며, 회귀 방지의 이점은 너무 적다.

### 결론

데이터베이스 테스트는 데이터베이스를 리팩터링하거나 ORM을 전환하거나 데이터베이스를 변경할 때 큰 도움이 된다. 관리 의존성에 직접 작동하는 통합 테스트는 대규모 리팩터링에서 발생하는 버그로부터 보호하기에 가장 효율적인 방법이다.
