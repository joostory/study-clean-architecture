# 아키텍처 요소 테스트하기

## 테스트 피라미드
- 시스템 테스트 -> 통합 테스트 -> 단위 데스트
- 기본전제: 비용 적음, 유지보수 쉬움, 빠른 실행, 높은 커버리지 유지
- 단위와 경계를 넘는 테스트는 비용이 비싸고 느리고, 깨지기 쉽다.
- 비싼 테스트일수록 커버리지 목표를 낮게 잡아야 한다. 테스트를 만드는데 시간이 많이 쓰기 때문이다.

## 단위 테스트로 도메인 엔티티 테스트하기

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/account/application/domain/AccountTest.java#L32
```java
class AccountTest {
	@Test
	void withdrawalSucceeds() {
		AccountId accountId = new AccountId(1L);
		Account account = defaultAccount()
				.withAccountId(accountId)
				.withBaselineBalance(Money.of(555L))
				.withActivityWindow(new ActivityWindow(
						defaultActivity()
								.withTargetAccount(accountId)
								.withMoney(Money.of(999L)).build(),
						defaultActivity()
								.withTargetAccount(accountId)
								.withMoney(Money.of(1L)).build()))
				.build();

		boolean success = account.withdraw(Money.of(555L), new AccountId(99L));

		assertThat(success).isTrue();
		assertThat(account.getActivityWindow().getActivities()).hasSize(3);
		assertThat(account.calculateBalance()).isEqualTo(Money.of(1000L));
	}
}
```
- 도메인 엔티티의 행동은 다른 클래스에 거의 의존하지 않기 때문에 다른 종류의 테스트는 필요하지 않다.

## 단위테스트로 유스케이스 테스트하기

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/account/application/service/SendMoneyServiceTest.java#L63
```java
class SendMoneyServiceTest {
	@Test
	void transactionSucceeds() {

		Account sourceAccount = givenSourceAccount();
		Account targetAccount = givenTargetAccount();

		givenWithdrawalWillSucceed(sourceAccount);
		givenDepositWillSucceed(targetAccount);

		Money money = Money.of(500L);

		SendMoneyCommand command = new SendMoneyCommand(
				sourceAccount.getId().get(),
				targetAccount.getId().get(),
				money);

		boolean success = sendMoneyService.sendMoney(command);

		assertThat(success).isTrue();

		AccountId sourceAccountId = sourceAccount.getId().get();
		AccountId targetAccountId = targetAccount.getId().get();

		then(accountLock).should().lockAccount(eq(sourceAccountId));
		then(sourceAccount).should().withdraw(eq(money), eq(targetAccountId));
		then(accountLock).should().releaseAccount(eq(sourceAccountId));

		then(accountLock).should().lockAccount(eq(targetAccountId));
		then(targetAccount).should().deposit(eq(money), eq(sourceAccountId));
		then(accountLock).should().releaseAccount(eq(targetAccountId));

		thenAccountsHaveBeenUpdated(sourceAccountId, targetAccountId);
	}
}
```
- given / when /then 으로 테스트를 구성했다.
- given에서 mockito의 given을 사용해 목객체의 행동을 정의했다.
- 유스케이스는 상태가 없기때문에 then에서 상태를 점검하는 대신 의존 대상 메서드와 상호작용했는지 검증한다.
- 이는 테스트가 행동뿐이라니 구조변경에도 취약해진다는 의미가 된다. 코드 리팩터링으로 테스트도 변경될 수 있다.
- 따라서 모든 동작을 검증하는 대신 핵심만 골라 테스트하면 클래스가 변경될때마다 테스트를 변경하지 않아도 된다.

## 통합 테스트로 웹 어댑터 테스트하기

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/account/adapter/in/web/SendMoneyControllerTest.java
```java
@WebMvcTest(controllers = SendMoneyController.class)
class SendMoneyControllerTest {

	@Autowired
	private MockMvc mockMvc;

	@MockBean
	private SendMoneyUseCase sendMoneyUseCase;

	@Test
	void testSendMoney() throws Exception {

		mockMvc.perform(post("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}",
				41L, 42L, 500)
				.header("Content-Type", "application/json"))
				.andExpect(status().isOk());

		then(sendMoneyUseCase).should()
				.sendMoney(eq(new SendMoneyCommand(
						new AccountId(41L),
						new AccountId(42L),
						Money.of(500L))));
	}
}
```
- 웹 어댑터의 책임 대부분은 이 테스트로 커버된다.
- 프레임워크를 믿고 프레임워크가 하는 부분은 테스트하지 않는다.
- 입력을 SendMoneyCommand로 매핑하는 전 과정은 다루고 있다.
    - 자체검증 커맨드로 유효성 검증
    - 유스케이스 호출 여부 검증
    - HTTP 응답 검증
- 왜 이것이 통합테스트인가?
    - 요청경로 검증
    - JSON 매핑 검증
    - 웹 컨트롤러가 프레임워크를 통해 잘 동작하는지 검증
- 웹 컨트롤러가 스프링 프레임어크에 강하게 묶여 있기 때문에 격리하기보다는 통합된 상태로 검증하는 것이 프로덕션에서의 작동을 검증할 수 있다.

## 통합 테스트로 영속성 어댑터 테스트하기

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/account/adapter/out/persistence/AccountPersistenceAdapterTest.java
```java
@DataJpaTest
@Import({AccountPersistenceAdapter.class, AccountMapper.class})
class AccountPersistenceAdapterTest {

	@Autowired
	private AccountPersistenceAdapter adapterUnderTest;

	@Autowired
	private ActivityRepository activityRepository;

	@Test
	@Sql("AccountPersistenceAdapterTest.sql")
	void loadsAccount() {
		Account account = adapterUnderTest.loadAccount(new AccountId(1L), LocalDateTime.of(2018, 8, 10, 0, 0));

		assertThat(account.getActivityWindow().getActivities()).hasSize(2);
		assertThat(account.calculateBalance()).isEqualTo(Money.of(500));
	}

	@Test
	void updatesActivities() {
		Account account = defaultAccount()
				.withBaselineBalance(Money.of(555L))
				.withActivityWindow(new ActivityWindow(
						defaultActivity()
								.withId(null)
								.withMoney(Money.of(1L)).build()))
				.build();

		adapterUnderTest.updateActivities(account);

		assertThat(activityRepository.count()).isEqualTo(1);

		ActivityJpaEntity savedActivity = activityRepository.findAll().get(0);
		assertThat(savedActivity.getAmount()).isEqualTo(1L);
	}

}
```
- 같은 이유로 영속성 테스트로 통합 테스트를 적용하는 것이 합리적이다.
- 이 테스트에서 DB를 모킹하지 않았다는 점이 중요하다.
- 스프링에서는 기본적으로 메모리 DB를 테스트에서 사용하나 프로덕션에서 환경의 차이로 문제가 생길 확률이 높다.
- 따라서 영속성 테스트는 실제 DB를 사용해야한다. [https://www.testcontainers.org/](TestContainer)같은 것을 사용해서..  
  (ㄷㄷㄷ 테스트에서 도커 막 시작함)

## 시스템 테스트로 주요 경로 테스트하기

https://github.com/thombergs/buckpal/blob/master/src/test/java/io/reflectoring/buckpal/SendMoneySystemTest.java
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class SendMoneySystemTest {

	@Test
	@Sql("SendMoneySystemTest.sql")
	void sendMoney() {

		Money initialSourceBalance = sourceAccount().calculateBalance();
		Money initialTargetBalance = targetAccount().calculateBalance();

		ResponseEntity response = whenSendMoney(
				sourceAccountId(),
				targetAccountId(),
				transferredAmount());

		then(response.getStatusCode())
				.isEqualTo(HttpStatus.OK);

		then(sourceAccount().calculateBalance())
				.isEqualTo(initialSourceBalance.minus(transferredAmount()));

		then(targetAccount().calculateBalance())
				.isEqualTo(initialTargetBalance.plus(transferredAmount()));

	}

	private ResponseEntity whenSendMoney(
			AccountId sourceAccountId,
			AccountId targetAccountId,
			Money amount) {
		HttpHeaders headers = new HttpHeaders();
		headers.add("Content-Type", "application/json");
		HttpEntity<Void> request = new HttpEntity<>(null, headers);

		return restTemplate.exchange(
				"/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}",
				HttpMethod.POST,
				request,
				Object.class,
				sourceAccountId.getValue(),
				targetAccountId.getValue(),
				amount.getAmount());
	}
}
```
- SpringBootTest는 스프링의 모든 객체 네트워크를 띄운다.
- 모킹을 하지 않고 실제 요청을 보내 프로덕션에 가깝게 테스트한다.
- 이런 시스템 테스트는 단위, 통합 테스트와 겹치는 부분이 많지만 발견하지 못하는 부분을 발견하게 해준다.
- 여러개의 유스케이스를 결합해서 시나리오를 만들때 더 빛이 난다.

## 얼마만큼의 테스트가 충분할까?
- 라인 커버리지는 100%가 아니면 무의미하다. 중요로직이 테스트되지 않았을 수 있기 때문이다.
- '마음편하게 배포할 수 있느냐?'를 성공기준으로 삼고 계속된 버그 수정과 그 부분에 대한 테스트 보강으로 점점 더 믿을 수 있는 테스트가 된다.
- 육각형 아키텍처에서 사용하는 테스트 전략
    - 도메인 엔티티: 단위테스트
    - 유스케이스: 단위테스트
    - 어댑터: 통합테스트
    - 사용자의 중요 어플리케이션 경로: 시스템 테스트
- 개발 후가 아닌 개발 중에 하면 귀찮은 일이 아닌 개발도구가 된다.
- 테스트를 고치는데 너무 시간이 소요되거나 리팩터링할때마다 수정해야한다면 테스트로서의 가치를 잃게 된다.
