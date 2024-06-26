뉴스 피드 시스템을 설계해보고자 합니다. 

개략적인 요구사항은 아래와 같다고 하겠습니다. 
- 모바일 앱, 웹 둘다 지원해야 합니다. 
- 사용자는 뉴스 피드에 새로운 스토리를 올릴 수 있고, 친구들이 올리는 스토리를 볼 수도 있습니다. 
- 피드는 시간의 역순으로 노출됩니다. 
- 대규모 시스템에서 확장에 유연하도록 설계 

API는 피드 발행과, 뉴스 피드 읽기 이렇게 2개를 생성할 만들겠습니다.
- 피드 발행: 사용자가 스토리를 포스팅하면 해당 데이터를 캐시와 데이터베이스에 기록한다, 그리고 작성된 피드가 사용자의 친구들의 뉴스 피드에도 전송된다.
- 뉴스 피드 읽기: 친구들이 발행한 피드의 목록을 읽는다. 

## 피드 발행

피드 발행 API는 아래와 같습니다. 
```
POST /v1/me/feed 
```
인자:
- 바디(body): 포스팅 내용에 해당 
- Authorization 헤더: API 호출을 인증하기 위해 사용 

사용자가 POST v1/me/feed로 요청을 전달하면, DNS에서 실제 로드밸런서의 ip를 전달받고 로드밸런서로 요청을 전달합니다. 로드밸런서는 피드를 처리하는 ```피드 저장 서비스```에 요청을 전달합니다.
```피드 저장 서비스```는 피드와 관련된 도메인 로직을 처리하는 서비스입니다. 데이터 안정성과 일관성이 중요하기 때문에 저장소는 RDB를 선택했습니다. 

도메인 로직이 처리된 이후에 피드를 전송하는 것은 핵심 도메인 로직은 아니라고 생각합니다. 따라서 도메인 이벤트가 처리된 이후에 내부 이벤트를 발행할 수 있을 것 같습니다.
따라서 ```피드 저장 서비스```에서 피드를 저장한 이후 내부 이벤트를 메세지 큐로 발행하고 해당 이벤트에 관심을 가진 ```피드 전송 서비스```가 구독을 합니다. 

대략적인 구조는 아래와 같습니다. 











