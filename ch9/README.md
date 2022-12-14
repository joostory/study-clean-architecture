# 애플리케이션 조립하기

## 왜 조립까지 신경써야할까?
- 왜 유스케이스와 어댑터를 필요할때 인스턴스화하면 안되는걸까?
    - 의존성이 올바른 방향으로 가리키게 하기 위해서
    - 유스케이스는 인터페이스만 알아야 하고 런타임에 이 구현을 제공받아야 한다.
- 이는 코드를 훨씬 더 테스트하기 쉽게 만든다.
    - 필요로하는 객체를 목으로 전달받을 수 있으면 단위 테스트를 생성하기 쉬워진다.
- 객체 인스턴스를 생성할 책임은 설정 컴포넌트에 있어야 한다.
    - 설정 컴포넌트는 모든 곳에 접근할 수 있다.
    - 단일 책임 원칙을 위반하나 이는 나머지 다른 부분을 위함

## 평범한 코드 조립하기
- 평범하게 필요한 인스턴스를 생성하여 필요로 하는 각 생성자에 주입해주는 코드를 만든다고 가정했을때
- 이 모두는 public이어야 한다. 유스케이스가 영속성 어댑터에 접근하는 것을 막도록 package-private로 막을 수 없다.
- 이런 제한을 제공하면서 의존성 주입을 해주는 프레임워크가 있다. 스프링이 대표적이다.

## 스프림의 클래스패스 스캐닝으로 조립하기
- 스프링은 클래스패스에서 접근 가능한 모든 클래스를 확인해서 @Component 애너테이션이 붙은 클래스를 찾고 객체를 생성한다.
- 직접 애너테이션을 만들 수도 있다.

```java
@Targe({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface PersistenceAdapter {
  @AliasFor(annotation = Component.class)
  String value() default "";
}
```
- 이제 @Component대신 @PersistenceAdapter로 더 명확하게 영속성 어댑터임을 표시할 수 있다.
- 단점
    - 클래스에 프레임워크에 특화된 애너테이션을 붙여야 한다.
    - 라이브러리나 프레임워크를 만드는 입장에서는 의존성에 엮이게 되어 피해야 한다.
    - 스프링 전문가가 아니라면 원인을 찾는데 오래 걸릴 수 있다.
    - 애플리케이션 컨텍스트에 실제로는 올라가지 않았으면 하는 클래스가 있을 수 있다. (vault 데펜던시를 추가하는 것만으로도 AutoConfiguration에 의해 vault 서버를 지정하지 않으면 오류가 발생한다.)

## 스프링의 자바 컨피그로 조립하기
- @Configuration내에 @Bean으로 Adapter를 직접 생성한다. Repoisitory는 EnableJpaRepositories로 스캐닝한다.
- @Component없는 애플리케이션 계층을 만들 수 있다.
- 단점
    - 설정 클래스가 접근할 수 있도록 public이거나 설정 클래스가 같은 패키지안에 있어야 한다.
    - 그런데 또 이렇게 하면 하위 패키지를 이용할 수 없다.

## 생각해 볼 부분
- 직접 빈 등록하는거 하지 않으려고 스프링 컴포넌트 스캐닝하는거 아니었나?
