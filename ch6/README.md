# 영속성 어댑터 구현하기

## 의존성 역전
- 영속성 어댑터는 아웃고잉 어댑터
- 영속성 코드는 포트의 계약을 만족하는한 마음껏 수정해도 어플리케이션에 영향을 주지 않는다.


## 영속성 어댑터의 책임

1. 입력을 받는다.
2. 입력을 데이터베이스 포맷으로 매핑한다.
  - dto -> entity    
3. 입력을 데이터베이스로 보낸다.
4. 데이터베이스 출력을 애플리케이션 포맷으로 매핑한다.
  - entity -> dto
  - 이때 dto는 애플리케이션 코어에 위치한다.
5. 출력을 반환한다.

## 포트 인터페이스 나누기

- 일반적으로 하나의 엔티티가 필요로 하는 모든 db연산을 하나의 리포지토리 인터페이스에 넣어둔다.
- 각 서비스는 하나의 메서드만 사용하더라도 넓은 인터페이스에 의존성을 갖게된다.
- 하나의 일을 하는 포트로 분리하는 것이 좋다.

## 영속성 어댑터 나누기

- 도메인 클래스(DDD에서의 애그리거트)하나당 하나의 어댑터를 구현하는 방식을 선택할 수 있다.
- 도메인 경계를 따라 자동으로 나뉘어 지고 이는 바운디드 컨텍스트의 영속성 요구사항을 분리하기 위한 좋은 토대가 된다.

## 스프링 데이터 JPA 예제

https://github.com/thombergs/buckpal/blob/master/src/main/java/io/reflectoring/buckpal/account/domain/Account.java
```
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Account {

	/**
	 * The unique ID of the account.
	 */
	@Getter private final AccountId id;

	/**
	 * The baseline balance of the account. This was the balance of the account before the first
	 * activity in the activityWindow.
	 */
	@Getter private final Money baselineBalance;

	/**
	 * The window of latest activities on this account.
	 */
	@Getter private final ActivityWindow activityWindow;

	/**
	 * Creates an {@link Account} entity without an ID. Use to create a new entity that is not yet
	 * persisted.
	 */
	public static Account withoutId(
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(null, baselineBalance, activityWindow);
	}

	/**
	 * Creates an {@link Account} entity with an ID. Use to reconstitute a persisted entity.
	 */
	public static Account withId(
					AccountId accountId,
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(accountId, baselineBalance, activityWindow);
	}

	public Optional<AccountId> getId(){
		return Optional.ofNullable(this.id);
	}

	/**
	 * Calculates the total balance of the account by adding the activity values to the baseline balance.
	 */
	public Money calculateBalance() {
		return Money.add(
				this.baselineBalance,
				this.activityWindow.calculateBalance(this.id));
	}

	/**
	 * Tries to withdraw a certain amount of money from this account.
	 * If successful, creates a new activity with a negative value.
	 * @return true if the withdrawal was successful, false if not.
	 */
	public boolean withdraw(Money money, AccountId targetAccountId) {

		if (!mayWithdraw(money)) {
			return false;
		}

		Activity withdrawal = new Activity(
				this.id,
				this.id,
				targetAccountId,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(withdrawal);
		return true;
	}

	private boolean mayWithdraw(Money money) {
		return Money.add(
				this.calculateBalance(),
				money.negate())
				.isPositiveOrZero();
	}

	/**
	 * Tries to deposit a certain amount of money to this account.
	 * If sucessful, creates a new activity with a positive value.
	 * @return true if the deposit was successful, false if not.
	 */
	public boolean deposit(Money money, AccountId sourceAccountId) {
		Activity deposit = new Activity(
				this.id,
				sourceAccountId,
				this.id,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(deposit);
		return true;
	}

	@Value
	public static class AccountId {
		private Long value;
	}
}
```
- Account는 최대한 불변성을 유지하려 한다.
- 유효하지 않은 도메인 모델을 생성할 수 없다. (팩토리 메서드를 제공, 유효성검증)

JPA를 사용하므로 Entity 애너테이션이 추가된 클래스도 필요하다
https://github.com/thombergs/buckpal/blob/master/src/main/java/io/reflectoring/buckpal/account/adapter/out/persistence/AccountJpaEntity.java
```
@Entity
@Table(name = "account")
@Data
@AllArgsConstructor
@NoArgsConstructor
class AccountJpaEntity {

	@Id
	@GeneratedValue
	private Long id;

}
```

```
@Entity
@Table(name = "activity")
@Data
@AllArgsConstructor
@NoArgsConstructor
class ActivityJpaEntity {

	@Id
	@GeneratedValue
	private Long id;

	@Column
	private LocalDateTime timestamp;

	@Column
	private Long ownerAccountId;

	@Column
	private Long sourceAccountId;

	@Column
	private Long targetAccountId;

	@Column
	private Long amount;

}
```
- JPA의 연관관계를 사용하지 않았는데 사용하다보면 즉시로딩, 지연로딩, 캐싱등을 저주하게 될지도 모른다.
- 좀 더 간단한 도구를 원하게 될 수도 있다. (?? 이 얘기 왜하지?)

- 영속성 Adapter는 아웃고잉포트를 구현한다.
- Account <-> AccountJpaEntity 와 같은 매핑을 하게되는데 이를 사용하지 않는 방법을 선택할 수 있다.
- 그러나 JPA의 ManyToOne 등의 연관관계때문에 원치않는 데이터도 가져오게 된다. 그래서 매핑을 사용하는 것이 좋다.

## 데이터베이스 트랜잭션은 어떻게 해야할까?

- 트랜잭션은 유스케이스에 대해 일어나는 모든 쓰기 작업에 걸쳐 있어야 한다. 그래서 실패하는 다같이 롤백할 수 있다.
- 영속성 어댑터는 어떤 연산이 유스케이스에 포함되는지 모르기에 호출하는 서비스에 위임해야한다.
