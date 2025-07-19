저번 글에 이어서 코틀린의 기본 내용을 정리하겠습니다. 


## 코틀린에서 조건문을 다루는 방법 

### if문 
코틀린과 자바는 if문이 거의 비슷합니다. 코틀린이나 자바 모두 동일한 if 문을 갖습니다. 

- 코틀린  
```kotlin
fun getPassOrFail(score: Int): String {
   if (score >= 50) {
       return "P"
   } else {
       return "F"
   }
}
```

- 자바 
```java
private String getPassOrFail(int score) {
    if (score >= 50) {
        return "P";
    } else {
        return "F";
    }
}
```

하지만 코틀린과 자바가 다른 부분이 있습니다.
**Java에서 if-else는 Statement이지만, Kotlin에서는 Expression입니다.**

- Statement: 프로그램의 문장, 하나의 값으로 도출되지 않는다. 
- Expression: 하나의 값으로 도출되는 문장 

자바에서는 if문은 항상 Statement 였습니다. 따라서 특정 변수에 할당하는 것이 불가능했습니다.
하지만, 코틀린에서는 if문을 Expression으로 보기 때문에 아래와 같은 코드가 가능합니다.   
```kotlin
// if문이 하나의 값으로 도출 가능하기 때문에 바로 return 가능합니다.
fun getPassOrFail(score: Int): String {
    return if (score >= 50) {
        "P"
    } else {
        "F"
    }
}
```
코틀린에선 삼항 연산자가 없습니다. if문을 삼항 연산자 처럼 사용할 수 있기 때문입니다.


### switch case 
자바에선 swich case 문을 아래와 같이 사용했습니다. 
```java
private String getGradeWithScore(int score) {
    switch (score / 10) {
        case 9:
            return "A";
        case 9:
            return "B";
        case 9:
            return "C";
        default:
            return "D";
    }
}
```
코틀린에선 아래와 같은 분기 처리를 지원합니다. 자바와 달리 코틀린에선 when이라는 키워드를 이용해서 분기 처리를 지원합니다.  
```kotlin
fun getGradeWithSwitch(
    score: Int
): String {
    return when (score / 10) {
        9 -> "A"
        8 -> "B"
        7 -> "C"
        else -> "D"
    }
}
```
그리고 특이한 점은 조건부에 어떠한 expression이든 들어갈 수 있습니다. 
아래 예제를 보면 in 구문을 Expression으로 이용하고 있습니다. 이를 통해 좀 더 섬세한 분기가 가능합니다.   
```kotlin
fun getGradeWithSwitch(
    score: Int
): String {
    return when (score / 10) {
        in 90..99 -> "A"
        in 80..89 -> "B"
        in 70..79 -> "C"
        else -> "D"
    }
}
```
when 구문에 값이 없을 수도 있습니다. 그러면 early return 처럼 동작합니다. when 구문의 조건에 Expression이 올 수 있기 때문에 다양한 처리가 가능합니다.

```kotlin
fun judgeNumber3(score: Int, subjects: String) {
    when {
        score >= 90 && subjects == "수학" -> println("수학 통과입니다.")
        score >= 85 && subjects == "영어" -> println("영어 통과입니다.")
        else -> println("불합격입니다.")
    }
}
```

위의 예시들을 살펴보면 코틀린의 when은 java의 switch-case에 비해 더 강력한 기능을 갖고 있음을 알 수 있습니다.


## 코틀린에서 반복문을 다루는 방법 

코틀린에서 반복을 하는 방법을 알아보겠습니다. 
```kotlin
val numbers = listOf(1L, 2L, 3L) // (1)
for (number in numbers) { // (2)
    println(number)
}

for (i in 1..3) { // (3)
    println(i)
}

for (i in 3 downTo 1) { // (4)
    println(i)
}

for (i in 1..10 step 2) { // (5)
    println(i)
}
```

(1): 코틀린에선 컬렉션을 만드는 방식입니다.
(2): for-each 반복 대신 in을 사용합니다. Java와 동일하게 Iterable이 구현된 타입이라면 모두 들어갈 수 있습니다.
(3): 1 ~ 3 까지 범위를 반복하기 위한 구문입니다. 
(4): 3 ~ 1 까지 숫자가 1씩 작아지는 반복을 합니다. 
(5): 1 ~ 10 까지 2씩 커지는 반복을 합니다.  


위의 예시에서 downTo, step 키워드가 나옵니다. 확인해보면 downTo, step 모두 함수입니다. 
모두 등차수열을 만든는 목적으로 사용됩니다. 기본적으로 ```변수.함수이름(argument)``` 이런 호출 순서를 가지지만, downTo, step은 ```변수 함수이름 argument``` 이런 호출 순서를 가집니다. 이러한 함수를 중위 호출 함수라 합니다. 
```kotlin
// downTo
public infix fun Int.downTo(to: Int): IntProgression {
    return IntProgression.fromClosedRange(this, to, -1)
}

// step 
// IntProgression에 대한 확장 함수 
public infix fun IntProgression.step(step: Int): IntProgression {
    checkStepIsPositive(step > 0, step)
    return IntProgression.fromClosedRange(first, last, if (this.step > 0) step else -step)
}
```

## 코틀린에서 예외를 다루는 방법 

자바와 코틀린의 예외처리 방식은 동일합니다. 

자바의 예외처리 방식 
```java
private Integer parseIntOrThrow(String str) {
    try {
        return Integer.parseInt(str);
    } catch (NumberFormatException e) {
        return null;
    }
}
```

코틀린의 예외처리 방식 
```kotlin
fun parseIntOrThrowV1(str: String): Int {
    try {
        return str.toInt()
    } catch (e: NumberFormatException) {
        throw IllegalArgumentException("주어진 ${str}는 숫자가 아닙니다.")
    }
}

fun parseIntOrThrowV2(str: String): Int? {
    return try {
        str.toInt()
    } catch (e: NumberFormatException) {
        null
    }
}
```

기본적으로 자바와 코틀린의 예외처리 방식은 같습니다. 다만 코틀린에선 try-catch 구문을 expression으로 보기 때문에 ```parseIntOrThrowV2``` 처럼 try-catch를 바로 반환할 수 있습니다.
그리고 추가적으로 kotlin은 Checked Exception과 UnChecked Exception을 구분하지 않습니다. 따라서 throws를 명시적으로 적어줘야 하는 경우가 사라졌습니다. 

### try with resources
자바에선 아래와 같이 try with resources 구분을 이용해서 자원을 회수 했습니다.

```java
public void readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        System.out.println(reader.readLine());
    }
}
```

하지만 코틀린에선 try with resources 구문이 없습니다. 대신 use 라는 inline 확장함수를 사용합니다.

```kotlin
fun readFile(path: String) {
    BufferedReader(FileReader(path)).use { reader ->
        println(reader.readLine())
    }
}
```

## 코틀린에서 함수를 다루는 방법

코틀린에선 함수가 하나의 값을 반환하면 Expression으로 사용 가능합니다. 따라서 아래와 같이 사용할 수 있습니다.
```{}```(중괄호) 없이 간결한 표현이 가능합니다. 
```kotlin
fun max(a: Int, b: Int): Int = 
    if (a > b) {
        a
    } else {
        b
    }
```

한 줄로도 표현 가능합니다. 블록을 사용하는 경우가 아니라면, 반환 타입도 생략 가능합니다.
```kotlin
fun max(a: Int, b: Int) = if (a > b) a else b
```

### default 파라미터

코틀린에선 함수 파라미터의 기본값을 지정할 수 있습니다. 
많은 클라이언트 코드에서 ```useNewLine``` 파라미터를 ```true```로 사용한다면 아래와 같이 기본 값을 적용할 수 있습니다.
```kotlin
fun repeat(
    str: String,
    num: Int = 3,
    useNewLine: Boolean = true
) {
    for (i in 1..num) {
        if (useNewLine)
            println(str)
        else 
            print(str)
    }
}
```

만약 위의 메서드에서 ```num```은 3으로 그대로 쓰고 싶고, ```useNewLine```은 false로 사용하고 싶다면 아래와 같이 named argument로 파라미터를 지정할 수 있습니다.
그러면 num은 3으로 기본값을 사용하고, ```useNewLine```값만 false로 값이 지정됩니다. 직접적으로 원하는 파라미터에 값을 지정할 수 있기 때문에 빌더의 장점도 생깁니다.
(하지만, 코틀린에서 자바 함수를 가져와 사용할 때, name arguemnt를 사용할 수 없습니다.) 
```kotlin
repeat("Hello World", useNewLine = false)
```

### 가변인자 

코틀린에선 가변인자를 다음과 같이 사용합니다. 메서드 파라미터 앞에 vararg를 적어주어야 합니다.  
```kotlin
fun printAll(vararg strings: String) {
    for (str in strings) {
        println(str)
    }
}
```
사용은 아래와 같이 합니다. 
```kotlin
val array = arrayOf("A", "B", "C")
// 배열을 사용할 때는 스프레드 연산자(*)를 앞에 붙여준다. 
printAll(*array)
printAll("A", "B", "C")
```

알아본 내용은 다음과 같습니다. 
- 메서드 body가 하나의 값을 반환하는 경우는 하나의 값(Expression)으로 간주됩니다. 따라서 block을 없앨 수 있고, 반환 타입도 없앨 수 있습니다. 
- 함수 파라미터에 기본값을 설정할 수 있습니다. 
- 함수를 호출할 때 특정 파라미터를 지정해 넣어줄 수 있습니다. 
- 가변인자에는 vararg 키워드를 사용하며, 배열과 호출할 때에는 스프레드 연산자(*)를 넣어줘야 합니다. 
