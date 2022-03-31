---
layout: post
title:  Decrator Pattern with Spring Boot
subtitle: 데코레이터 패턴을 이용해 사전점검 프로그램에 어떻게 실제로 적용하는지 살펴봅니다
gh-repo: 6unpk/myrepository
gh-badge: [follow]
tags: [Spring Boot, Decroator]
comments: true
---

 안녕하세요 스마트닥터의 벡엔드 개발을 맡고 있는 박준우입니다. 데코레이터 패턴을 실제로 도메인 지식과 결합하여 어떻게 해결했는지 살펴보도록 합시다.

## Domain Problem
 처방내역에는 각각 처방에 대해 별도로 메모(기재)가 필요한 처방들이 몇몇 존재합니다. 이러한 처방들에 알맞게 부가적인 상세 설명이 제대로 들어갔는가를 점검하는 과정을 구현해야합니다. 예를 들어서 병원에 가서 '골밀도 검사'를 받는다고 가정해봅시다. '골밀도 검사'가 처방전에 기록되고 여기에는 덧붙여서 이 검사 결과에 대한 내용을 기재해야만 합니다. 이렇게 어떤 특정한 경우데 대해 추가적인 메모가 필요한 각 경우를 코드로 나눠 처방단위와 명세서 단위로 기록하는 부분을 '특정내역'이라 일컫습니다. 

 물론 모든 특정내역들이 이렇게 단순히 어떤 약이나 검사가 처방이 되었을 떄 해당 특정내역이 존재하는가를 체크하는 수준으로 간단하지 않습니다. 예를 들어서 다음은 JT018 특정내역에 대한 설명 부분입니다.

![Untitled]({{ site.baseurl }}/assets/images/decorator_pattern_in_spring/img1.png)

 만약 위의 설명을 토대로 검증 로직을 구성해야 한다면, 이 밖에도 존재하는 수십가지의 특정내역들을 처리하기 까다로울 수밖에 없습니다. 물론 모든 특정내역들에 대해 일일히 점검을 시도하진 않을 것이지만, 위의 JT018와 같이 특정한 몇개의 일부 특정내역들만 상세하게 검증할 필요가 있습니다. 따라서 이런 경우에 대해 생각해 볼 수 있는 구조적인 해결책은 다음과 같습니다.

 먼저 특정내역의 점검을 위해 다음과 같은 2단계로 나누어 생각합니다.

 1. 모든 처방에 대해 특정내역이 존재하는지 여부만을 체크하는 기본 점검 로직
 2. 일부 특정내역(JT018과 같은)에 대해서만 상세하게 검증하는 점검 로직

여기서 1번의 경우 모든 처방에 대해 동일하게 단순히 해당 처방에 대해 이 특정내역이 누락되었는가 아닌가 여부만을 판단합니다. 해당 특정내역이 요구하는 상세한 검증 로직은 건너뛰도록합니다. 그 이후 2번에 해당하는 특정내역이라면 검증을 시도합니다.

 위의 스텝을 구현하는데 있어 고려할 부분은 다음과 같습니다.
 
 1. 특정내역 값은 고정된 값이 아니라 DB상에 존재합니다. 따라서 특정내역 검증 로직의 추가는 런타임상에서 이뤄져야 합니다.
 2. 향후에 특정내역이 변경되거나 기획사항이 변경되어 검증할 특정내역 몇 개가 추가될 수 있습니다.
 3. 1번의 기본 점검을 수행한 후 2번의 상세 검증을 수행합니다. 

 종합적으로 위의 요소들을 고려하면 대략 데코레이터 패턴이나 템플릿 메서드 패턴과 전략 패턴등을 사용해 구조화 할 수 있을 것입니다. 여기서 주목해야할 부분은 기본의 점검에 덧붙여 상세검증을 추가한다는 것입니다. 덧붙인다는 말에서도 알 수 있듯이 이번의 문제를 해결하기 위해 데코레이터 패턴을 활용해보기로 하였습니다. 
 
  데코레이터 패턴의 대안으로 혹은 비교대상으로 자주 언급되는 패턴 중 템플릿 메서도 패턴과 전략패턴이 존재합니다. 두 패턴 모두 비슷하게 같은 객체에 대해 여러 기능을 구현하게 끔 한다는 공통점이 있습니다. 그러나 전략 패턴은 데코레이터 패턴과는 다르게 기본 구현 내용을 바꿔버리기 떄문에 알맞지 않으며 템플릿 메서드 패턴은 객체의 조합이 폭발적으로 늘어날 수 있다는 문제점이 있습니다.  

## Decroator Pattern
 데코레이터(Decorator) 패턴은 쉽게 말해 런타임상에서 객체에 추가적인 기능을 추가하는 역할을 하는 패턴입니다. 마치 스타벅스에서 커피를 주문하고 그 뒤에 드리즐과 모카시럽을 뿌려달라고 얘기하는 것과 비슷합니다. 

![Untitled]({{ site.baseurl }}/assets/images/decorator_pattern_in_spring/img4.jpg)

 실제로 스타벅스에 가서 커피를 주문받는다고 가정해봅시다. ```Beverage.kt``` 인터페이스는 음료를 제조하는 ```make``` 메서드를 가지고 있고 ```Coffee.kt``` 클래스는 커피를 제조하는 역할을 하며 ```Beverage.kt``` 인터페이스를 상속받습니다.

**```Beverage.kt```**
 
```kotlin
interface Beverage {
  fun make(): String
}
```

**```Coffee.kt```**

```kotlin
class Coffee: Beverage {
  override fun make(): String {
    return "Java Chip Frappuccino"
  }
}
```

여기까지 보면 단순한 커피 제조를 나타낸 따분한 코드이지만, 위에서 주문할 커피에 언급했듯이 드리즐과 모카시럽을 추가한다고 생각해봅시다. 이 때 각각 추가될 토핑을 데코레이터로 표현할 수 있습니다. 
데코레이터들의 추상 클래스를 정의하고 새로운 데코에이터를 만들게 되면 해당 추상클래스를 상속하는 데코레이터로 만듭니다. 

**```BaseCoffeeTopping.kt```**

```kotlin
class BaseCoffeeTopping(beverage: Beverage): Beverage {
  override fun make() {
    berverage.make()
  }
}
```

```BaseCoffeeTopping.kt``` 는 데코레이터들의 공통 부모 클래스입니다. 만약 새로운 토핑 클래스를 만들고 싶다면 ```BaseCoffeeTopping.kt``` 를 상속받는 새로운 토핑을 만들면 그만입니다. 아래 두 토핑 클래스인 ```Drizzle.kt```와 ```MochaSyrup.kt```는 각각 드리즐과 모카시럽 텍스트를 추가하는 역할을 수행하죠.

**```Drizzle.kt```**

```kotlin
class Drizzle(beverage: Beverage): BaseCoffeeTopping(beverage) {
  override fun make(): String {
    return super.make() + " with Drizzle"
  }
}
```

**```MochaSyrup.kt```**

```kotlin
class MochaSyrup(beverage: Beverage): BaseCoffeeTopping(beverage) {
  override fun make(): String {
    return super.make() + " with Mocha Syrup"
  }
} 
```
그리고 ```BaseCoffeeTopping.kt``` 도 마찬가지로 ```Coffee.kt``` 처럼 ```Beverage.kt``` 인터페이스를 상속합니다. 왜냐하면 우리가 제조할 커피 위에 토핑을 덧붙이는 과정이기 때문이죠. 따라서 
```Coffee.kt``` 의 make가 먼저 호출되어 자바칩 프라푸치노가 나오게 되고 선택적으로 토핑을 구현하면 드리즐과 모카시럽이 덧붙여 나오게 됩니다.

**```Main.kt```**

```kotlin
fun main() {
  val coffee = Coffee()
  print(coffee.make()) // Java Chip Frappuccino

  val coffeeWithDrizzle = Drizzle(Coffee)
  print(coffeeWithDrizzle.make()) // Java Chip Frappuccino with Drizzle

  val coffeeWithSomeTopping = MochaSyrup(coffeeWithDrizzle)
  print(coffeeWithSomeTopping.make()) // Java Chip Frappuccino with Drizzle with Mocha Syrup
}
```

구성된 커피 토핑 데코레이터를 테스트 해봅시다. 맨 처음에는 프라푸치노만 만들어지는 Coffee 객체를 하나 만듭니다. 이후 드리즐 토핑을 올리려면, 드리즐 토핑 생성시에 기존의 커피를 넣어주기만 하면 그만입니다. 모카 시럽을 넣는 경우도 마찬가지로 모카시럽에 '드리즐을 올린 커피'을 넣어주기만 하면 됩니다.


지금 까지 언급한 내용을 다이어그램으로 표현하면 아래와 같습니다.

![Untitled]({{ site.baseurl }}/assets/images/decorator_pattern_in_spring/img2.png)

## Decorator Pattern in Domain 

 위에서 정의한 패턴의 흐름을 그대로 다시 특정내역에 덧붙여서 생각하면 다음과 같습니다.

1. 커피(Coffee)            -> 특정내역
2. 음료 인터페이스(Beverage)  -> 특정내역 점검 인터페이스
3. 드리즐, 모카시럽(Drizzle, MochaSyrup) -> JT018과 같은 상세 검증이 필요한 특정내역들


 따라서 JT018이라는 특정내역 점검 로직을 기본 특정내역위에 덧붙여 수행할 수 있게됩니다. 위의 내용을 토대로 짤막히 코드를 작성해봅시다. 

**```PretestSymbol.kt```**

```kotlin
interface PretestSymbol {
  fun checkParticularsSymbol(format: String) 
}
```

**```Particulars.kt```**

```kotlin
// 특정내역 유무를 점검합니다
class Particulars: PretestSymbol { 
  override fun checkParticularsSymbol(format: String) {
    // 특정 내역 점검 시행을 구현
  }
}
```

**```BaseParticulars.kt```**

```kotlin
// 특정내역 데코레이터들의 Base 클래스입니다
class BaseParticulars(pretestSymbol: PretestSymbol): Particulars(pretestSymbol) { 
  override fun checkParticularsSymbol(format: String) {
    pretestSymbol.checkParticularsSymbol(format: String)
  }
}
```

**```JT018Symbol.kt```**

```kotlin
// JT018 특정내역 데코레이터 클래스입니다
class JT018Symbol: PretestSymbol { 
  override fun checkParticularsSymbol(format: String) {
    // JT018 특정내역을 점검 하는 부분에 대한 구현이 이뤄집니다.
    // 가령 여기서 입력된 'format에 대해 A코드가 맞게 들어갔는가?'와 같은 조건들을 검사하게 됩니다.
  }
}
```

여기서 주목해야할 부분은 ```JT018Symbol``` 클래스입니다. 아까 Domain Problem에서 언급한 JT018 특정내역의 검사 조건을 바로 ```checkParticularsSymbol``` 메서드 내에서 구현하여 검증하는 과정을 거칩니다.

마찬가지로 같은 맥락에서 다이어그램으로 표현하면 아래와 같을 것입니다.

![Untitled]({{ site.baseurl }}/assets/images/decorator_pattern_in_spring/img3.png)

## Decorator Pattern in Spring Boot

 일반적인 데코레이터 패턴은 객체를 생성하여 덧붙이는 방식으로 생성됩니다. 그러나 Spring 내에서는 DI로 구현하기 위해 Decorator를 Bean으로 등록하여 사용합니다. 따라서 위의 예제처럼 데코레이터 객체를 생성하는 방식이 아니라 의존성 주입이 가능하게 만들기 위해서는 약간의 변경이 필요합니다.
 

만약 Spring의 DI 방식이 아니라면 다음과 같이 사용해야겠지만, 문제는 이렇게 의존성을 주입하지 않고 쓰면 Spring의 다른 Bean들에 대해 의존성을 갖기 어려워집니다. 만약 disease 라는 병과 관련된 정보를 Repository로 부터 불러오려면 아래의 JT018 데코레이터에서는 참조가 불가능합니다.

```kotlin
// BDD with Kotest
class ParticularTest(): BehaviorSpec({
  Given("*") {
    When("*") {
      Then("*") {
        val partricularsTest = Particulars() // 먼저 특정내역 점검에 대한 객체를 생성하고

        val jt018Test = JT018Symbol(particularsTest) // JT018 특정내역 점검도 추가합니다

        jt018Test.checkParticularsSymbol(...) shouldbe true
      }
    }
  }
})
```
따라서 Spring 에 알맞게 조금씩 바꿔줘야 합니다. 그러기 위해선 다음과 같이 Decorator 내에서 Configuration을 만들고 JT018 데코레이터를 반환하는 Bean을 하나 만들어줍니다.
JT018 데코레이터가 Disease Repository로 부터 의존성을 주입 받는 다면 아래와 같이 그대로 넘겨줄 수 있습니다.

**```DecoratorConfiguration.kt```**

```kotlin
@Configuration
class DecoratorConfiguration {
    @Bean
    @Primary
    fun symbolPretestService(
        @Autowired
        particulars: Particulars,
        diseaseRepositroy: DrugsRepository,
    ): JT018SymbolPretest {
        return JT018SymbolPretest(particulars, diseaseRepository)
    }
}
```

Bean으로 등록되었기 떄문에 아래와 같이 DI를 쉽게 할수 있습니다. 만약 테스트 코드에서 사용한다고 가정시 다음과 같이 사용할 수 있습니다.  

```kotlin
// BDD with Kotest
class ParticularTest(private val jt018: JT018SymbolPretest): BehaviorSpec({
  Given("") {
    When("") {
      Then("") {
        jt018.checkParticularsSymbol(...) shouldbe true
      }
    }
  }
})
```



## Conclusion

 이로써 복잡한 도메인 지식의 문제를 디자인 패턴을 활용해 해결해보는 과정을 거쳐봤습니다. 사실 관련된 용어들이 복잡할 뿐이지 추상화하여 생각하면 그리 어렵지 않은 개념들입니다. 

## 참조

[Decorator Pattern](https://refactoring.guru/design-patterns/decorator)

[특정내역 리스트](https://m.blog.naver.com/39954/221836600195)


