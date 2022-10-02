# 코드 구성하기

## 계층으로 구성하기
```
buckpal
├─ domain
│  ├─ Account
│  ├─ Activity
│  ├─ AccountRepository
│  └─ AccountService
├─ persistence
│  └─ AccountRepositoryImpl
└─ web
   └─ AccountController
```
- 웹 계층, 도메인 계층, 영속성 계층으로 구분하고 의존성 역전을 이용해 의존성이 도메인 코드로 향하도록 했다.
- 애플리케이션의 기능조각이나 특성을 구분 짓는 패키지 경계가 없다. User가 추가된다면 Account, User가 같은 패키지 안에서 존재하게 될 것이다.
- 애플리케이션이 어떤 유스케이스들을 제공하는지 파악할 수 없다. 특정기능을 찾기 위해서 어떤 서비스가 이를 구현했는지 추측해야한다.

## 기능으로 구성하기
```
buckpal
└─ account
   ├─ Account
   ├─ Activity
   ├─ AccountRepository
   ├─ SendMoneyService
   ├─ AccountRepositoryImpl
   └─ AccountController
```
- 기능으로 구분하여 package-private으로 패키지간의 경계를 결합하면 기능 사이의 불필요한 의존성을 방지할 수 있다.
- 계층에 의한 패키징 방식보다 가시성을 훨씬 떨어뜨린다. 도메인 코드가 영속성 코드에 의존하는 것을 막을 수 없다.

## 아키텍처적으로 표현력있는 패키지 구조
```
buckpal
└─ account
   ├─ adapter
   │  ├─ in
   │  │  └─ web
   │  │     └─ AccountController 
   │  └─ out
   │     └─ persistence
   │        ├─ AccountPersistenceAdapter
   │        └─ SpringDataAccountRepository
   ├─ domain
   │  ├─ Account
   │  └─ Activity  
   └─ application
      ├─ SendMoneyService
      └─ port
         ├─ in
         │  └─ SendMoneyUseCase
         └─ out
            ├─ LoadAccountPort
            └─ UpdateAccountStatePort
```
- SendMoneyService는 SendMoneyUseCase구현
- LoadAccountPort, UpdateAccountStatePort는 영속성 어댑터가 구현
- 패키지구조가 헷갈릴수 있지만 패키지명으로 원하는 기능을 바로 찾아낼 수 있다.
- 패키지 구조가 아키텍처를 반영할 수 없다면 시간이 지남에 따라 코드는 점점 목표하던 아키텍처로부터 멀어지게 될거다.
- 패키지에 들어있는 모든 클래스들은 port 인터페이스를 통하지 않고는 바깥에서 호출되지 않기 때문에 package-private 접근수준으로 둬도 된다.

## 의존성 주입의 역할
- 가장 본질적인 요건은 애플리케이션계층이 인커밍/아웃고잉 어댑터에 의존성을 갖지 않는 것이다.
- 포트 인터페이스를 구현한 실제 객체를 누가 애플리케이션 계층에 제공해야할까? 애플리케이션 계층에서 어댑터에 대한 의존성을 추가하고 싶지는 않다.
- 의존성 주입을 사용해 이를 해결할 수 있다. 모든 계층에 의존성을 가진 중립적인 컴포넌트를 하나 도입하여 대부분의 클래스를 초기화하는 역할을 한다.


## 유지보수 가능한 소프트웨어를 만드는데 어떻게 도움이 될까?
- 아키텍처의 특정요소를 찾으려면 패키지 구조를 탐색하면 된다.
