재미있으면서도 고전적인 시스템 설계 문제 

tiny url 같은 URL 단축기를 설계하는 문제

# 1단계: 문제 이해 및 설계 범위 확정

시스템을 성공적으로 설계해 내려면 질문을 통해 모호함을 줄이고 요구사항을 알아내야 한다.

지원자: URL 단축기가 어떻게 동작해야 하는지 예제를 보여줄수 있나?

면접관: https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long 이 입력으로 주어졌다고 해 봅시다. 
이 서비스는 https://tinyurl.com/y7ke-ocwj와 같은 단축 URL을 결과로 제공해야 합니다. 이 URL에
접속하면 원래 URL로 갈 수도 있어야 하죠.

지: 트래픽은 어느 정도 일까요?
면: 매일 1억개의 단축 url을 만들어 낼 수 있어야 합니다.
지: 단축 url 의 길이는 어느 정도여야 하나유?
면: 짧으면 짧을수록 좋다.
지: 단축 url 에 포함될 문자 제한이 있나?
면: 단축 url 에는 숫자, 영대소문자만 사용가능
지: 단축된 url 을 시스템에서 지우거나 갱신할 수 있습니까?
면: 시스템을 단순화하기 위해 삭제나 갱신은 할 수 없다고 가정합시다.

======================
이 시스템의 기본적 기능은

1. URL 단축:주어진긴 URL을 훨씬 짧게 줄인다.
2. URL 리디렉션(redirection): 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
3. 높은 가용성과 규모 확장성, 그리고 장애 감내가 요구됨

개략적 추정
• 쓰기 연산: 매일 1억 개의 단축 URL 생성
• 초당 쓰기 연산: 1억(100million)/24/3600=1160
• 읽기 연산: 읽기 연산 과 쓰기 연산 비율은 10:1 이라고 하자.
그 경우 읽기 연산은 초당 11,600회 발생한다(1160x10 =11,600).
• URL 단축 서비스를 10년간 운영한다고 가정하면 1억(100million)x365x10 = 3650^(365billion) 개의 레코드를 보관해야 한다.
• 축약 전 URL의 평균 길이는 100이라고 하자.
• 따라서 10년 동안 필요한 저장 용량은 3650억(365billion)x 100바이트 = 36.5TB 이다.

# 2단계: 개략적 설계안 제시 및 동의 구하기
## API 엔드포인트(endpoint), URL 리디렉션, 그리고 URL 단축 flow

## API endpoint
클라이언트는 서버가 제공하는 API 엔드포인트를 통해 서버와 통신한다. 우리는 이 엔드포인트를 REST 스타일로 설계할 것이다.

URL 단축기는 기본적으로 두 개의 엔드포인트를 필요로 한다.

> 1. URL 단축용 엔드포인트: 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 한다. 이 엔드포인트는 다음과 같은 형태를 띤다.
-
POST/api/vl/data/shorten
• 인자: {longUrl: longURLstring} • 반환: 단축 URL

> 2. URL 디리렉션용 엔드포인트: 단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트.
-
GET/api/vl/shortUrl
• 반환: HTTP 리디렉션 목적지가 될 원래 URL


단축 url 을 입력하면 무슨일이 생기는가
![](https://velog.velcdn.com/images/noahshin__11/post/a7d17dee-d898-448d-9a43-40a271a96f07/image.png)


단축 URL을 받은 서버는 그 URL을 원래 URL로 바꾸어서 301 웅답의 Location 헤더에 넣어 반환한다.

![](https://velog.velcdn.com/images/noahshin__11/post/b415d7b8-f2c4-42af-bd23-0b445c7798c0/image.png)

여기서 유의할 것은 301 응답과 302 응답의 차이이다. 둘 다 리디렉션 응답이 긴 하지만 차이가 있다.

>• 301 Permanently Moved: 이 응답은 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답이다. 영구적으로 이전되었으므로, 브라우저는 이 응답을 캐시(cache)한다. 따라서 추후 같은 단축 URL에 요청을 보낼 필요가 있을 때 브라우저는 캐시된 원래 URL로 요청을 보내게 된다.

>• 302 Found: 이 응답은 주어진 URL로의 요청이 ‘일시적으로’ Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답이다. 따라서 클라이언트의 요청은 언제나 단축 URL서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야 한다.

⚠️ 장단점:
. 서버 부하를 줄이는 것이 중요하다면 301 Permanent Moved를 사용하는 것이 좋은데 
첫 번째 요청만 단축 URL서버로 전송될 것이기 때문이다. 

하지만 트래픽 분석(analytics)이 중요할 때는 302 Found를 쓰는 쪽이 클릭 발생률이나 발생 위치를 추적하는 데 좀 더 유리 할 것이다.

URL 리디렉션을 구현하는 가장 직관적인 방법은 해시 테이블을 사용하는 것이다. 

해시 테이블에 〈단축 URL, 원래 URL〉 의 쌍을 저장한다고 가정한다면, URL 리디렉션은 다음과 같이 구현될 수 있을 것이다.


> 
• 원래 URL = hashTable.get(단축URL)
• 301또는302응답Location헤더에원래URL을넣은후전송

# URL 단축

단축 UR1이 www.tinyurl.com/{hashValue} 같은 형태라고 해 보자. 결국 중요 한 것은 긴 URL을 이 해시 값으로 대응시킬 해시 함수 fx를 찾는 일이 될 것이다
![](https://velog.velcdn.com/images/noahshin__11/post/a2d14880-7859-47a3-97ea-aaee21f098cf/image.png)
⚠️ 이 해시 함수는 다음 요구사항을 만족해야 한다.
> • 입력으로 주어지는 긴 URL이 다른값이면 해시 값도 달라야 한다.
• 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 한다.

# 3단계: 상세 설계
해시 테이블 좋지만, 메모리는 유한데다 비쌈

〈단축 URL, 원래 URL〉 의 쌍 을 RDBMS 에 널는게 났다.

![](https://velog.velcdn.com/images/noahshin__11/post/9dbebd0c-853a-4fdf-9326-8cdb4a65f949/image.png)

### 해시 함수
해시 함수(hash function)는 원래 URL을 단축 URL로 변환하는 데 쓰인다. 
편의상 해시 함수가 계산하는 단축 URL 값을 hashvalue 라고 하겠따

### 해시 값 길이 
hashValue는 [0-9, a-z, A-Z]의 문자들로 구성된다.
문자 갯수 62 개

hashValue의 길이를 정하기 위해서는 62n승 >= 3650억(365billion)인 n의 최솟값을 찾아야 한다.

![](https://velog.velcdn.com/images/noahshin__11/post/68003139-0753-4adf-8805-a7fa88f86a9d/image.png)

hashValue의 길이는 7로 하도록 하겠다.
7이면 충분

해시 함수 구현에 쓰일 기술 두가지 
‘해시후 충돌 해소’ 방법이고, 
다른 하나는 ‘base-62 변환’ 법이다.

### 해시 후 충돌 해소

긴 URL을 줄이려면, 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요하다. 손쉬운 방법은 CRC32, MD5, SHA-1같이 잘 알려진 해시 함수를 이용하는 것이다
함수를 사용하여 https://en.wikipedia.org/wiki/Systems_design을 축약한 결과다.

![](https://velog.velcdn.com/images/noahshin__11/post/48e63561-9f11-4bde-8449-55b4d77f1dd8/image.png)

 CRC32가 계산한 가장 짧은 해시값조차도 7보다는 길다. 어떻게 하면 줄일 수 있을까?










