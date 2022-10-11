# 아키텍처 경계 강제하기

- 시간이 지나면 아키텍처가 점점 무너진다.
- 계층간 경계가 약화되고 테스트하기 어려워진다.
- 새로운 기능 구현에 점점 더 시간이 많이 소요된다.
- 아키텍처 내의 경계를 강제하는 방법, 아키텍처의 붕괴를 맞서 싸우기위해 몇가지 조치를 살펴본다.

## 경계와 의존성
- 아키텍처의 경계를 강제한다는 것은 의존성이 올바른 방향을 향하도록 강제하는 것을 의미한다.
- 인커밍 어댑터 -> 입력포트 - 서비스 -> 엔티티
- 서비스 -> 출력포트 - 어댑터

## 접근제한자
- package-private (default)는 중요하다.
    - 자바 패키지를 통해 클래스들을 응집적인 모듈로 만들어준다.
    - 모듈의 진입점으로 활용될 클래스들만 골라서 public으로 만들면 의존성이 잘못된 방향을 가리키는 위험이 줄어든다.

```
buckpal
└─ account
   ├─ adapter
   │  ├─ in
   │  │  └─ web
   │  │     └─ o AccountController 
   │  └─ out
   │     └─ persistence
   │        ├─ o AccountPersistenceAdapter
   │        └─ o SpringDataAccountRepository
   ├─ domain
   │  ├─ + Account
   │  └─ + Activity  
   └─ application
      ├─ o SendMoneyService
      └─ port
         ├─ in
         │  └─ + SendMoneyUseCase
         └─ out
            ├─ + LoadAccountPort
            └─ + UpdateAccountStatePort
```
- o는 package-private, +는 public
- package-private는 작은 모듈에 효과적이다. 크기가 커지면 하위패키지를 만들게되고 이는 public으로 노출시켜야 한다.

## 컴파일 후 체크
- public을 사용하면 컴파일러가 의존성 방향을 확인하지 못한다.
- ArchUnit과 같은 도구를 통해 의존성을 체크할 수 있다.
    - https://www.archunit.org/
    - https://d2.naver.com/helloworld/9222129

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/DependencyRuleTests.java
```java
class DependencyRuleTests {

	@Test
	void validateRegistrationContextArchitecture() {
		HexagonalArchitecture.boundedContext("io.reflectoring.buckpal.account")

				.withDomainLayer("domain")

				.withAdaptersLayer("adapter")
				.incoming("in.web")
				.outgoing("out.persistence")
				.and()

				.withApplicationLayer("application")
				.services("service")
				.incomingPorts("port.in")
				.outgoingPorts("port.out")
				.and()

				.withConfiguration("configuration")
				.check(new ClassFileImporter()
						.importPackages("io.reflectoring.buckpal.."));
	}

	@Test
	void testPackageDependencies() {
		noClasses()
				.that()
				.resideInAPackage("io.reflectoring.reviewapp.domain..")
				.should()
				.dependOnClassesThat()
				.resideInAnyPackage("io.reflectoring.reviewapp.application..")
				.check(new ClassFileImporter()
						.importPackages("io.reflectoring.reviewapp.."));
	}

}
```
- 실패에 안전하지 않다. buckpal 이라는 패키지명에 오타를 내면 이는 찾지 못할 것이다.
- 패키지명을 리팩터링해도 테스트 전체가 무의미해진다.

## 빌드 아티팩트
- 빌드 아티팩트는 빌드 프로세스의 결과물이다.
- 빌드 도구의 주요한 기능 중 하나는 의존성 해결이다.
    - 코드베이스가 의존하고 있는 모든 아티팩트가 사용가능한지 확인
    - 불가능한 것이 있다면 리포지토리에서 가져오려고 시도
    - 실패하면 빌드 실패
- 이를 활용해 모듈와 아키텍처 계층간의 의존성을 강제할 수 있다.
    - 각 모듈 혹은 계층에 대해 전용 코드베이스와 빌드 아티팩트로 분리된 빌드 모듈(JAR)을 만든다.
    - 모듈의 빌드 스크립트에서 아키텍처에서 허용하는 의존성만 지정
    - 잘못된 의존성을 만들고 싶어도 클래스패스에 존재하지도 않게 된다.
- 장점
    - 빌드도구가 순환 의존성을 싫어하므로 순환의존성이 없음을 확신할 수 있다.
    - 다른 모듈을 고려하지 않고 특정 모듈의 코드를 격리한채로 변경할 수 있다.
    - 새로 의존성을 추가하는 일은 우연이 아닌 의식적인 행동이 된다.
- 단점
    - 빌드 스크립트를 유지보수하는 비용을 수반
    - 아키텍처가 어느정도 안정된 상태여야 한다.

## 생각해 볼 부분
- 이렇게까지?
- 언제 아키텍처가 안정되는건가? 안정된 상태는 거의 프로덕션에 준한다. 프로덕션에 준하게 안정된 상태에서 모듈을 분리하는 대규모 리팩터링을 할 수 있을까?
- package-private은 kotlin에 없다. 다른 방식으로 접근제어를 해야한다. 어떻게?
