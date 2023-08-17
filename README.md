## 학습 설명

사전에 알고리즘API를 이용한 [Spring Cloud OpenFeign을 학습](https://github.com/Heo-y-y/study_api_call)하고, 스터디원들과 간단한 서점 대여 서비스를 만들어보는 프로젝트를 진행했다.
- GitHub: <https://github.com/Heo-y-y/study_toy_MSA>

우선 인원이 3명이여서 LoanService을 빼고 나머지 3가지 서비스를 각각 맡아서 진행하기로 했었는데, 내가 **HistoryService**를 담당하기로 했었지만, 스터디원 1명이 중간에 빠져야하는 상황이 생겨 추가적으로 **LoanService**와 **BookService**까지 구현하게 됐다.

- 프로젝트 서비스 관계 흐름도
![스크린샷 2023-06-29 오후 9 10 46](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/a79613cc-4aff-4309-97a8-7e1ff44f8f0a)

일단 보면 과제의 **메인 흐름을 담당**하는 **Loan Service**를 통해 대여 서비스를 주고 받는다.

그리고 **대여 기록을 담당(거의 저장소 역할)**을 하는 **HistoryService**를 통해 실시간으로 변경되는 데이터를 기록하여 **다른 Service가 HistoryService의 데이터를 사용하는 흐름**이다.

일단 UserService와 BookService가 HistoryService에서 조회해야하기 때문에 필자가 대여 횟수와 책 잔여 개수를 보내주는 mock test용 server를 만들기로 했다.

그 뒤에 HistroyService → BookService를 수정하고 → LoanService를 구현했다.
## 테이블 구조
![스크린샷 2023-08-17 오후 5 45 53 1](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/10a6e7a0-5b0d-4ec0-9331-67dd2366cbe5)

## API 요구사항 명세서

4가지 마이크로 서비스로 이루어진 구조를 구현해보며 MSA의 호출 흐름에 익숙해진다.

### Over View

User service, Loan Service, Book Service, History Service로 이루어진 서점 대여 서비스를 구현한다.

```
- Client는 Loan Service에게 대여를 부탁한다.
- Loan Service는 User Service를 통해 인당 최대 대여 횟수 (3권)를 확인한다.
- Loan Service는 Book Service를 통해 책의 남은 수량을 확인한다.
- 최종적으로 대여가 성공적으로 완료되면 해당 트랜잭션을 영속하고 반환한다.
```

### User Service

유저 서비스는 유저가 책을 대여할 수 있는 권한이 있는지를 확인하는 역할을 담당한다.

검증 요청이 발생하면, 다음과 같은 확인을 진행한다.

```
1. 자체 DB를 조회하여 userId 존재 여부를 확인한다.
2. History Service를 조회하여 현재 대여중인 책의 수를 확인한다.
3. 현재 대여중인 책의 수가 3과 같거나 크다면 false, 아니라면 true를 반환한다.
```

**추가 요구 사항**

- user를 등록할 수 있는 API

### [Loan Service](https://github.com/Heo-y-y/study_toy_MSA/tree/main/Loan-Service)

이번 과제의 메인 흐름인 대여를 담당하는 서비스이다.

```
1. UserServie와 BookService에 대여중인 책의 수와 책의 잔여 수를 확인한다.
2. 대여가 성공적으로 이루어지면 HistoryService에게 대여 기록을 보낸다.
```

### [Book Service](https://github.com/Heo-y-y/study_toy_MSA/tree/main/Book-Service)

책의 남은 수량을 확인하는 역할을 담당한다.

```
1. 자체 DB를 조회하여 책의 총 수량을 확인한다.
2. History Service를 조회하여 대여 중인 수량을 비교한다. // 옵션
3. 두 값을 비교하여 남은 수량을 확인한다.
```
**추가 요구 사항**

- 책을 최종적으로 대여한 경우 수량을 감소시키는 API 구현 (동시성 해결을 위해 원하는 방식으로 구현)

### [History Service](https://github.com/Heo-y-y/study_toy_MSA/tree/main/history-service)

책 대여 기록을 담당한다.

```
요구사항
1. 책 대여 기록을 자체 DB에 적재하는 API
2. User Service, Book Service 구현에 필요한 API
```
### [MockTest Service](https://github.com/Heo-y-y/study_toy_MSA/tree/main/mock-test)
UserService와 BookService가 먼저 테스트할 수 있게 도와주는 역할을 담당한다.

### 설계 주의 사항

- 잘 생각해보면, 동시에 여러 요청이 들어왔을 때 문제가 생길 여지가 많습니다.
예를 들어, 동시에 한 유저가 여러 대여 요청을 할 시 User Service를 요청들이 통과하는 시점에는 책 대여 횟수가 0 이지만, 요청 자체가 4개 이상인 경우 해당 유저는 3권 초과의 책을 대여할 수 있게 됩니다.
