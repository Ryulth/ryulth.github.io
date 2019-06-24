---
layout: post
title:  "[Pattern]Event Sourcing Pattern이란"
subtitle: "Event Sourcing Pattern이란"
categories: devnote
tags: pattern
---

> Event Sourcing Pattern을 간단하게 알아보자

## Event Sourcing Pattern

* Status 를 update 를 하는 것이 아닌 Status 가 변화는 일련의 이벤트를 저장하는 형태이다.


### 기존 방식

예를 들어 재고 관리 프로그램이라고 하자. 어떠한 DB 를 사용하든 수량이 변하는 과정에서 DB에 update 로직을 반영시켜 최종 상태를 반영했다. 수량이 변하는 기록은 단순히 기록용 로그로 남겨져서 큰 활용은 되지 않고 있었다. 심지어 로깅을 안 하는 경우도 있다. 

``` java
public void updateStock(Request request) {
    Stock stock = stockRepository.getOne(request.getStockId()); // 요청한 stock id 로 DB 에서 읽어온다
    stock.setCount(stock.getCount + request.stockModifyCount()); // 요청한 변경값을 반영한다.
    stockRepository.save(stock); // DB 에 업데이트 해준다.
}
```

이런식으로 간단하게 작성해 볼 수 있다.

Request 목록

1. id 가 1인 상품을 재고에 등록한다. - add()
2. id 가 1 인 상품의 재고를 3개 추가한다. - update(3)
3. id 가 1 인 상품의 재고를 2개 추가한다. - update(2)
4. id 가 1 인 상품의 재고를 3개 삭제한다. - update(-3)

이 방식으로 구현 하면 최종 디비에는 id 1 인 값에 +3+2-3 해서 2 라는 재고를 가지고 있다고 보일 것이다.

하지만 재고 3개 삭제라는 이벤트가 잘못된 이벤트라고 판단되었고 그날의 모든 요청이 다 버그여서 되돌리려고 한다면 로그를 뜯어보면서 수정해야 할 것이다. 

### 이벤트 도메인 저장

이제 재고 관리 프로그램이라고 한다면 변경하는 request 를 이벤트 도메인으로 만들어 저장해보자.

1. id 가 1인 상품을 재고에 등록한다. 
2. id 가 1 인 상품의 재고를 3개 추가한다.
3. id 가 1 인 상품의 재고를 2개 추가한다.
4. id 가 1 인 상품의 재고를 3개 삭제한다.

DB 에 반영시 이런 식 으루

| id   | version | eventType | metaData |
| ---- | ------- | --------- | -------- |
| 1    | 1       | ENROLL    | Count:0  |
| 1    | 2       | ADD       | Count:2  |
| 1    | 3       | ADD       | Count:2  |
| 1    | 4       | DELETE    | Count:-3 |

현재 상태의 값은 간단하게

```java
public long getCount(List<Event> events){
    long sum = 0L;
    for (Event event : events){
        sum += event.getCount();
    }
    return sum;
}
```

이런 식으로 구하면 된다.

### 이벤트 소싱 패턴의 장점

* 굳이 데이터 스트림을 많이 생성하고 저장해야 하는 이유가 번거롭고 비효율적 이라고 생각 할 수 있다. 여러가지 장점이 존재한다.

1. 특정 시점까지의 이벤트를 재생하면 해당 시점의 모델의 Status 을 얻을 수 있다. 
2. 이벤트 데이터는 수정이나 삭제가 없기 때문에 데이터의 불변성을 유지 할 수 있다.
3. MSA 상에서 동기화 처리, 동시성 처리에 좋은 모델이다.

### 스냅샷 기능

이벤트 스트림이 1000개가 있다고 가정한다면 현재의 상태를 얻기 위해 1000번의 loop 를 돌려야할 것이다. 한 과정의 적용을 apply 라고 칭한다면 1000번의 apply 가 존재하며 추가비용이 발생 할 수 있다. 따라서 스냅샷이라는 개념을 도입하면 성능저하를 막을 수 있다.

1000개의 이벤트 스트림을 apply 하는데 비용이 0.5 초 든다고 하고 500개의 이벤트 스트림을 apply 하는 과정이 0.25초 걸린다고 가정하자. 사람이 변화를 감지할 수 있는 시간은 0.3초라고 한다. 그렇다면 500개씩 스트림을 끊어서 복원과정을 거친다면 실제 성능저하를 체감하지 못할 것이다. 이러한 개념으로 접근 한 것이 스냅샷 개념이다. 

이벤트 스트림이 500개 이상으로 넘어가면 500번 apply 를 시켜 임시 저장 버젼을 데이터로 가지고 있고 500번대 부터 요청이 들어오면 500 번대부터 apply 를 시켜서 데이터를 재생한다. 

말이 어렵게 썻지만 그냥 게임에서 중간 체크 포인트? 중간 세이브 라고 생각하면 된다. 

#### TODO 

* [Apache Kafka](http://kafka.apache.org/) 를 통해 메시지 스트림 구현해보기 

