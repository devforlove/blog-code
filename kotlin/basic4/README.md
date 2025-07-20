## 코틀린에서 배열과 컬렉션을 다루는 방법 

### 배열 

코틀린에선 배열을 다음과 같이 사용합니다. 
```kotlin
val array = arrayOf(100, 200)
for (i in array.indices) {
    println("${i} ${array[i]}")
}

for ((idx, value) in array.withIndex()) {
    println("인덱스: ${idx} :: 값: ${value}")
}

```
첫 번째 for 문에서는 ```array.indices```를 이용해 배열의 인덱스를 가져오도록 하였습니다. 
두 번째 for 문에서는 ```array.withIndex```를 이용해 인덱스와 값을 한 번에 가져오도록 하였습니다.


### 컬렉션 

코틀린에서 컬렉션을 만들 때 불변인지, 가변인지 설정해야 합니다. 

```kotlin
// 불변 리스트 생성 
val umModifiableList = listOf(100, 200)
// 가변 리스트 생성 
val mutableList = mutableListOf(100, 200)
// 값 추가 가능 
mutableList.add(300)
```
보통 ```mutable***of```와 같은 네이밍을 갖고 있다면 가변 컬렉션을 만드는 함수입니다.
위의 예제에서 ```mutableListOf``` 함수는 가변 ArrayList를 생성합니다. 

```mutableListOf``` 함수를 확인해보면, 가변리스트인 MutableList 타입의 컬렉션을 생성하는 것을 확인할 수 있습니다. 이는 코틀린에서 가변 리스트 타입과 불변 리스트 타입을 나눈다는 것을 알 수 있습니다. 
```kotlin
public fun <T> mutableListOf(vararg elements: T): MutableList<T> =
    if (elements.size == 0) ArrayList() else ArrayList(ArrayAsCollection(elements, isVarargs = true))
```

코틀린에서 Map은 다음과 같이 사용합니다. 
```kotlin
// 불변 맵 생성 
mapOf(1 to "MONDAY", 2 to "TUESDAY")

// 가변 맵 생성 
val map = mutableMapOf<Int, String>()
map[1] = "MONDAY"
map[2] = "TUESDAY"

// 반복 방법 
for (key in map.keys) {
    println(key)
    println(map[key])
}

for ((key, value) in map.entries) {
    println(key)
    println(value)
}
```

컬렉션을 이용할 때 주의할 점이 있습니다. 만약 코틀린에서 만든 컬렉션을 자바에서 활용한다면 코틀린과, 자바의 차이에 의해 오동작이 발생할 수 있습니다. 

위의 예제에서 봤듯이, 코틀린에선 가변 컬렉션과 불변 컬렉션 타입이 나뉘어져 있습니다. 하지만 자바에선 따로 타입을 구분하진 않습니다. 따라서 코틀린 코드를 자바에서 실행하게 된다면, 불변 컬렉션에 값을 추가하는 것이 가능합니다. 
만약 그것을 모르고 자바에서 코틀린의 불변 컬렉션에 값을 넣는다면 의도치 않은 버그를 만들 수도 있습니다. 

또한 자바의 컬렉션을 가져오면, 플렛폼 타입으로 가져오게 됩니다. 이때 코틀린에선 해당 타입이 nullable 인지, non-null 인지 알 수 없습니다. 이 경우에도 코틀린 코드 상에서 의도치 않은 NPE가 발생할 수 있습니다.

이런 문제를 방지하기 위해서는, 자바 코드를 보며 맥락을 확인하고 자바 코드를 가져오는 지접을 wrapping 해야 합니다. 

## 코틀린에서 다양한 함수를 다루는 방법 

### 확장함수
코틀린은 자바와 100% 호환하는 것을 목표로 하고 있습니다. 따라서 자바 코드 위에 자연스럽게 코틀린 코드를 추가하고자 하는 요구가 있었습니다. 
이러한 요구에 의해 나타난 것이 확장함수입니다. 

확장함수는 자바 클래스 안에 있는 함수처럼 호출할 수 있지만, 함수는 밖에서 만들 수 있습니다. 

```kotlin
fun main() {
    val str = "ABC"
    println(str.lastChar())
}

fun String.lastChar(): Char {
    return this[this.length - 1] // this를 이용해 인스턴스에 접근 가능 
}
```

String 클래스에 ```lastChar```라는 확장함수를 정의하였습니다. 물론 해당 함수는 String 클래스에 존재하지 않는 함수이지만 마치 String 클래스에 있는 것처럼 사용할 수 있습니다. 
예시를 보면 this를 통해 String 인스턴스에 접근할 수 있습니다. 

만약 멤버함수와 같은 시그니처의 확장함수가 있다면, 멤버함수의 우선순위가 더 높습니다. 그리고 자식타입의 확장함수 여부와 상관없이 변수의 현재 타입을 기준으로 확장함수가 호출됩니다.

확장함수 개념을 이용해서 확장 프로퍼티도 만들 수 있습니다. 
```kotlin
// 확장함수를 이용한 확장 프로퍼티
val String.lastChar: Char 
    get() = this[this.length - 1]
```

### infix
코틀린에선 중위함수라는 개념이 있습니다. for문에서 사용했었던 downTo, step 모두 IntProgression을 생성하는 중위함수 입니다. 
```kotlin
for (i in 10 downTo 2 step 2) {
    println(i)
}
```

```변수.함수이름(argument)``` 대신 ```변수 함수이름 argument``` 방식으로 함수를 호출합니다. 중위함수를 사용하기 위해서는 ```infix``` 키워드를 붙여주어야 함니다.  
```kotlin
public infix fun Int.downTo(to: Int): IntProgression {
    return IntProgression.fromClosedRange(this, to, -1)
}
```
### inline
inline 함수는 함수를 호출하는 쪽의 코드에 함수 본문을 그대로 삽입하도록 컴파일러에 지시하는 키워드입니다. 
주로 고차 함수에서 성능 최적화를 위해 사용됩니다. 

아래의 inline 함수가 있다고 가정하겠습니다. 
```kotlin
inline fun doSomething(block: () -> Unit) {
    block()
}

fun main() {
    doSomething { 
        println("hello")
    }
}
```

위의 inline 함수가 적용되면 실제로는 아래와 같이 컴파일됩니다. 즉 고차함수를 호출하는데 람다 객체를 생성하고, 객체를 함수에 전달하면서 발생하는 오버헤드를 줄일 수 있습니다.  
```kotlin
fun main() {
    println("hello")
}
```

## 코틀린에서 람다를 다루는 방법 

### 람다
코틀린에서 람다를 사용하는 방식은 아래와 같습니다. 
```kotlin
fun main() {
    isApple(Fruit("사과", 1000))
    isApple.invoke(Fruit("사과", 1000))
}

val isApple: (Fruit) -> Boolean = fun (fruit: Fruit): Boolean {
    return fruit.name == "사과"
}

// 함수의 타입 생략 가능
val isApple2 = fun (fruit: Fruit): Boolean {
    return fruit.name == "사과"
}

val isApple3 = { fruit: Fruit -> fruit.name == "사과" }
```

코틀린에선 함수를 값처럼 파라미터로 넘길 수 있습니다. 아래에서는 함수의 타입을 ```(Fruit) -> Boolean```으로 넘기고 있습니다. 
```kotlin
fun main() {
    val fruitList = listOf(Fruit("사과"))
    // 마지막 파라미터가 함수인 경우 소괄호 밖에서 람다 사용 가능 
    filterFruits(fruitList) { fruit ->
        println("사과만 받는다.")
        // 마지막 줄의 결과가 람다의 반환 값이다. 
        fruit.name == "사과"
    }
}

private fun filterFruits(fruits: List<Fruit>, filter: (Fruit) -> Boolean) {
    val results = mutableListOf<Fruit>()
    for (fruit in fruits) {
        if (filter.invoke(fruit)) {
            results.add(fruit)
        }
    }
    
    return results
} 
```
위에서 볼 수 있듯이, 마지막 파라미터가 함수인 경우에 소괄호 밖에서 람다를 사용할 수 있습니다. 그리고 람다의 마지막 줄에 ```fruit.name == "사과"```가 있는데 
마지막 라인의 결과가 람다의 결과가 됩니다. 

### Closure
자바에서는 람다에서 참조하고 있는 지역변수가 final 이거나 사실상 final 이어야 했습니다. 하지만 코틀린에서는 이런 제약이 없습니다. 그 이유는 코틀린에서는 람다가 시작하는 지점에 참조하고 있는 변수들을
모두 포획하여 그 정보를 가지고 있습니다. 

즉, 코틀린은 지역 변수의 참조를 람다에서 안전하게 쓰기 위해 IntRef, ObjectRef 같은 임시 객체에 변수 값을 감싸서 공유합니다. 
그래서 값이 바뀌어도, 같은 참조 객체의 값이 바뀌는 것이라 문제가 없습니다. 

예를 들면 코틀린에서 다음과 같은 코드가 있습니다.
```kotlin
fun example() {
    var x = 10

    val runnable = Runnable {
        x += 1
    }

    runnable.run()
}
```

이는 내부적으로 다음과 코드로 바뀝니다. 
```java
final Ref.IntRef x = new Ref.IntRef();
x.element = 10;

Runnable runnable = new Runnable() {
    public void run() {
        x.element += 1;
        System.out.println("x in lambda: " + x.element);
    }
};
```

기본 타입의 변수가, 참조 타입으로 감싸져서 공유되는 것을 확인할 수 있습니다. 이를 통해 코틀린의 람다는 지역 변수에 제약 없이 접근할 수 있게 되었습니다.

## 코틀린에서 컬렉션을 함수형으로 다루는 방법 

코틀린에선 컬렉션을 함수형 프로그래밍 방식으로 다루 수 있는 방법들을 제공합니다. 

```kotlin
fun main() {
    val fruitList = listOf(
        Fruit("사과", 1_000),
        Fruit("딸시", 1_000),
        Fruit("포도", 2_000),
        Fruit("배", 2_500),
    )
    
    // 필터 
    val apples1 = fruitList.filter {
        it.name == "사과"
    }
    
    // 필터에서 인덱스가 필요하다면
    val apples2 = fruitList.filterIndexed { idx, fruit -> 
        println(idx)
        fruit.name == "사과"
    }
    
    // 맵
    val applePrices1 = fruitList.filter {
        it.name == "사과"
    } .map { fruit -> fruit.currentPrice}

    // 맵에서 인덱스가 필요하다면
    val applePrices2 = fruitList.filter {
        it.name == "사과"
    } .mapIndexed { idx, fruit ->
        println(idx)
        fruit.currentPrice
    }

    // 맵의 결과가 null이 아닌 것만 가져오고 싶다면
    val applePrices3 = fruitList.filter {
        it.name == "사과"
    } .mapNotNull { idx, fruit ->
        fruit.nullOrValue()
    }
}
```

filter, filterIndexed, map, mapIndexed, mapNotNull 등 다양한 함수에 대해 알아보았습니다. 
코틀린에서는 필터, 맵 이외에도 다양한 컬렉션 처리 함수를 제공합니다. 

```kotlin
// all
val isAllApple = fruits.all { fruit -> fruit.name == "사과"}
// none
// any
// count
// sortedBy
// sortedByDescending
// distinctBy
// first
// firstOrNull
```