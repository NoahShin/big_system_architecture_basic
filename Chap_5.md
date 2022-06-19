# 안정 해시

**수평적 규모 확장성**을 달성하기 위해서는 **요청 또는 데이터**를 

**서버에 균등하게** 나누는 것이 중요하다.

안정 해시는 이 목표를 달성하기 위해 보편적으로 사용하는 기술이다.

하지만 우선 이 해시 기술이 풀려고 하는 문제부터 좀 더 자세히 알아야할 필요가 있따.

# 해시 키 재배치(rehash) 문제
예를 들어
N개의 캐시 서버가 있다고 하자.

이 서버들에 부하를 균등하게 나누는 보편적
방법은 아래의 해시 함수를 사용하는 것이다.

> serverIndex = hash(key) % N (N은 서버의 개수이다)

(4개 서버 사용한다고 가정)

어진 각각의 키에 대해서 해시 값과 서버 인덱스를 계산한 예제다.
![](https://velog.velcdn.com/images/noahshin__11/post/bbbdec1d-ee41-4500-b436-95bc36c75655/image.png)

hash(keyO) % 4 = 1 이면, 
클라이언트는 캐시에 보관된 데이터를 가져오기 위해
서버 1에 접속하여야 한다. 

키 값이 서버에 어떻게 분산되는지
![](https://velog.velcdn.com/images/noahshin__11/post/5e5485a9-c08c-4b5d-b19f-151c792c3fa2/image.png)

서버 풀(server pool)의 크기가 고정 되어 있을 때, 
그리고 데이터 분포가 균등할 때는 잘 동작하지만

서버가 추가되거나 기존 서버가 삭제되면 문제가 생긴다. 

예를 들어 1번 서버가 장애를 일으켜 동작을 중단했다고 하면 
서버 풀의 크기는 3으로 변하고, 

키에 대한 해시 값은 변하지 않지만 나머지 % 연산하면 서버 인덱스 값 달라지니까,,(서버 수 1 줄어들어서)
![](https://velog.velcdn.com/images/noahshin__11/post/8bcdd3a3-470d-4ebd-97ee-68139e1d349e/image.png)
![](https://velog.velcdn.com/images/noahshin__11/post/ea118303-59f7-4ec9-a0ea-613c3a6476e4/image.png)

1번 서버가 죽으면 대부분 캐시 클라이언트가 데이터가 없는 엉뚱한 서버에 접속하게 된다는 뜻이다.

그 결과로 **대규모 캐시미스(cache miss)**가 발생하게 될 것이다. 

#### 안정 해시는 이 문제를 효과적으로 해결하는 기술이다.

## 안정 해시 
위키피디아 왈 
> 안정 해시(consistent hash)는 해시 테이블 크기가 조정될 때 
평균적으로 오직 k/n개의 키만 재배치하는 해시 기술이다. 
여기서 k는 키의 개수이고, n은 슬롯(slot)의 개수다. 
이와는 달리 대부분의 전통적 해시 테이블은 슬롯의 수가 바뀌면 거의 대부분 키를 재배치한다.
(뭐라는거야..)


### 해시 공간과 해시 링

이제 안정 해시의 정의는 이해했으니,?????????????????????? 그 동작 원리를 살펴보자.

해시 함수로 SHA-1 
함수의 출력 값 범위는 x0, x1, x2, ~ xn

해시 공간 범위는 0부터 2의 160승 -1까지

해시공간을 그림으로 하면
![](https://velog.velcdn.com/images/noahshin__11/post/3691489b-0dca-4d88-8913-b482e3ff5b6e/image.png)
![](https://velog.velcdn.com/images/noahshin__11/post/d10df154-ecba-48d5-8219-29d497787667/image.png)

### 해시 서버
해시 함수 f 를 사용하면 서버IP나 이름을 이 링위의 어떤 위치에 대응시킬 수 있다.
![](https://velog.velcdn.com/images/noahshin__11/post/f611dd0e-83d7-4bb7-b6f0-88b611d4c8c5/image.png)

### 해시 키
![](https://velog.velcdn.com/images/noahshin__11/post/13781d9a-351f-4452-9196-401f0112c88e/image.png)

### 서버 조회
해당 키의 위치로 부터 시계 방향으로 링을 탐색해 나가면서 만나는 첫 번째 서버다.
따라서 keyO 은 서버 0에 저장되고, keyl은 서버 1에 저장되며, key2는 서버 2, key3은 서버 3에 저장된다.
![](https://velog.velcdn.com/images/noahshin__11/post/3e4bda99-ebd8-4aa0-b9ec-08ee5a8dd361/image.png)



## 안정 해시 기본 구현법의 문제
- 균등 분포 해시 함수를 사용
- 시계 방향 첫 번째의 서버를 지정하는 것


이 해시의 문제점은 균등 분포 해시 함수를 사용해도 
- 해시 파티션(partition) (위 그림의 링 위의 서버 사이의 간격) 의 크기를 균등하게 유지하는게 불가능 하다는 것과
- 이로 인해 키가 균등하게 분포되기 어렵다는 문제가 있다.

이를 해결 하기 위해 가상노드(Virtual Node) or 복제(Replica)라 불리는 기법을 사용한다

![](https://velog.velcdn.com/images/noahshin__11/post/e1c95eaf-3c0e-49d2-9698-aebdd046ebed/image.png)

하나의 서버는 링 위에서 여러 개의 가상 노드를 가지게 된다. (우측 그림)

따라서 안정 해시는 탐색 후 첫번째 만나는 서버를 대상으로 하므로, 가상 노드가 없는 좌측 그림의 구조와 달리 가상 노드가 많을 수록 키의 분포가 균등하게 된다.

가상 노드의 개수를 늘릴 수록 표준 편차의 값은 떨어지지만 가상 노드 데이터를 저장할 공간은 더 많이 필요하므로 시스템의 요구사항에 맞는 적절한 수치가 필요하다.


## 안정 해시의 이점


• 서버가 추가되거나 삭제될 때 재배치되는 키의 수가 최소화된다.
• 데이터가 보다 균등하게 분포하게 되므로 수평적 규모 확장성을 달성하기
쉽다.
• 핫스팟(hotspot) 키 문제를 줄인다. 특정한 샤드(shard)에 대한 접근이 지나
치게 빈번하면 서버 과부하 문제가 생길 수 있다.
케이티 페리, 저스틴 비버, 레이디 가가 같은 유명인의 데이터가 전부 같은 샤드에 몰리는 상황을 생각 해보면 이해가 쉬울 것이다. 
안정 해시는 데이터를 좀 더 균등하게 분배하므로 이런 문제가 생길 가능성을 줄인다.

보라색이 버츄얼 노드  S0 을 여러군데 안착한다.
그리고 해시함수를 여러개 하면 된다.
![](https://velog.velcdn.com/images/noahshin__11/post/47333c54-9905-420c-b88a-aebc855a8518/image.png)


안정 해시는 실제로 널리 쓰이는 기술이다
• 아마존 다이나모 데이터베이스(DynamoDB)의 파티셔닝 관련 컴포넌트(3’ 
• 아파치 카산드라(Apache Cassandra) 클러스터 에서 의 데이터 파티 셔닝41 
• 디스코드(Discord) 채팅 어플리케이셴51
• 아카마이 (Akamai) CDN【61
• 매그레프(Meglev) 네트워크 부하 분산기1기


[참조1](https://velog.io/@mmy789/System-Design-6)
[참조2](https://velog.io/@dev-log/%EC%95%88%EC%A0%95-%ED%95%B4%EC%8B%9C-%EC%84%A4%EA%B3%84Consistent-hashing#:~:text=%EB%B0%9C%EC%83%9D%ED%95%98%EA%B2%8C%20%EB%90%9C%EB%8B%A4.-,%EC%95%88%EC%A0%95%20%ED%95%B4%EC%8B%9C%EC%99%80%20%ED%95%B4%EC%8B%9C%20%ED%85%8C%EC%9D%B4%EB%B8%94,%ED%95%98%EB%8F%84%EB%A1%9D%20%ED%95%98%EB%8A%94%20%ED%95%B4%EC%8B%9C%20%EA%B8%B0%EC%88%A0%EC%9D%B4%EB%8B%A4.)
[참조3](https://www.youtube.com/watch?v=tHEyzVbl4bg)
[참조4](https://www.youtube.com/watch?v=zaRkONvyGr8)


