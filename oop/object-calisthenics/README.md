우아한형제들에서 코드 리뷰를 객체지향 생활 체조 원칙을 기반으로 한다는 이야기를 들었습니다. 

그래서 객체지향에 대한 명확하고, 정량적인 방법론을 알고 싶었던 저는 객체지향 생활 체조 원칙에 대해 관심이 생겼습니다. 
찾아본 결과 많은 블로그, 강연에서 객체지향 생활 체조 원칙에 대해 이야기하고 있었습니다. 

그래서 이번 포스팅에서는 객체지향 생활 체조 원칙에 대해 작성해보겠습니다. 

## 객체지향 생활 체조 원칙 

객체지향 생활 체조 원칙은 소트윅스 앤솔러지(ThoughtWorks Anthology) 라는 책에 나오는 원칙입니다. 

소트윅스 앤솔로지에서는 9가지 원칙을 준수하면 객체지향을 추구할 수 있다고 이야기합니다. 

1. 한 메서드에 오직 한 단계의 들여쓰기(indent)만 한다. 
2. else 예약어를 쓰지 않는다. 
3. 모든 원시 값과 문자열을 포장한다. 
4. 일급 컬렉션을 사용한다. 
5. 한 줄에 점을 하나만 찍는다. 
6. 줄여 쓰지 않는다. 
7. 모든 엔티티를 작게 유지한다. 
8. 3개 이상의 인스턴스 변수를 가진 클래스를 쓰지 않는다. 
9. getter/setter/프로퍼티를 쓰지 않는다. 

## 1. 한 메서드에 오직 한 단계의 들여쓰기만 한다. 

1개 메서드 안에서 if/for/while 등 2 depth 이상 사용하지 않아야 합니다. 
아래와 같은 메서드가 있다고 가정하겠습니다. 메서드를 보시면 for문에서 indent가 추가되면서 depth가 2가 됩니다. 
이렇게 depth가 추가되면 가독성이 안좋아집니다. 

```java
public class StringCalculator {
	public static int splitAndSum(String text) {
		int result = 0;
		if (text == null || text.isEmpty()) {
			result = 0;
		} else {
			String[] values = text.split(",|:");
			for (String value : values) {
				result += Integer.parseInt(value);
			}
		}
		return result;
    }
}
```

위의 코드를 아래의 코드로 수정해보았습니다. 

```java
public class StringCalculator {
	public static int splitAndSum(String text) {
		int result = 0;
		if (text == null || text.isEmpty()) {
			result = 0;
		} else {
			String[] values = text.split(",|:");
			result = sum(values);
		}
		return result;
	}
	
	private static int sum(String[] values) {
		int result = 0;
		for (String value : values) {
			result += Integer.parseInt(value);
		}
		return result;
    }
}
```

함수는 한가지 역할만을 해야 합니다. 만약 함수의 indent가 크다면 함수가 많은 역할을 하고 있다는 의미일 수 있습니다.  
따라서 위의 함수의 예시 처럼 메서드를 역할에 따라 분리하면 자연스럽게 indent도 줄어드는 것을 볼 수 있습니다. 

## 2. else 예약어를 사용하지 않는다. 

else가 있는 코드는 의도를 파악하기가 어렵습니다. 이 경우 **early exit pattern**을 이용하여 의도를 분명히 나타낼 수 있습니다. 

아래의 코드에서 else를 사용함으로써, 함수의 의도를 파악하기가 더 어려워졌습니다. 
```java
public static int splitAndSum(String text) {
    int result = 0;
    if (text == null || text.isEmpty()) {
        result = 0;
    } else {
        String[] values = text.split(",|:");
        result = sum(values);
    }
    return result;
}
```

이 메서드에서 else를 없애고 **early exit pattern**을 적용해보겠습니다. 

```java
public static int splitAndSum(String text) {
    if (text == null || text.isEmpty()) {
        return 0;
    }
    String[] values = text.split(",|:");
	return sum(values);
}
```

else를 제거하고 결과를 바로 return 하도록 코드를 수정하였습니다. 결과적으로 indent가 1줄어들었고, 코드 가독성도 더 좋아졌습니다. 

## 3. 모든 원시 값과 문자열을 포장한다. 

기본 타입을 클래스로 한번 감싸면 표현력이 더 좋아지고, 내부적으로 값의 유효성을 체크하기에 용이합니다.

num이 음수이면 예외를 발생시켜야 한다고 가정해보겠습니다. 아래 코드에서는 기본 타입으로 값을 표현하고 있기 때문에 표현에 한계가 있고, 양수를 체크하는 로직이 바깥에 노출되어 있습니다.   

```java
private static int sum(int[] numArray) {
    int result = 0;
	for (int i = 0; i < numArray.length; i++) {
	    int num = numArray[i];
		if (num < 0) {
			throw new RuntimeException();
        }
		result += num;
    }
	return result;
}
```

기본 타입을 클래스로 감싸보겠습니다. 

```java
public class Positive {
	private int num;
	
	public Positive(int num) {
		if (num < 0) {
			throw new RuntimeException();
		}
		this.num = num;
    } 
	
	public Positive add(Positive other) {
		return new Positive(this.num + other.num); 
    }
	
	public int getNumber() {
		return number;
    }
}

private static int sum(Positive[] nums) {
	Positive result = new Positive(0);
	for (Positive num : nums) {
		result = result.add(num);
	}
	return result.getNumber();
}
```

```Positive``` 클래스로 기본 타입을 감싸주니 훨씬 더 표현력이 좋아졌습니다. 클래스명(Positive)이 양수 값임을 표현하고 있기 때문입니다.
그리고 생성자에서 만약 음수 값이 들어오면 RuntimeException을 발생시킵니다.

추가적으로 클래스로 감싸고 클래스 내부에 ```add``` 처럼 필요한 도메인 로직을 담을 수 있습니다. 이는 코드의 도메인 로직의 응집도를 높여줍니다.   

## 4. 일급 컬렉션을 이용한다. 

일급 컬렉션이란 **Collection을 Wrapping 하면서 Collection 외 다른 멤버 변수가 없는 상태**를 말합니다. 

```java
public class Store {
	private Set<Brand> brands;
	
	public Store(List<Brand> brands) {
		
    }
	
	private void validSize(List<Brand> brands) {
		if (brands.size() >= 10) {
			throw new IllegalArgumentException("브랜드는 10개 초과로 입점할 수 없습니다.");
		}
    }
}
```
일급 컬렉션은 Collection을 내부에 감싸고, Collection과 관련된 도메인 로직을 추가할 수 있습니다. 이로써 비지니스 로직을 검증하는 책임을 컬렉션을 사용하는 쪽에서 하는 것이 아니라, 일급 컬렉션 내부에서 검증할 수 있게 됩니다.

그리고 일급 컬렉션을 사용하면 추가적으로 클라이언트에 불필요한 메서드가 노출되는 가시성 문제도 해결됩니다. 
만약 Map을 사용했다면 클라이언트에 그대로 remove, removeAll과 같은 메서드가 노출됩니다. 일급 컬렉션으로 감싸면 불필요하게 메서드가 노출됨으로써 발생하는 메서드의 오용을 막을 수 있습니다.
- 이는 클린 코드 8장에서 이야기하는 '경계 인터페이스를 노출하지 말라'는 이야기와 통합니다. 

## 5. 한 줄에 점을 하나만 찍는다. 

점은 **멤버 변수에 점근함**을 의미합니다.

```java
if (user.getMoney().getValue() > 100_000L) {
	throw new IllegalArgumentException("소지금은 100_000원을 초과할 수 없습니다.");
}
```

해당 코드에서는 User, Money 두 객체에 의존하고 있습니다. 
User 객체에서 Money 객체를 가져와서 Value를 물어보고 있는 형태입니다. 

이는 디미터 법칙을 어기는 코드입니다. 디미터 법칙에 따르면 모든 객체는 자신과 가까운 객체의 메서드만 호출해야 합니다. 이를 어기면 클아이언트 객체는 너무 많은 객체에 대한 의존성을 갖게 됩니다. 
객체간 결합도가 생기고 캡슐화도 깨집니다. 

```java
if (user.hasMoney(100_000L)) {
	throw new IllegalArgumentException("소지금은 100_000원을 초과할 수 없습니다.");
}
```

그래서 위와 같이 데이터를 갖고 있는 객체에 단지 물어보면 됩니다. 코드를 에선 ```hasMoney```로 user에게 물어보고 있습니다.


## 6. 줄여쓰지 않는다. 

프로그래밍에서 중요한 원칙중 하나는 **명확함이 간결함보다 중요하다는 것**입니다.
따라서 최대한 축약을 지양해야 합니다.

만약 메서드의 이름이 길다면 **책임을 너무 많이 갖고 있어서** 일 수 있습니다.

```java
public void checkValidityAndSave() {
    // 유효성 검사 
    // 저장     
}
```

위 코드는 2개의 책임을 갖고 있는데, 아래와 같이 각각 메서드를 나누어주어야 합니다. 

```java
public boolean checkValidity() {
	// 유효성 검사
}

public void save() {
    // 저장 	
}
```

그리고 이름을 지나치게 축약하는 경우, 다른 사람이 알아보기 어려울 수 있기 때문에 네이밍을 줄여쓰는 것은 최대한 지양해야 합니다. (예: ViewController -> vc)

## 7. 모든 엔티티를 작게 유지한다. 

50줄이 넘는 클래스와, 파일이 10개 이상인 패키지를 지양하자는 원칙입니다. 

보통 클래스의 줄수가 50줄이 넘어가면 클래스가 여러가지 책임을 갖게됨을 의미합니다. 이는 코드의 이해와 재사용을 어렵게 만듭니다. 
패키지에도 파일 수가 10개 이상이 된다면 패키지에 의해 목적에 따라 파일이 쪼개져 있지 않음을 의미할 수 있습니다. 패키지의 파일 수를 줄여야 패키지가 하나의 목적을 달성하기 위한 클래스의 집함임을 명확하게 알 수 있습니다.

만약 헥사고날 아키텍처를 적용한 프로젝트라면, 아웃바운드 어댑터의 패키지에는 많은 클래스 파일이 들어갈 수 있습니다. 
만약 영속성 어댑터 패키지라면 어댑터 클래스는 ```package-private``` 접근 제한자를 사용하므로 JPA 엔티티나 Mapper, Adapter 등이 한 패키지에 있어야 합니다. 그래서 한 패키지 안에 여러 클래스가 존재할 수 있습니다. 
영속성 어댑터 패키지 안에 클래스가 너무 많아진다면 바운디드 컨텍스트를 적절히 분리했는지 고려해볼 필요가 있습니다.

## 8. 3개 이상의 인스턴스 변수를 가진 클래스를 쓰지 않는다. 

인스턴스 변수가 많아질수록 클래스의 응집도는 낮아집니다. 이 원칙은 객체지향 생활 체조 원칙 중에 가장 여려운 원칙이라고 생각합니다. 
하지만 클래스의 캡슐화를 높혀주기 위해 필요한 원칙입니다.

```java
public class Customer {
	private int customerId;
	private String firstName;
	private String lastName;
	
	public Customer(int customerId, String firstName, String lastName) {
		this.customerId = customerId;
		this.firstName = firstName;
		this.lastName = lastName;
    }
}
```
위의 클래스에서 ```firstName```과 ```lastName```은 이름이라는 범주에 포함될 수 있는 정보입니다. 하지만 위의 Customer 클래스에선 하나의 범주로 포함하고 있지 않습니다. 
위의 클래스를 더욱 응집도가 높고, 캡슐화된 상태로 콴리하기 위해서 ```firstName```과 ```lastName```를 이름이라는 범주로 묶어보겠습니다. 

```java
public class Customer {
	private int customerId;
	private Name name;
	
	public Customer(int customerId, String firstName, String lastName) {
        this.customerId = customerId;
		this.name = new Name(firstName, lastName);
    }
}

public class Name {
	private String firstName;
	private String lastName;
	
	public Name(String firstName, String lastName) {
		this.firstName = firstName;
		this.lastName = lastName;
    }
}
```

이렇게 클래스를 분리함으로써 Customer 클래스는 customerId, name 두개의 인스턴스만 갖게 되었습니다. 
그리고 Name 클래스에 firstName과 lastName을 갖게 되기 때문에 더욱 클래스의 응집도를 더욱 높일 수 있습니다.   

## 9. 게터/세터/프로퍼티를 쓰지 않는다. 

이부분은 Object, DDD 등에서 모두 강조하는 부분입니다. 
묻지말고 시켜라(Tell Don't Ask) 원칙에 따르면 객체에게 묻지 말고 행위를 시키라고 합니다. 보통 **한 객체의 상태에 기반한 모든 행동은 객체가 스스로 결정**하도록 해야합니다.
객체의 외부에서 결정하는데 사용하는 것이 아니라면, 객체의 상태를 가져오기 위해 꼭필요하다면 getter는 사용해도 됩니다. 

```java
public class Game {
	private int score;
	
	public void setScore(int score) {
		this.score = score;
    }
	
	public int getScore() {
		return score;
    }
}

//Usage
game.setScore(game.getScore() + ADDED_GAME_SCORE)
```

위의 코드에서 getScore()는 외부에서 결정하는데 사용되고 있습니다. 
결국, score를 변경하는데 getScore()로 score를 가져오고 추가된 값을 더해주고 있습니다. 이는 Game 객체 내부의 상태를 자신이 스스로 책임지지 못하는 상태입니다. 
또한 setScore()는 어떠한 도메인의 의미를 담고 있지 않습니다. 이것이 DDD에서 setter를 사용하지 말라고 하는 주요 이유입니다. 

getter/setter가 아닌, 객체에게 행위를 시켜보겠습니다. 객체에게 행위를 시킨다는 것은 getter도 필요가 없다는 것을 의미합니다. 
객체 내부의 상태를 알 필요 없이, 그저 행위를 시키면 되기 때문입니다. 

아래는 개선된 로직입니다. 

```java
public class Game {
	private int score;
	
	public void addScore(int delta) {
		score += delta;
    }
}

//Usage
game.addScore(ADDED_GAME_SCORE)
```

위의 코드에서 객체에게 그져 스코어를 더하라는 행위를 시키기만 합니다. getter/setter 도 필요 없습니다.
그리고 addScore 라는 메서드명에서 스코어를 더한다는 도메인의 의미도 제대로 전달됩니다. 

이제 Game 객체는 자신 스스로 전달받은 요청에 대해 결정을 내릴 수 있게 되었습니다.  



