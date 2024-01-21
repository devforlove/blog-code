저는 지금 까지 단위 테스트 코드를 작성하면서 어떤 코드를 단위 테스트 해야 하는지에 대해 항상 의문을 가져왔습니다. 

단위 테스트가 중요하다는 말은 너무나도 많이 들어 왔기 때문에, 모든 코드를 단위 테스트로 만들려고 노력했습니다. 
하지만 최근 [unit testing](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=280870631)를 읽고 모든 코드를 테스트 하는 것은 오히려 안티 패턴이라는 것을 알게 되었습니다. 

**테스트를 해야 하는 코드는 정해져 있던 것입니다.** 이번글은 어떤 코드를 테스트 해야 하는지에 대해 이야기 해보고자 합니다.

왜 모든 코드를 테스트해서는 안될까요? 답은 쉽습니다. 코드는 자산이 아니라 책임이기 때문입니다. 테스트 코드가 많아지면 많아진 만큼 코드를 관리하는 비용이 높아집니다.
그래서 테스트할 가치가 높은 코드를 테스트해야 합니다. 

## 어떤 코드를 테스트해야 하는가?
그렇다면 테스트할 가치가 높은 코드는 무슨 코드일까요? 아래의 두가지 경우 테스트할 가치가 높습니다. 그리고 해당 경우에만 테스트를 진행해야 합니다.
- 클라이언트의 목표 중 하나에 직접적인 연관이 있다. 
- 외부 의존성에서 사이드 이펙트가 발생한다. 

이외의 코드에 대한 모든 테스트는 대상 코드의 내부 구현에 대해 테스트를 하는 것입니다. 내부 구현에 대해 테스트 코드를 작성하면 테스트 코드와 대상 코드의 결합도가 커집니다. 이는 리펙토링 내성의 악화로 이어집니다. 
리펙토링 내성 악화는 코드 내부를 리펙토링를 했고, 정상 동작을 하는데 테스트는 실패하는 경우가 많아진 다는 것을 의미합니다.   

이렇게 테스트 코드가 대상 코드와 결합도가 생겨, 거짓 실패 알림을 보낸다면 무엇이 문제일까요? 테스트 코드의 가치를 희석시킵니다. 테스트 코드는 프로젝트가 지속가능한 개발을 할 수 있도록 도와주는 역할을 합니다. 하지만 
계속해서 거짓 실패 알림을 보낸다면 테스트 코드에 대한 신뢰도가 떨어지고, 개발자들은 리펙토링에 대한 자신감을 잃게 됩니다. 이는 프로젝트가 복잡해지고, 발전할 수록 크게 다가옵니다. 

아래는 유저의 이메일을 변경하는 기능을 구현하는 코드입니다. 
```java
@Service
@RequiredArgsConstructor
public class ChangeEmailService {

	private final UsesrRepository userRepository;

	@Transactional
	public void changeUserEmail(ChangeEmailCommand command) {
        User user = userRepository.findById(command.memberId());
        
        if (!user.isActive();) {
			return;
        }
		
        user.changeEmail(command.email());
        
        userRepository.updateUserEmail(user);
	}
}

@Entity
@Getter
@Table(name = "user")
@NoArgsConstructor
public class User {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	@Column(name = "user_id")
	private Long userrId;

	@Column
	private String userName;

	@Column
	private String userEmail;

	@Column
	private boolean isActive;
	
	public void changeEmail(String email) {
		this.userEmail = email;
    }
	
	public void isActive() {
		return this.isActive;
    }
}
```
위의 ```ChangeEmailService```는 유저의 이메일을 변경하는 ```changeUserEmail``` 메서드를 가지고 있습니다. 그리고 도메인 객체인 ```User```에서 유저가 활성화된 유저인지 확인하고, 활성화된 유저인 경우에만 database에 유저 이메일 변경을 업데이트 합니다. 
위의 경우는 다음의 코드를 테스트해야 합니다. 
- ```ChangeEmailServiec```의 ```changeUserEmail``` 
- ```ChangeEmailServiec```의 ```updateUserEmail```
- ```User```의 ```isActive```
- ```User```의 ```changeEmail```
  
물론 현재 예제가 단순해서 테스트 하는 것이 이상해 보일 수 있습니다. 실제 도메인 로직은 예제 보다 훨씬 복잡한 경우가 대부분일 것입니다.

이메일 변경 기능에서 프레젠테이션 계층에서 service를 호출했다면, 프레젠테이션 계층의 목표는 유저의 이메일을 변경하는 것입니다.
```changeUserEmail```는 정확히 클라이언트의 목표를 반영하고 있습니다. 
그렇다면 ```changeUserEmail``` 목표인 유저의 이메일 변경과 직접적인 연관이 있관이 있는 ```user```의 기능은 자연스럽게 ```isActive```와 ```changeEmail```임을 알 수 있습니다.
도메인 객체의 메서드는 대부분 도메인의 행위를 가지고 있습니다. 따라서 도메인 모델의 메서드는 항상 테스트 대상인 경우가 대부분입니다. 

그렇다면 ```updateUserEmail```은 왜 테스트 대상일까요? 
외부 의존성인 database에 사이드 이펙트를 남기기 때문입니다. 유저의 변경된 이메일이 database에 반영되고 이는 비즈니스 로직의 목표와 연관이 있습니다. 보통은 이런 사이드 이펙트를 남기는 의존성은 목으로 대체 하여 
적절한 사이드 이펙트를 남기는지 검증합니다. 

그렇다면 한가지 궁금점이 생깁니다. ```findById```도 같은 외부 의존성인데 왜 검증하지 않을까요? 그 이유는 ```findById```가 사이드 이펙트를 남기지 않기 때문입니다.
따라서 ```findById```는 클라이언트의 목표와 직접전인 연관이 없습니다. 사이드 이펙트를 남기지 않는 외부 의존성은 스텁으로 대체하여 테스트합니다.  
만약 ```findyById```를 테스트에서 검증한다면 테스트 코드와 대상 코드에 높은 결합도가 생기고 리펙토링 내성이 안좋아집니다. 

## 정리
테스트 코드는 도메인의 목표와 연관된 코드에 대해 작성되어야 합니다. 그렇기에 클라이언트의 목표에 직접적인 연관이 있는 코드만이 테스트 대사이 됩니다. 
그리고 사이드 이펙트를 남기는 외부 의존성도 도메인의 목표와 연관이 있으므로 테스트 대상입니다. 만약 이외의 코드를 테스트 한다면 이는 대상 코드에 대한 세부 구현을 테스트한다는 의미입니다. 
이는 테스트 코드와 구현 코드의 결합도를 높여 리펙토링 내성을 낮추게 됩니다. 또한 불필요한 테스트 코드가 많아져 코드 관리 비용을 높입니다.  