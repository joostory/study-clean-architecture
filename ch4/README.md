# 유스케이스 구현하기

도메인 엔티티를 만드는 것으로 시작하여 유스케이스를 구현

## 도메인 모델 구현하기
```kotlin
package buckpal.domain

class Account {
  var id: AccountId
  var baselineBalance: Money
  var activityWindow: ActivityWindow

  fun calculateBalance(): Money = Money.add(
    baselineBalance,
    activityWindow.calculateBalance(id)
  )

  fun withdraw(money: Money, targetAccountId: AccountId): Boolean {
    if (!mayWithdraw(money)) {
      return false
    }

    val withdrawal = Activity(
      id, id, targetAccountId, LocalDateTime.now(), money
    )
    activityWindow.addActivity(withdrawal)
    return true
  }

  private fun mayWithdraw(money: Money): Boolean = Money.add(
    calculateBalance(), money.negate()
  ).positive

  fun deposit(money: Money, sourceAccountId: AccountId): Boolean {
    val deposit = Activity(
      id, sourceAccountId, id, LocalDateTime.now(), money
    )
    activityWindow.addActivity(deposit)
    return true
  }
}
```
- Account 엔티티는 실제 계좌의 현재 스냅숏을 제공
- ActivityWindow에서 지난 며칠, 몇주간으 활동만 보관
- 총잔고는 baselineBalance의 모든 활동 잔고를 합한 값

## 유스케이스 둘러보기
유스케이스는 다음과 같은 단계를 따른다.

1. 입력을 받는다.
  - 도메인 로직에만 신경쓰고 입력유효성 검증으로 오염되면 안된다.
2. 비지니스 규칙을 검증한다.
  - 유스케이스는 비지니스 규칙을 검증할 책임이 있다. 도메인 엔티티와 이 책임을 공유한다.
3. 모델 상태를 조작한다.
  - 일반적으로 영속성 어댑터를 통해 이 상태를 저장할 수 있다. 혹은 또다른 아웃고잉 어댑터를 호출할 수 있다.
4. 출력을 반환한다.

```kotlin
package buckpal.application.service

@Transactional
class SendMoneyService(
  private val localAccountPort: LocalAccountPort,
  private val accountLock: AccountLock,
  private val updateAccountStatePort: updateAccountStatePort
): SendMoneyUseCase {
  override fun sendMoney(command: SendMoneyCommand) {
    // TODO: 비지니스 규칙 검증
    // TODO: 모델 상태 조작
    // TODO: 출력값 반환 
  }
}
```

## 입력 유효성 검증
- 입력 유효성 검증은 결국 애플리케이션 계층의 책임이다.
- 어댑터가 유스케이스에 전달하기 전에 검증하면 호출하는 모든 어댑터가 검증을 해야한다.
- 입력 유효성 검증은 입력 모델이 한다. 입력모델은 SendMoneyCommand이다.

```kotlin
package buckpal.application.port.in

class SendMoneyCommand {
  val sourceAccountId: AccountId?
  val targetAccountId: AccountId?
  val money: Money?

  constructor(sourceAccountId: AccountId?, targetAccountId: AccountId?, money: Money?) {
    this.sourceAccountId = sourceAccountId
    this.targetAccountId = targetAccountId
    this.money = money

    requireNonNull(sourceAccountId)
    requireNonNull(targetAccountId)
    requireNonNull(money)
    requireGreaterThan(money, 0)
  }
}
```
- SendMoneyCommand는 유스케이스 API의 일부이기 때문에 유효성검증이 애플리케이션 코어에 남아있지만 유스케이스를 오염시키지 않는다.
- 하지만 이것은 이미 Bean Validation API가 이를 대신해준다.

```kotlin
package buckpal.application.port.in

class SendMoneyCommand: SelfValidating<SendMoneyCommand> {
  val sourceAccountId: AccountId?
  val targetAccountId: AccountId?
  val money: Money?

  constructor(sourceAccountId: AccountId?, targetAccountId: AccountId?, money: Money?) {
    this.sourceAccountId = sourceAccountId
    this.targetAccountId = targetAccountId
    this.money = money
    requireGreaterThan(money, 0)
    this.validateSelf()
  }
}
```

## 생성자의 힘
- SendMoneyCommand의 생성자에 많은 책임을 지우고 있다. 유효하지 않는 상태의 객체를 만들 수 없다.
- 파라미터가 더 많다면 builder를 사용할 수 있다.

```kotlin
SendMoneyCommandBuilder()
  .sourceAccountId(AccountId(41L))
  .targetAccountId(AccountId(42L))
  .build()
```
- 실수로 빌더를 호출하는 코드에 필드 추가를 안하면 어떻게될까? 런타임에서는 에러를 알 수 있지만 컴파일러가 경고해주지 않는다.
- 생성자는 이를 컴파일러에서 경고해주고 어떤 값인지 IDE가 힌트를 줄 수도 있다.
- null 같은 경우는 코틀린의 nonnull을 사용하면 된다.

## 유스케이스마다 다른 입력 모델
- 다른 유스케이스에 같은 입력 모델을 사용하고 싶은 생각이 들 때가 있다.
- 문제는 입력 유효성을 검증할때다 커스텀 검증 로직을 만들어야 하고 이는 비니지스코드를 유효성 검증으로 오염시킨다.
- 각 유스케이스 전용 모델가 부수효과 발생하지 않게 한다.

## 비지니스 규칙 검증하기
- 입력 유효성 검증은 구문상의 유효성 검증
- 비지니스 규칙은 의미적인 유효성 검증
- 도메인 엔티티에서 구현할 수 있다.
- 도메인 엔티티를 사용하기 전에 해도 된다.

```kotlin
class SendMoneyService {
  override fun sendMoney(command SendMoneyCommand) {
    requireAccountExists(command.sourceAccountId)
    requireAccountExists(command.targetAccountId)
    ...
  }
}
```

## 풍부한 도메인 모델 vs 빈약한 도메인 모델
- 풍부한 도메인 모델에서는 withdraw, deposit과 같은 로직이 엔티티에 있다.
- 빈약한 도메인 모델에서는 엔티티에 getter, setter만 있고 유스케이스 클래스에 있다.
- 필요에 맞는 스타일로 선택해서 사용하면 된다.

## 유스케이스마다 다른 출력 모델
- 입력과 마찬가지로 출력도 구체적일수록 좋다.
- 춮력이 달라질때마다 그에 따른 유스케이스를 사용하는 것이 좋을지 정답은 없다.
- 유스케이스들끼리 출력을 공유하면 유스케이스도 강하게 결합된다.

## 읽기 전용 유스케이스는 어떨까?
- 읽기 전용 작업을 유스케이스라고 하는 것은 조금 이상하다.
- 이를 쿼리로 구현할 수 잇다.

```kotlin
package buckpal.application.service

class GetAccountBalanceService(
  private loadAccountPort: LoadAccountPort
): GetAccountBalanceQuery {
  override fun getAccountBalance(accountId: AccountId) {
    return localAccountPort.loadAccount(accountId, LcoalDateTime.now()).calculateBalance()
  }
}
```

## 유지보수 가능한 소프트웨어를 만드는데 어떻게 도움이 될까?
- 모델을 공유하는 것보다 더 많은 작업이 필요하지만 부수효과를 피할 수 있다.
- 이는 지속가능한 코드를 만드는데 큰 도움이 된다.
