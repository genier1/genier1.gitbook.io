# Consumer Group 정리

#### Consumer Group이란

* 개별 Consumer Client를 묶는 논리적인 그룹 단위. 즉, 컨슈머 그룹은 논리적으로는 하나의 컨슈머이다.
* 같은 group.id 설정값을 가진 consumer들은 하나의 Consumer Group을 이룬다.
* 같은 Consumer Group에 속한 Consumer들이 Topic에 속한 파티션을 나눠서 읽는다.
* Consumer Group Coordinator가 Consumer Group을 관리하며, Consumer Group 내부에는 Leader가 파티션을 할당한다.

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/ac13f8e4-32ab-4e11-87ce-63dd800c2bed" %}

#### 왜 Consumer Group을 사용하는가

* 가용성
  * 컨슈머 그룹 내의 컨슈머 인스턴스에 장애가 발생하더라도, 같은 컨슈머 그룹의 다른 인스턴스로 대체가 가능하다.
  * 브로커를 재시작할 필요 없이 더 유연하고 확장 가능한 파티션을 지원할 수 있음
* Offset 관리
  * 컨슈머 그룹들은 자신의 그룹에 대한 offset 관리를 한다.
  * 동일한 토픽을 여러 컨슈머 그룹이 컨슘하더라도 서로 다른 offset을 가지고 데이터 손실 없이 메시지를 처리할 수 있다.

#### Consumer Group과 Partition의 관계

* 컨슈머 그룹의 컨슈머 수와 파티션 수가 같다면 컨슈머 그룹의 컨슈머 한개당 파티션 하나가 할당 됨

{% embed url="https://github-production-user-asset-6210df.s3.amazonaws.com/19471818/325974580-9c89e77d-3413-48df-8c8d-72b56169ea7b.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA/20240427/us-east-1/s3/aws4_request&X-Amz-Date=20240427T111318Z&X-Amz-Expires=300&X-Amz-Signature=5f56bafc0c5b2fd39f6502b147bdd5143bed5f0c1b3f261934e14452d742c78a&X-Amz-SignedHeaders=host&actor_id=19471818&key_id=0&repo_id=783676058" %}
출처: [https://www.popit.kr/kafka-consumer-group/](https://www.popit.kr/kafka-consumer-group/)
{% endembed %}

* 컨슈머 그룹의 컨슈머 수 < 파티션 수
  * 하나의 컨슈머에 여러개의 파티션이 할당됨

{% embed url="https://github-production-user-asset-6210df.s3.amazonaws.com/19471818/325974727-660b5fb2-b09c-45b4-b4d8-a8bc0e204e19.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA/20240427/us-east-1/s3/aws4_request&X-Amz-Date=20240427T111336Z&X-Amz-Expires=300&X-Amz-Signature=49bfb2446751603c116339e96fa587fe8722be410b310579bae981167429da0b&X-Amz-SignedHeaders=host&actor_id=19471818&key_id=0&repo_id=783676058" %}
출처: [https://www.popit.kr/kafka-consumer-group/](https://www.popit.kr/kafka-consumer-group/)
{% endembed %}

* 컨슈머 그룹의 컨슈머 수 > 파티션 수
  * 하나의 파티션에는 하나의 컨슈머 인스턴스만 접근할 수 있음
  * 즉, 파티션 수를 초과하는 컨슈머는 유휴상태로 남는다.

{% embed url="https://github-production-user-asset-6210df.s3.amazonaws.com/19471818/325974745-f78d5fa7-58bd-49f0-8297-c50be79acaa3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA/20240427/us-east-1/s3/aws4_request&X-Amz-Date=20240427T111216Z&X-Amz-Expires=300&X-Amz-Signature=c8e7539c4728bf787d88c0ab38c60bb50920f2d797c10c9afacad6f6dcd38e20&X-Amz-SignedHeaders=host&actor_id=19471818&key_id=0&repo_id=783676058" %}
출처: [https://www.popit.kr/kafka-consumer-group/](https://www.popit.kr/kafka-consumer-group/)
{% endembed %}

* 하나의 토픽에 2개 이상의 컨슈머 그룹
  * 하나의 토픽에 대해 각 Group 별로 메시지를 가져가서 처리한다.

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/499bafac-5fb8-4395-9b0c-d05597b62b55" %}
출처: [https://www.popit.kr/kafka-consumer-group/](https://www.popit.kr/kafka-consumer-group/)
{% endembed %}

#### 리밸런싱

리밸런싱이란, Consumer Group에 변동사항이 발생할 경우 해당 그룹 안에서 파티션의 소유권을 조정하는 작업이다.

리밸런싱은 다음과 같은 상황에서 발생할 수 있다.

* Consumer Group 내에 컨슈머가 생성 혹은 삭제된 경우
* max.poll.interval.ms로 설정된 시간 내에 poll() 요청을 보내지 못한 경우
* session.timeout.ms로 설정된 시간 내에 하트비트를 보내지 못한 경우

리밸런싱이 발생할 경우, 해당 컨슈머 그룹은 리밸런싱이 완료될 때까지 메시지를 처리할 수 없다. 또한 메시지 중복 컨슈밍 현상이 발생할 수도 있다.

#### 참고 링크

* [https://deview.kr/data/deview/session/attach/\[124\]네이버스케일로카프카컨슈머사용하기.pdf](https://deview.kr/data/deview/session/attach/\[124]%EB%84%A4%EC%9D%B4%EB%B2%84%EC%8A%A4%EC%BC%80%EC%9D%BC%EB%A1%9C%EC%B9%B4%ED%94%84%EC%B9%B4%EC%BB%A8%EC%8A%88%EB%A8%B8%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0.pdf)
* [https://techblog.gccompany.co.kr/카프카-컨슈머-그룹-리밸런싱-kafka-consumer-group-rebalancing-5d3e3b916c9e](https://techblog.gccompany.co.kr/%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%BB%A8%EC%8A%88%EB%A8%B8-%EA%B7%B8%EB%A3%B9-%EB%A6%AC%EB%B0%B8%EB%9F%B0%EC%8B%B1-kafka-consumer-group-rebalancing-5d3e3b916c9e)
* [https://devocean.sk.com/community/detail.do?ID=165478\&boardType=DEVOCEAN\_STUDY\&page=1](https://devocean.sk.com/community/detail.do?ID=165478\&boardType=DEVOCEAN\_STUDY\&page=1)
* [https://www.popit.kr/kafka-consumer-group/](https://www.popit.kr/kafka-consumer-group/)
