
## 코틀린에서 클래스를 다루는 방법 

### 클래스와 프로퍼티 
자바에선 필드를 따로 생성하고, 롬복을 통해 getter, setter를 만들어 줬던 경험이 있습니다. 
하지만, 코틀린에선 필드만 만들면 getter, setter를 자동으로 만들어 줍니다.

```kotlin
fun main() {
    val person = Person("홍길동", 30)
    // getter, setter 자동으로 만들어준다.
    println(person.name)
    person.age = 10
    println(person.age)
}

class Person (val name: String, var age: Int)
```

위와 같이 Person 클래스를 만들어 보았습니다. ```.필드```를 통해 getter, setter를 바로 호출 할 수 있습니다. name은 val로 설정하여 변경할 수 없도록 하였습니다. 

### init 블록 
자바에선 생성자에서 값을 검증하였는데, 그렇다면 코틀린에선 어디서 값을 검증해야 할까요? 코틀린에선 ```init``` 블록에서 검증합니다. 생성자가 호출되는 시점에 init 블록이 함께 호출됩니다. 

```kotlin
class Person (
    val name: String, 
    var age: Int
) {
    
    // 초기화 블록을 통해 검증 가능 
    init {
        if (age < 0) {
            throw IllegalArgumentException("나이는 ${age}일 수 없습니다.")
        }
        println("초기화 블록")
    }
    
    constructor(name: String): this(name, 20) {
        println("부 생성자1")
    }

    constructor(): this("임꺽정") {
        println("부 생성자2")
    }
}
```
위에서 ```Person``` 클래스는 init 블록에서 값을 검증하고 있습니다. 그리고 여러개의 부생성자를 가지고 있습니다. 부생성자는 다른 생성자를 호출하지만 최종적으로는 주생성자를 this로 호출합니다.
부생성자를 통해 기본값을 설정할 수 있음을 확인할 수 있습니다. 하지만 개인적으로 기본값을 설정해야 한다면 default parameter 기능을 활용하는 것이 좋다고 생각합니다. 


## 커스텀 getter, setter 

자바에선 객체의 속성을 나타낼때, 아래와 같은 메서드를 사용하곤 했습니다. 
```java
public boolean isAdult() {
    return this.age >= 20;
}
```
코틀린에선 커스텀 getter, setter를 이용해서 객체의 속성을 추가할 수도 있고, 자신의 값을 변경할 수도 있습니다. 

```kotlin
fun main() {
    val person = Person("홍길동", 25)
    // isAdult 커스텀 getter 사용 
    println(person.isAdult)
    println(person.name)
}

class Person(
    name: String,
    var age: Int
) {

    val isAdult: Boolean
        get() = this.age >= 20
    
    // name의 값을 대문자로 변경 
    var name: String = name
        // 자신의 속성을 변경시키는 커스텀 getter
        get() = field.uppercase()
        // 속성을 변경해서 저장하는 커스텀 setter
        set(value) {
            field = value.uppercase()
        }
}
```
위의 예시를 보면 isAdult 라는 커스텀 getter를 추가하여, 객체에 새로운 속성을 추가하였습니다. 
name 필드에 대해서는 값을 변경하는 용도로 커스텀 getter를 사용하였습니다. 이때 blocking field를 이용하는 것을 볼 수 있습니다. 이렇게 bloking field로 접근하지 않고 직접 필드에 접근하면 getter로 접근하게 되고 무한루프가 발생하기 때문입니다.

```kotlin
val name: String = name
    // name필드를 호출하면 getter를 호출하면서 무한루프 발생!    
    get() = name.uppercase()
```

## 코틀린에서 접근 제어를 다루는 방법 

자바에서는 다음과 같은 접근 제어가 있었습니다. 
- public: 모든 곳에서 접근 가능 
- protected: 같은 패키지 또는 하위 클래스에서만 접근 가능 
- default: 같은 패키지에서만 접근 가능 
- private: 같은 클래스에서만 접근 가능 

코틀린에서는 자바와 접근 제어가 조금은 달라졌습니다. 
다음과 같은 접근 제어를 제공합니다. 
- public: 모든 곳에서 접근 가능 
- protected: 선언된 클래스 또는 하위 클래스에서만 접근 가능 
  - 코틀린 파일에서는 protected 사용 불가 (클래스가 아니기 때문에 상속 받을 하위 클래스가 없음)
  - 자바에서는 protected 접근 제어자는 같은 패키지에서 접근 가능하기 때문에, 자바와 함께 사용할 때는 주의가 필요 
- internal: 같은 모듈에서만 접근 가능 
  - 같은 build.gradle.kts 단위로 관리되는 유닛에서만 접근할 수 있습니다. 
- private: 같은 클래스에서만 접근 가능 

자바와 비교해보면, default 접근 제어자가 사라지고, internal 접근 제어자가 생겼습니다. 
여기서 알 수 있는 중요한 점은  **코틀린은 패키지를 namespace를 관리하기 위한 용도록만 사용하고, 가시성 제어에는 사용하지 않는다는 것입니다.**

또한 자바는 접근 지시어를 따로 적지 않으면, default 접근 제어자가 기본이었습니다. 하지만 코틀린에선 public 접근 제어자가 기본입니다. 

그리고 이전에 말씀 드린 커스텀 getter, setter 에도 각각 다른 접근 제어자를 붙여줄 수 있습니다. 
```kotlin
var name: String = name
        get() = field.uppercase(Locale.getDefault())
        // setter에만 private 접근 제어자 추가  
        private set(value) {
            field = value.uppercase(Locale.getDefault())
        }
```


## 코틀린에서 상속을 다루는 방법 

### 추상 클래스 

코틀린에서는 상속을 다음과 같이 다룹니다. 
```kotlin
abstract class Animal(
    protected val species: String,
    // override 하기 위해선 open 키워드가 붙어야 함
    protected open val legCount: Int // (1)
) {

    abstract fun move()
}

class Cat(
    species: String
) : Animal(species, 4) { // (2)

    override val legCount: Int // (3)
        get() = super.legCount + this.wingCount
    
    // override 어노테이션이 없다.
    override fun move() { // (4)
        println("고양이가 사뿐 사뿐 걸어갑니다.")
    }
}
```
(1): 자식 클래스에서 재정의하기 위해서는 open 키워드를 꼭 붙여야합니다. 
(2): ```Cat``` 클래스는 부모 클래스인 ```Animal``` 클래스를 상속합니다. 자바와는 달리 extends 키워드를 사용하지 않고 ```:```를 사용하여 상속 클래스를 지정합니다.
(3): 부모 클래스의 open 이 붙은 필드에 한해서, 자식 클래스에서 재정의가 가능합니다. 
(4): 자식 클래스에서 부모 클래스의 메서드를 재정의 하기 위해선 override 키워드를 붙여햐 합니다. 

### 인터페이스 

코틀린의 인터페이스는 아래와 같습니다. 
```kotlin
interface Swimable {
    
  fun act() {
      println("어푸 어푸")
  }
}

interface Flyable {
    
  fun act() {
      println("파닥 파닥")
  }
}
```
디폴트 메서드를 따로 선언해줄 필요 없이, 바디가 있으면 디폴트 메서드가 됩니다. 
구현은 다음과 같이 합니다. 
```kotlin
class penuin(
  species: String  
): Animal(species, 2), Swimable, Flyable {
    
  override fun act() {
      // 중복되는 인터페이스를 특정할 때 super<타입>.함수 사용 
      super<Swimable>.act()
      super<Flayble>.act()
  }
}
```
default 메서드가 중복된다면 위와 같이, ```super<타입>.함수``` 방식으로 인터페이스를 특정할 수 있습니다. 

코틀린의 인터페이스 에서는 backing field가 없는 프로퍼티를 인터페이스에 만들 수 있습니다. 
```kotlin
interface Swimmable {

    val swimAbility: Int
        get() = 3

    // 인터페이스에서 backing field 참조에 의한 컴파일 에러 
//    val swimAbility: Int
//        get() = field
}
```

### 클래스 상속받을 때 주의할 점 
상위 클래스를 설계할 때 생성자 또는 초기화 블록에 사용되는 프로퍼티에는 open을 피해야 합니다. 
다음 예시를 통해 알아보겠습니다. 
```kotlin
open class Parent {
    open val message: String = "Parent Message"

    init {
        println("Parent init: $message") // 하위 클래스의 오버라이드된 값이 예상대로 초기화되지 않을 수 있음
    }
}

class Child : Parent() {
    override val message: String = "Child Message"
}
```

부모 클래스가 초기화 되고, 다음으로 자식 클래스가 초기화 됩니다. 따라서 open 키워드가 붙은 키워드를 부모 클래스의 init 블록에서 사용하게 된다면 자식 클래스가 아직 초기화 되지 않은 상태이기 때문에 ```message```가 null로 나옵니다.

## 코틀린에서 object 키워드를 다루는 방법 

코틀린에서는 자바의 정적 필드와 정적 메서드는 어떻게 구현할까요? 코틀린에선 static 대신 companion object를 이용합니다. 
```kotlin
class Person private constructor(
    private val name: String, 
    private val age: Int
) {
    
    companion object { 
      private val MIN_AGE = 0
      private const val CONST_MIN_AGE = 10
      fun newBaby(name: String): Person {
          return Person(name, MIN_AGE)
      }
    }
}
```

companion object는 클래스와 동행하는 유일한 오브젝트라는 뜻입니다. 
여기서 상수는 ```const```를 붙이는게 좋습니다. 그러면 컴파일 시점에 실제 상수명이 숫자로 치환됩니다, 따라서 클래스 로딩 없이 곧바로 상수값을 접근할 수 있어 효율적입니다.

여기서, companion object, 즉 동반객체도 하나의 객체로 간주됩니다. 따라서 이름을 붙일 수도 있고, interface를 구현할 수도 있습니다.

```kotlin
fun main() {
  val person = Person.Factory.newBaby("A")
}

class Person private constructor(
  private val name: String,
  private val age: Int
) {

  companion object Factory : Log {
    private const val MIN_AGE = 0
    fun newBaby(name: String): Person {
      return Person(name, MIN_AGE)
    }

    override fun log() {
      println("LOG")
    }
  }
}
```

자바에서 코틀린의 companion object를 사용하려면 ```@JvmStatic```을 붙여야 합니다. 그럼 정적 메서드 처럼 접근이 가능합니다. 
```kotlin
companion object {
      @JvmStatic
      fun newBaby(name: String): Person {
          return Person(name, MIN_AGE)
      }
}
```

### 익명 클래스 

코틀린에선 익명 클래스를 구현할 때, object 키워드를 사용합니다. 

```kotlin
fun main() {
    moveSomeThing(object: Movable {
      override fun move() {
          println("움직인다.")
      }
      
      override fun fly() {
          println("난다.")
      }
    })
}

private fun moveSomeThing(movable: Movable) {
    movable.move()
    movable.fly()
}
```

## 코틀린에서 중첩 클래스를 다루는 방법 

코틀린에도 자바 처럼 중첩 클래스를 만들 수 있습니다. 다만, 다른 점은, 자바는 기본적으로 외부 클래스의 참조를 가진 내부 클래스를 만들었다면, 코틀린의 중첩 클래스는 바깥 클래스에 대한 참조를 가지고 있지 않습니다. 

```kotlin
class House(
    var address: String, 
    var livingRoom: LivingRoom = LivingRoom(10.0)
) {
    
  // 바깥 클래스인 House 클래스에 대한 참조를 가지고 있지 않다.   
  class LivingRoom(
      private var area: Double,
  )
}
```

만약 바깥 클래스에 대한 참조를 가진 중첩 클래스를 만드려면 ```inner``` 키워드를 붙여 클래스를 만들어야 합니다. 
```kotlin
class House(
    var address: String,
) {
  var livingRoom: LivingRoom = this.LivingRoom(10.0)
    
  // 바깥 클래스인 House 클래스에 대한 참조를 가지고 있지 않다.   
  inner class LivingRoom(
      private var area: Double,
  ) {
      val address: String
        // 바깥 클래스 인스턴스 멤버 참조
        get() = this@House.address
  }
}
```

## 코틀린에서 다양한 클래스를 다루는 방법 

### Data 클래스 
코틀린에서 계층간의 데이터를 전달하기 위한 DTO 클래스는 어떻게 만들어야 할까요?
- 데이터(필드)
- 생성자, getter
- equals, hashCode
- toString

코틀린에선 ```data``` 키워드를 통해 쉽게 데이터를 전달하기 위한 DTO 클래스를 만들 수 있습니다. 
```kotlin
data class PersonDto(
    val name: String, 
    val age: Int
)
```
이렇게 data 키워드를 붙여주면 equals, hashCode, toString을 자동으로 만들어줍니다. 

### Enum 클래스 
코틀린에서 enum 클래스는 자바와 동일합니다. 
```kotlin
enum class Country(
    val code: String
) {
    KOREA("KO"),
    AMERICA("US"),
  ;
}
```

Enum 클래스는 Sealed 클래스와 마찬가지로 When과 함께 사용할 때, 진가를 발휘합니다. 
```kotlin
private fun handleCountry(country: Country) {
    when (country) {
      Country.KOREA -> TODO()
      Country.AMERICA -> TODO()
    }
}
```
여기서 컴파일러가 이미 country의 모든 타입을 알고 있어, 나머지 타입에 대한 로직(else)을 작성하지 않아도 됩니다. 그리고 만약 Enum에 새로운 타입이 추가된다면 알람을 통해 when에 추가적인 처리를 해야한다는 것을 알 수 있습니다.

### Sealed 클래스 

Sealed 클래스는 코틀린에만 존재하는 개념입니다. 
주로 상속이 가능한 클래스를 만들고 싶은데, 외부에서는 해당 클래스를 상속받지 않았으면 하는 경우 Sealed 클래스를 만듭니다. 즉 하위 클래스를 봉인하는 것입니다. 

Sealed 클래를 이용하면, 런타임때 클래스 타입이 새로 추가될 수 없습니다. 즉 처음에 컴파일 타임때 만들어졌던 하위 클래스만 이용합니다. 또한 같은 파일 내에서만 하위 클래스를 지정할 수 있습니다. 

```kotlin
sealed class ApiResult<out T> {
    data class Success<out T>(val data: T) : ApiResult<T>()
    data class Failure(val errorMessage: String) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}
```

sealed 클래스도 enum 클래스와 마찬가지로  when과 함께 사용되었을 때 좋습니다. 컴파일 타임에 하위 타입을 모두 알고 있기 때문에, 새로운 타입이 추가된다면 컴파일 시점에 알려줍니다. 
```kotlin
fun handleResult(result: ApiResult<String>) {
    when (result) {
        is ApiResult.Success -> println("성공: ${result.data}")
        is ApiResult.Failure -> println("실패: ${result.error}")
        ApiResult.Loading -> println("로딩 중...")
    }
}
```

코틀린에 대한 기본기 세번째 포스팅입니다. 다음 포스팅에도 코틀린의 기본 문법에 대해 이야기해보겠습니다. 