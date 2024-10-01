안녕하세요, 우아한테크코스 6기 백엔드 낙낙입니다.

오늘은 MySQL에서 발생할 수 있는 동시성 문제와 이를 해결하기 위한 접근 방식을 소개하려고 합니다.
데이터베이스를 사용하는 애플리케이션에서는 여러 사용자가 동시에 데이터를 조회하거나 수정하려고 할 때, 예기치 못한 동시성 문제가 발생할 수 있습니다.

예를 들어, 같은 데이터에 대해 동시에 접근하려는 트랜잭션 간의 충돌이나 데드락(교착 상태)이 대표적인 문제입니다.
이러한 동시성 이슈는 데이터의 일관성을 깨뜨릴 뿐만 아니라, 애플리케이션의 성능 저하와 사용자의 불편으로 이어질 수 있습니다.

이 글에서는 MySQL의 락 메커니즘과 트랜잭션 격리 수준을 바탕으로 동시성 문제를 이해하고, 실제로 제가 겪었던 '존재하지 않는 레코드'에 대한 동시성 문제를 해결한 경험을 공유하려고 합니다.

저의 경험을 통해 동시성 이슈를 해결할 때 주의해야 할 점들과 문제 해결의 다양한 접근 방식을 공유함으로써, 여러분도 비슷한 문제를 겪을 때 조금 더 수월하게 해결해 나갈 수 있기를 바랍니다.

그럼, 동시성 이슈의 기본 개념부터 시작해보겠습니다.
<br>

# 동시성 이슈란?

동시성 이슈는 여러 스레드(또는 여러 프로세스)가 동시에 공유 자원에 접근할 때 발생하는 문제입니다.
동시에 접근하는 스레드들의 실행 순서에 따라 공유 자원의 최종 상태가 달라질 수 있으며, 이로인해 예측 불가능한 결과가 발생할 수 있습니다.

예를들어 인기 있는 콘서트의 취소표가 하나 생겼다고 해보겠습니다.
이때 A와 B는 취소표로 생긴 자리를 확인하고, 둘 다 동시에 예매를 시도합니다.
만약 DB나 어플리케이션에서 동시성 문제를 제대로 처리하지 않으면, A와 B 둘 다 같은 자리에 대해 예매를 성공하는 문제가 발생할 수 있습니다.

이러한 이유로 개발자는 동시성 문제를 잘 처리할 수 있어야 합니다.
<br>

# 문제 발생 상황

저희 팀은 여행기 장소를 조회하고 저장하는 기능을 개발하던 중, 여러 사용자가 동시에 장소를 저장할 때 **중복된 장소가 저장되는 문제**를 겪었습니다.

문제가 발생한 코드는 다음과 같습니다:

```java
	@Transactional
    public Place getPlace(PlanPlaceCreateRequest planRequest) {
        return placeRepository.findByNameAndLatitudeAndLongitude(
                planRequest.placeName(),
                planRequest.position().lat(),
                planRequest.position().lng()
        ).orElseGet(() -> placeRepository.save(planRequest.toPlace()));
    }
```

`findByNameAndLatitudeAndLongitude` 메소드는 다음과 같이 정의되어 있습니다:

```java
public interface PlaceRepository extends JpaRepository<Place, Long> {

    Optional<Place> findByNameAndLatitudeAndLongitude(String name, String lat, String lng);
}
```

이 메소드는 DB에 해당 장소가 존재하면 해당 장소를 반환하고, 존재하지 않을 경우 새로 저장한 후 반환하는 방식으로 동작합니다.
<br>

## 문제 재현

테스트 코드를 통해 문제를 재현해보면, 다음과 같습니다:

```java
    @Test
    void createTraveloguePlacesWithConcurrency() throws InterruptedException {
        TraveloguePlaceRequest request = new TraveloguePlaceRequest(...);

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(() -> traveloguePlaceService.getPlace(request));
        }
        executorService.shutdown();
        executorService.awaitTermination(30, TimeUnit.SECONDS);

        String placeName = request.placeName();
        TraveloguePositionRequest position = request.position();

        assertThat(placeRepository.findByNameAndLatitudeAndLongitude(placeName, position.lat(), position.lng()))
                .isPresent();
    }
```

위 테스트에서, 10개의 스레드가 동시에 `getPlace`를 호출한 후 `findByNameAndLatitudeAndLongitude` 메소드로 해당 장소가 저장되었는지 확인해보면 다음과 같은 예외가 발생합니다:

```
Query did not return a unique result: 10 results were returned
```

이는 `findByNameAndLatitudeAndLongitude` 메서드가 동일한 장소를 10개 모두 반환하기 때문입니다.
실제로 DB를 확인해보면 다음과 같이 중복된 장소가 10개 저장된 것을 확인할 수 있습니다:
![](https://i.imgur.com/gMbzkiW.png)

이는 **여러 스레드**에서 동시에 `getPlace` 메소드를 호출하면, 모든 스레드에서 `findByNameAndLatitudeAndLongitude`가 빈 `Optional`을 반환하면서 각 스레드가 동시에 `save`를 호출하여 중복 저장되는 것이었습니다.

처음에는 해당 문제를 MySQL에서 제공해주는 락을 이용해서 해결하려 했습니다.
하지만 그 과정에서 데드락(교착상태)이 발생하여 해당 방법으로는 문제를 해결할 수 없었습니다.

왜 데드락이 발생했는지 살펴보기 전에, 먼저 MySQL에서 제공해주는 S/X 락에 대해 간략하게 설명하겠습니다.
<br>

# MySQL에서의 락 - S락/X락

## S 락 (Shared Lock)

S 락(공유 락)은 **읽기 락**이라고도 불리며, 여러 트랜잭션에서 동시에 사용할 수 있는 락입니다.

예를 들어, 트랜잭션 A가 S 락을 획득한 상태에서도 트랜잭션 B가 동일한 데이터에 대해 S 락을 추가로 획득할 수 있습니다.
이렇게 S 락은 트랜잭션 간에 **공유**가 가능하기 때문에, 여러 트랜잭션이 동시에 데이터를 읽을 수 있도록 허용됩니다.

S 락은 주로 **SELECT** 문에서 사용됩니다. S 락을 명시적으로 걸고 싶을 때는 다음과 같이 `SELECT ... FOR SHARE`를 사용하면 됩니다:

```sql
SELECT * FROM place WHERE ... FOR SHARE;
```

참고로 `FOR SHARE`를 붙이지 않은 일반적인 `SELECT` 문은 아무런 락을 걸지 않고 데이터를 조회합니다.
이 경우 다른 트랜잭션에서 락을 걸어 둔 상태에서도 데이터를 읽을 수 있습니다.
<br>

## X 락 (**Exclusive Lock)**

X 락(배타 락)은 **쓰기 락**이라고도 불리며, 이름 그대로 **배타적**으로만 사용할 수 있는 락입니다.

즉, 트랜잭션 A가 X 락을 획득한 상태에서는 다른 트랜잭션 B가 해당 데이터에 대해 **어떠한 락(S락, X락)도** 획득할 수 없습니다.
반대로, 트랜잭션 A가 S 락을 획득한 상태에서도 트랜잭션 B가 해당 데이터에 대해 **X 락**을 획득할 수 없습니다.

X 락은 주로 `INSERT`, `UPDATE`, `DELETE` 같은 **쓰기 작업**을 수행할 때 자동으로 설정됩니다.
만약 `SELECT` 문에서 명시적으로 X 락을 걸고 싶다면, 다음과 같이 `FOR UPDATE`를 사용할 수 있습니다:

```sql
SELECT * FROM place WHERE ... FOR UPDATE;
```

`FOR UPDATE`를 사용하면 해당 데이터를 읽어오는 동시에 X 락이 걸려, 다른 트랜잭션에서 S 락 또는 X 락을 획득하지 못하도록 방지할 수 있습니다.
<br>

# S 락을 걸어보자

처음에는 `Place` 테이블을 조회할 때 S 락(공유 락)을 사용하면 동시성 문제를 해결할 수 있을 것이라 기대했습니다.

이유는 다음과 같습니다. 
만약 한 트랜잭션에서 `Place`를 조회한 후 존재하지 않아 `INSERT`를 수행하면, X 락(배타 락)이 걸리기 때문에 다른 트랜잭션에서 해당 레코드를 읽지 못할 것이라 생각했기 때문입니다.

이에 따라 다음과 같이 `SELECT` 시 S 락을 걸도록 설정하였습니다:

```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<Place> findByNameAndLatitudeAndLongitude(String name, String lat, String lng);
}
```
<br>

## 데드락 발생

하지만, 이전과 동일한 테스트를 수행해 보니 여전히 다음과 같은 **데드락**이 발생했습니다:

```
Exception in thread "pool-3-thread-4" org.springframework.dao.CannotAcquireLockException: could not execute statement [Deadlock found when trying to get lock; try restarting transaction] [insert into place (created_at,deleted_at,google_place_id,latitude,longitude,modified_at,name) values (?,?,?,?,?,?,?)]; SQL [insert into place (created_at,deleted_at,google_place_id,latitude,longitude,modified_at,name) values (?,?,?,?,?,?,?)]
```

이 문제는 10개의 스레드가 동시에 `getPlace` 메서드를 호출하여 `findByNameAndLatitudeAndLongitude`를 수행할 때, 모두 S 락을 획득한 데서 비롯되었습니다.
이후 각 스레드가 `save` 메서드를 호출하려고 할 때 X 락을 요청하지만, 이미 다른 스레드가 S 락을 걸고 있기 때문에 X 락을 획득할 수 없는 상황이 발생한 것입니다.

즉, **10개의 스레드가 모두 S 락을 획득한 상태에서 서로 X 락을 기다리는 교착 상태(데드락)가 발생**하여 모든 스레드가 대기하는 상황에 빠진 것입니다.
<br>

# 그렇다면 X 락을 걸어보자

조회에 X 락을 거는 것은 동시성이 많이 떨어지기 때문에 조심해서 사용해야 하지만, 일단 현재 문제를 해결할 수 있는지 확인하기 위해 X 락을 걸어 보았습니다:

```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Place> findByNameAndLatitudeAndLongitude(String name, String lat, String lng);
}
```

처음부터 `findByNameAndLatitudeAndLongitude`를 수행할 때 X 락을 걸면 데드락이 발생하지 않을 것이라 예상했기 때문이었습니다.
X 락은 공유가 불가능하기 때문에 트랜잭션 하나씩만 락을 획득할 것이라 생각했기 때문입니다.

## 또 다시 데드락 발생
하지만 테스트 코드를 돌려보니 이번에도 데드락이 발생했습니다.

> 결론부터 말하자면, **존재하지 않는 레코드에 대해 락을 걸 때** 발생한 문제였습니다. 
> `SELECT ... FOR UPDATE` 문을 실행할 때 스캔되는 인덱스의 결과가 없는 경우 MySQL은 갭 락이나 supremum pseudo-record X 락을 거는데, 해당 락은 서로 다른 트랜잭션이 동시에 획득 가능해서 발생한 문제였습니다.

지금 당장 위의 내용이 이해가 가지 않아도 괜찮습니다.
천천히 실험을 통해 확인해 봅시다.

처음에는 데드락의 원인이 이해되지 않아 MySQL 서버에서 직접 실험해 보았습니다. 
콘솔 2개를 열고 다음 명령어들을 실행해 보았습니다:

```sql
start transaction;  
  
SELECT *  
FROM place  
WHERE name = 'place1'  
  AND latitude = '12.345'  
  AND longitude = '12.345'  
FOR UPDATE;  
  
insert into place(created_at, name, latitude, longitude)  
    value ('2024-01-01', 'place1', '12.345', '12.345');  
  
rollback;
```

MySQL의 락은 트랜잭션이 **커밋**되거나 **롤백**될 때 해제되므로, 먼저 트랜잭션을 시작했습니다.

트랜잭션 A가 `SELECT ... FOR UPDATE` 쿼리를 먼저 실행하면, X 락이 걸려 트랜잭션 B는 같은 `SELECT ... FOR UPDATE` 쿼리를 실행할 때 A가 롤백될 때까지 대기할 것이라 예상했습니다.

하지만 예상과 달리 트랜잭션 B도 `SELECT` 문을 실행하자마자 결과가 바로 반환되었습니다.
혹시 락이 걸리지 않았나 확인하기 위해 다음 명령어로 락 정보를 확인하였습니다:

```sql
SELECT * FROM performance_schema.data_locks;
```

결과는 다음과 같았습니다:

![](https://i.imgur.com/LFzPV70.png)

<br>

### 락은 인덱스와 밀접한 관계가 있다.
분명 X 락은 동시에 획득이 불가능하다고 말했는데, 왜 이러한 결과가 나온 것일까요?

이를 위해서는 먼저 락과 인덱스 사이의 관계를 이해해야 합니다.

MySQL 공식 문서를 보면, SQL 은 S/X 락을 걸 때 레코드 단위로 락을 건다고 나와 있습니다.
좀 더 정확하게 설명하면, **SQL 문을 실행할 때 스캔되는 모든 인덱스 레코드에 락을 겁니다.**

그런데 만약 SQL 문을 실행할 때 인덱스의 레코드를 단 하나도 스캔하지 못한다면 어떻게 될까요? 


현재 place 테이블은 테스트를 위해 비어져 있는 상황이었고, PK 를 제외하고는 어떠한 인덱스도 걸려 있지 않은 상황이었습니다.

이 상황에서 `SELECT * FROM place WHERE ... FOR UPDATE` 를 실행하면 PK 인덱스를 스캔하는데, 현재 테이블이 비어 있기 때문에 인덱스 스캔 결과로 어떠한 레코드도 나오지 않게 됩니다.

이때 MySQL은 갭 락이나 supremum pseudo-record X 락을 겁니다.
<br>

### 갭 락과 Supremum pseudo-record X 락

갭 락(Gap Lock)은 **두 인덱스 레코드 사이의 간격**에 대해 걸리는 락으로, 특정 구간에 새로운 레코드가 **삽입**되지 않도록 막는 락입니다. 

예를들어 인덱스에 레코드 1, 5, 10 이 있는 경우 각 레코드에 대한 갭 락은 다음과 같습니다. 
- 레코드 1에 대한 갭 락은 -∞ ~ 0 사이에 락을 거는 것입니다.
- 레코드 5에 대한 갭 락은 2 ~ 4 사이에 락을 거는 것입니다.
- 레코드 10에 대한 갭 락은 6 ~ 9 사이에 락을 거는 것입니다.

그렇다면 10보다 큰 레코드가 삽입되는 것을 막기 위해서는 어떻게 해야할까요?
이를 위해 거는 것이 바로 supremum pseudo-record X 락입니다.

supremum pseudo-record X 락은 InnoDB 인덱스에서 **가장 큰 레코드보다 큰 값이 삽입**되지 않도록 막는 락입니다.
앞에서 본 것 처럼 갭 락은 특정 레코드 **앞의 간격**(gap)에 대해서만 잠금을 걸 수 있기 때문에, 인덱스에서 **가장 큰 레코드 이후의 간격에는 갭 락을 걸 수 없습니다**.  

따라서, 인덱스의 **가장 큰 레코드 이후의 값 삽입을 제어하기 위해** `Supremum pseudo-record X 락`을 사용합니다.

