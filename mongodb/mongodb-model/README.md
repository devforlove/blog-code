이번글에서는 mongodb의 데이터 모델링에 대해 알아보고자 합니다.
공식 도큐먼트에 설명된 방식에 따르면 크게 두가지 방식이 있습니다.. 

- Embedded Data
- References

## Embedded Data

먼저 Embedded Data 모델입니다. 이 방식에서는 도큐먼트 안에 연관된 도큐먼트를 함께 저장합니다. 연관된 도큐먼트는 배열 형태이거나, 서브 도큐먼트 형태가 됩니다.

유저 컬렉션에 들어가는 도큐먼트에 대한 예시입니다. 
```json
{
  "_id": <ObjectId>,
  "username": "123xyz",
  "contact": {
    "phone": "123-456-7890",
    "email": "xyz@example.com"
  },
  "address": [
    {
      "city": "seoul",
      "zipCode": "123-456"
    },
    {
      "city": "tokyo",
      "zipCode": "abc-789"
    }
  ]
}
```

위의 도큐먼트에서 ```contact```, ```address``` 속성은 각각 서브 도큐먼트와 배열을 값으로 가지고 있습니다. 
이 모델은 애플리케이션 서버에서 한번의 호출로 많은 정보를 가져올 수 있기 때문에 API는 단순한 호출이라는 이점이 있습니다. 
다만 도큐먼트의 사이즈가 커질 수 있습니다. 도큐먼트의 크기가 커지면 working set의 크기가 커져 메모리에 부담을 줄 수 있습니다. 또한 이 방식은 데이터의 중복이 많아지게 되는데, 이는 중복 데이터에 변경이 발생하면 데이터의 일관성을 유지하기 어렵게 만듭니다.

## References

이 모델은 참조해야 할 key만 가지는 모델입니다. 아래 샘플을 보시 듯 곡의 앨범은 앨범ID만 가지고 곡의 아티스트 리스트는 아티스트ID만 가집니다. 
```json
{
    "_id" : ObjectId("5e13540a7c1cd06c3ea596e9"),
    "track_id" : NumberLong(3001),
    "track_nm" : "track title",
    "disk_id" : 1,
          .
          .
          .
    "create_dtime" : ISODate("2019-03-29T12:44:26.000Z"),
    "album" : NumberLong(2001),
    "artist_list" : [
          NumberLong(1001), 
          NumberLong(1002), 
          NumberLong(1003)
    ]
          .
          .
          .
}

```

Reference 모델의 특징은 도큐먼트가 Embedded Data 방식에 비해 가볍고 갱신에 대한 부담이 없습니다. 다만 애플리케이션은 참조하는 대상만큼 쿼리를 호출해야 합니다. 즉 위의 사례에서는 하나의 곡을 조회하기 위해 곡, 아티스트, 앨범 collection에 대해 총 3번 조회해야 합니다.


위의 사례에서 볼 수 있듯이, Embedded Data 방식과 Reference 방식은 서로 상충되는 관계입니다. 
Embedded Data 방식은 조회 성능이 좋지만, 업데이트를 희생합니다. 반면 Reference 방식은 업데이트가 효과적이지만, 조회 성능을 희생합니다. 

이러한 두 방식의 장점을 어느 정도 만족하는 방식이 Subset pattern입니다. 

## Subset pattern 


## 다양한 패턴 

MongoDB의 모델을 구현하는데에는 Subset pattern 이외에도 다양한 패턴들이 있습니다.
- Computed pattern
- Attribute pattern 
- Bucket pattern 
- Extended Reference Pattern 
- Approximation patter
- Tree pattern 
- 
