코틀린을 공부한 내용을 정리하는 포스팅입니다. 

## 코틀린은 모든 변수에 수정 가능 여부를 표시해 줘야 한다.
그리고 타입을 자동으로 컴파일러가 추론해 준다.

```kotlin
fun main() {
    // 바꿀 수 없는 변수
    var number1 = 10L
    number1 = 20L

    // 바꿀 수 없는 변수
    val number2 = 10L
    // 컴파일 에러 발생 
    // number2 = 20L
}
```

위의 예시에서 볼 수 있듯이 var이 붙은 변수는 변경 가능하지만, val이 붙은 변수는 변경 불가능합니다. 그리고 추가적으로 자바는 항상 타입을 명시해주어야 했지만, 코틀린은 타입을 붙이지 않아도 컴파일러에 의해 추론된다. 덕분에 더 간결한 코드가 가능해졌습니다. 

## 코틀린은 primitive 타입과, reference 타입을 분리하지 않는다. 

자바에서는 int, long, float, double 등 기본타입과 Int, Long, Float, Double 등 참조타입이 명확하게 분리되어 있었습니다. 하지만 코틀린에서는 참조타입으로 타입을 지정해도 내부적으로 기본타입으로 처리됩니다. 

```
// 코틀린
var num: Long = 10L
// 자바
long num = 10L;
```
                                                                                                                                       
코틀린을 자바로 바꾸면 위 코드 처럼 기본타입으로 바뀌는 것을 확인할 수 있습니다.

### 코틀린에서 null을 다루는 방법

코틀린에선 null이 가능한 타입과, 불가능한 타입이 나뉩니다. null이 가능한 타입은 타입 뒤에 ?를 붙여 null 가능함을 표시합니다. 만약 null이 가능한 타입임에도 null 체크를 하지 않으면 컴파일 에러가 발생합니다. 코틀린은 이처럼 null에 대한 위험성을 줄여줍니다. 

```kotlin
fun startsWithA1(str: String?): Boolean? {
	// 컴파일 에러 발생!
    // return str.startsWith("A")
    
    // safe call 
    return str?.startsWith("A")
  }
```

그리고 null이 가능한 타입만을 위한 기능을 제공합니다.

### Safe Call

Safe Call은 변수가 null이 아니면 실행하고, null이면 실행하지 않고 그대로 null이 됩니다.

```kotlin
val str: String? = "ABC"
// null이 가능하므로 불가능
//str.length 
// safe call을 이용하여 가능
str?.length
```

위 예시에서 str 변수가 만약 null을 참조했다면, str?.length는 그 자체가 null이 됩니다. 

### Elvis 연산자

Elvis 연산자는 앞의 결과가 null이면 뒤의 코드를 실행합니다. 

```kotlin
val str: String? = "ABC"
str?.length ?: 0
```

위의 예시에서 str이 null을 참조한다면 어떻게 될까요? Safe Call에 의해 str?.length는 null이 되고, Elvis 연산자에 의해 뒤의 값이 이용되면서 결과는 0이 될 것입니다. 

만약 변수가 null이 아님을 단언하려면, !! 표시를 합니다. 주의해야 할 점은 만약 null이 들어온다면 런타임에 NPE 에러가 발생할 수 있습니다.

```kotlin
fun startsWithA1(str: String?): Boolean {
	return str!!.startsWith("A")
}
```

## 코틀린에서 Type을 다루는 방법 

코틀린에서 타입을 다루는 방식은 자바의 방식과 다소 차이가 있습니다.

### 기본 타입

자바에서는 int 타입에서 long 타입으로 암시적 변경이 가능했습니다.

```java
int number1 = 4;
long number2 = number1;
```

하지만 코틀린에선 암시적 변경이 불가합니다. 아래와 같이 명시적으로 타입을 변경해주어야 합니다. 아래 처럼 다른 기본 타입으로 변경할때, to변환타입() 메서드를 사용해야 합니다. 물론 타입이 nullable인 경우는 적절한 처리가 필요합니다.

```kotlin
// 타입 추론이 가능하므로 타입 생략 
val number1 = 4;
// 명시적 타입 변경 
val number2: Long = number1.toLong()

val number3 = 3
val number4 = 5
val result = number3 / number4.toDouble()
```

### 타입 캐스팅
코틀린에선 기본 타입이 아닌 일반 타입은 어떻게 변경할까요? 자바에선 아래와 같이 변경하였죠

```java
public static void printAgeIfPerson(Object obj) {
	if (obj instanceof Person) {
    	// 타입 캐스팅  
    	Person person = (Person) obj;
        System.out.println(person.getAge());
    }
}
```

코틀린에선 아래와 같이 타입을 변경합니다. 

```kotlin
fun printAgeIfPerson(obj: Any) {
	// instanceof 대신 is 사용 
	if (obj is Person) {
    	// 타입 캐스팅 
    	val person = obj as Person
        println(person.age)
    }
}
```

그리고 스마트 캐스팅을 이용해서 위 코드를 더 간결하게 만들 수도 있습니다. 명시적으로 타입 캐스팅을 하지 않았음에도 컴파일러에서 인지를 하고 자동적으로 캐스팅이 됩니다. 

```kotlin
fun printAgeIfPerson(obj: Any) {
	// instanceof 대신 is 사용 
	if (obj is Person) {
    	// 스마트 캐스팅 
        println(obj.age)
    }
}
```

### 코틀린의 특이한 타입 3가지

Any

-   Java의 Object 역할 (모든 객체의 최상위 타입)
-   모든 primitive type의 최상위 타입도 Any이다. 
-   Any 자체로는 null을 포함할 수 없어 null을 포함하고 싶다면, Any?로 표현 
-   Any에 equals / hashCode / toString 존재 

Unit

-   Unit은 Java의 void와 같은 역할
-   void와 다르게 Unit은 자체로 타입 인자로 사용 가능하다. 
-   함수형 프로그래밍에서 Unit은 단 하나의 인스턴스만 갖는 타입을 의미. 즉 코틀린의 Unit은 실제 존재하는 타입이라는 것을 표현 

```kotlin
fun saveData(): Result<Unit> {
    return try {
        println("데이터 저장 성공")
        // Void와는 달리 Unit을 타입 인자로 사용하였다.
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

Nothing

-   Nothing은 함수가 정상적으로 끝나지 않았다는 사실을 표현한다. 
-   무조건 예외를 반환하는 함수 / 무한 루프 함수 등 

### String interpolation

코틀린에선 문자에 들어가는 변수를 아래와 같이 사용할 수 있습니다. ${변수}를 사용하면 값이 들어갑니다. 

```kotlin
val person = Person("name1", 100)
val log = "사람의 이름은 ${person.name}이고 나이는 ${person.age}세 입니다"
```

$변수로 문자열을 변경할 수도 있습니다.

```kotlin
val name = "name"
val age = 100
val log = "사람의 이름: $name 나이: $age"
```

이상 코틀린의 기본 문법에 대해 정리해 보았습니다. 다음 포스팅에서도 계속해서 코틀린의 기본 문법에 대해 다루어 보겠습니다.