팀원들에게 테스트 코드에 있어 발생할 수 있는 안티 패턴에 대해 공유를 한 적이 있습니다. 그래서 이번 글에서는 테스트 코드 작성에 있어서 발생할 수 있는 안티 패턴에 대해 이야기 해보고자 합니다. 

## 비공개 메서드에 단위 테스트 

비공개 메서드는 클라이언트에서 식별할 수 없는 동작입니다. 즉, 내부 구현을 위한 동작으로 클라이언트에서는 알지도 못하고, 관심도 없는 동작입니다. 
이는 캡슐화의 목적과 관련이 있습니다. 캡슐화의 목적이 클라이언트에는 필요한 동작만 노출하고 내부적으로 리펙토링의 위험도를 낮추는 것이기 때문입니다. 

그런데 만약 테스트 코드에서 이렇게 리펙토링을 쉽게 하려고 숨겨놓은 구현 세부 사항을 굳이 꺼내서 테스트를 한다면 테스트 코드의 리펙토링 내성은 낮아질 수 밖에 없습니다. 
보통 리펙토링에 의해서 비공개 메서드의 동작이 달라질 수도 있고, 심지어 비공개 메서드가 사라질 수도 있습니다. 따라서 제품 코드는 정상 동작하는데 테스트 코드는 실패가 발생하는 문제가 생깁니다. 
이는 테스트의 본질적인 목적인 지속가능한 개발을 저해하는 심각한 요소입니다. 

### 해결방법은?
저 또한 비공개 메서드에 때문에 테스트하기 어려웠던 적이 많습니다. 비공개 메서드가 너무 커서 충분한 커버리지를 얻지 못하였기 때문입니다. 하지만 이것은 테스트 코드가 주는 설계가 잘못되었다는 신호일 수도 있습니다. 
즉 비공개 메서드가 크다는 것은 별도의 클래스로 도출해야 하는 추상화가 누락되었다는 신호일 수 있습니다. 

아래와 같은 코드가 있다고 가정해봅시다. 
```java
public class Player {
	
	private int score;
	
	public Player (int score) {
		this.score = score;
    }
	
	public void addPoint(int point) {
		score += calculatePoint(point); 
    }
	
	private int calculatePoint(int point) {
		final int multiplier = /*배수를 구하기 위한 복잡한 계산*/
		final int min = /*최솟값을 위한 복잡한 계산*/
        final int max = /*최댓값을 위한 복잡한 계산*/
		
		return Math.min(Math.max(point * multiplier, min), max);
    }
}
```
공개 메서드는 point를 더해주는 단순한 동작입니다. 하지만 비공개 메서드인 ```calculatePoint```는 훨씬 더 복잡한 로직을 갖고 있습니다.
그럼에도 private로 해당 메서드를 노출하지 않기 때문에 테스트를 할 수 없습니다. 만약 private 메서드가 크다면 메서드가 가진 역할이 방대하다는 것을 의미할 수 있습니다. 
따라서 클래스 분리를 통해서 위의 문제를 해결할 수 있습니다. 아래는 개선된 코드입니다. 

```java
public class Player {
	
	private int score;
	private final ScoreCalculator calculator;
	
	public Player (int score, ScoreCalculator calculator) {
		this.score = score;
	    this.calculator = calculator;	
    }
	
	public void addPoint(int point) {
		score += calculatePoint(point); 
    }
}

public class ScoreCalculator {

	private int calculatePoint(int point) {
		final int multiplier = /*배수를 구하기 위한 복잡한 계산*/
		final int min = /*최솟값을 위한 복잡한 계산*/
		final int max = /*최댓값을 위한 복잡한 계산*/

		return Math.min(Math.max(point * multiplier, min), max);
	}
}
```
```calculatePoint``` 메서드를 분리하였기 때문에 이젠 ```calculatePoint```를 테스트할 수 있습니다. 뿐만 아니라 클래스를 분리하였기 때문에 두 개의 클래스는 역할 분리를 통해 하나의 역할만을 갖는 
코드로 변경되었습니다. 테스트 코드는 이처럼 설계의 문제점을 지적해 주고, 더 좋은 방향으로 나아가도록 도움을 줍니다.

## 테스트에 도메인 지식 유출 

테스트 코드를 작성하면서 흔히 저지르는 실수가 도메인의 로직을 테스트에 그대로 노출시키는 것입니다. 테스트 코드는 도메인 로직을 테스트하려고 작성하는 것인데, 검증 대상이 테스트 코드에 등장한다면 이것은 확실한 안티 패턴입니다. 
아래와 같은 코드가 있습니다. 
```java
public static class Calculator {
	public static int add(int value1, int value2) {
		return value1 + value2;
    }
}
```
그리고 이를 테스트하는 코드입니다. 
```java
@Test
public void 두개의_숫자를_더한다() {
	// given
    int value1 = 1;
	int value2 = 3;
	int expected = value1 + value2;
	
	// when
	int actual = Calculator.add(value1, value2);

	// then
	Assertions.assertEquals(actual, expected);
}
```
준비 부분에서 사실상 제품 코드의 로직을 그대로 사용하고 있습니다. 이러한 테스트는 또다른 구현 세부 사항과 결합되는 유형입니다.
만약 제품 코드가 변경된다면 어떻게 될까요? 이런 유형의 테스트는 아래와 같은 문제점이 있습니다. 
- 도메인 로직이 변경될 경우, 테스트 코드도 같이 변경되어야 합니다. 
- 테스트 코드와 제품 코드의 결합으로 인해 리펙토링 내성이 안좋아 지고, 거짓 양성을 구분할 수 없게 됩니다. 

### 해결 방법 

이런 경우 대부분 도메인 로직을 테스트에서 지우고, 하드 코딩으로 테스트 하면 도메인 로직과 제품 코드간의 결합도가 낮아집니다. 그리고 자연스럽게 리펙토링 내성도 높아집니다.
```java
@ParameterizedTest
@CsvSource({
		"1, 2, 3",
		"100, 1, 101",
		"5, 5, 10",
		"10, 3, 13"})
@Test
public void 두개의_숫자를_더한다(int value1, int value2, int expected) {
	// when
	int actual = Calculator.add(value1, value2);

	// then
	Assertions.assertEquals(actual, expected);
}
```

## 코드 오염 

다음 안티 패턴은 코드 오염입니다. 이는 테스트 코드에서만 필요한 코드가 제품 코드에 있는 경우입니다. 
코드 오염의 문제는 테스트 코드와 제품 코드가 혼재돼 유지비가 증가하는 것입니다. 

아래의 코드를 보겠습니다.

```java
public class SheetDataService {
    ...
	public String getSheetData() {
		if (!isTest) {
			restTemplate.getForEntity(String.format(SHEET_API_URL, sheetId, tabName), String.class);
		}
	}
}
```
위의 코드는 테스트 코드에서 restTemplate으로 요청 하는 것을 막기 위해서 ```isTest``` 라는 플레그를 통해 로직을 체크합니다. 
이렇게 하면 코드의 가독성이 떨어지고, 유지비가 증가합니다. 

### 해결 방법 

이러한 상황을 막기 위해서 인터페이스를 사용합니다. 
```java
public interface SheetDataHelper {
	public String getSheetData();
}

public class SheetDataService implements SheetDataHelper {
    ...
	public String getSheetData() {
		if (!isTest) {
			restTemplate.getForEntity(String.format(SHEET_API_URL, sheetId, tabName), String.class);
		}
	}
}

public class FakeSheetDataService implements SheetDataHelper {
    ...
	public String getSheetData() {
		// 아무것도 하지 않음 
	}
}
```
하나는 제품 코드에서 사용하는 구현체, 하나는 테스트 코드에서 사용하는 구현체를 만들어서 코드를 분리할 수 있습니다. 이렇게 하면 더 이상 다른 환경을 설명할 필요가 없으므로 단순하게 할 수 있습니다. 

## 시간 처리하기 

마지막 안티 패턴은 코드 내에서 직접 시간을 처리하는 부분입니다. 많은 어플리케이션 기능에서는 현재 날짜와 시간에 대한 접근이 필요합니다. 하지만 시간에 따라 달라지는 기능을 테스트하면 거짓 양성이 발생할 수 있습니다. 

아래 코드를 보겠습니다. 
```java
class Schedule {

	private final LocalDateTime start;
	private final LocalDateTime end;
        ...

	public boolean isProgress() {
		LocalDateTime now = LocalDateTime.now();
		return now.isAfter(start) && now.isBefore(end);
	}
}
```

위의 코드를 보면 ```isProgress``` 메서드 내부에서 현재 시간을 구하기 위해 ```LocalDateTime.now()``` 메서드를 사용하고 있습니다. 
하지만 ```LocalDateTime.now()``` 는 테스트 코드가 수행될 때마다 다른 결과를 발생시킵니다. 테스트 코드가 시간에 따라 성공하기도, 실패하기도 한다면 이는 
테스트 코드의 리펙토링 내성이 좋지 않다는 증거입니다. 
```java
@Test
public void scheduleTest(int value1, int value2, int expected) {
	// given
	LocalDateTime start = LocalDateTime.of(2023, 10, 4, 12, 0, 0);
	LocalDateTime end = LocalDateTime.of(2023, 10, 14, 12, 0, 0);
	
    Schedule schedule = new Schedule(start, end);   
	
	// when
	boolean isProgress = schedule.isProgress();

	// then
	Assertions.assertEquals(isProgress, true); // 시간에 따라 테스트의 결과가 달라진다. 
}
```

### 해결 방법 
시간에 따라 테스트의 결과가 달라지는 문제를 해결하기 위해서는 외부에서 시간을 명시적으로 주입 받도록 해야 합니다. 아래는 개선된 코드입니다.

```java
@Test
public void scheduleTest(int value1, int value2, int expected) {
	// given
	LocalDateTime now = LocalDateTime.of(2023, 10, 11, 12, 0, 0);
	LocalDateTime start = LocalDateTime.of(2023, 10, 4, 12, 0, 0);
	LocalDateTime end = LocalDateTime.of(2023, 10, 14, 12, 0, 0);
	
    Schedule schedule = new Schedule(start, end);  
	
	// when
    boolean isProgress = schedule.isProgress(now);

	// then
	Assertions.assertEquals(actual, expected);
}
```

이젠 테스트 코드를 아무리 실행해도 결과값이 달라지는 일이 없어졌습니다. 이에 따라 거짓 양성을 보여주는 일도 없어졌기 때문에, 리펙토링 내성도 자연스럽게 개선되었습니다. 


이상 테스트 코드의 안티 패턴에 대해 알아보았습니다. 
이후로도 테스트 코드 작성시 발생할 수 있는 문제점에 대한 개선점들을 포스팅할 예정입니다. 
긴 글 읽어주셔서 감사합니다. 


