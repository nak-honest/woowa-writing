# 동시성 문제(데드락) 해결기 : X 락인데 왜 공유가 가능하지??????
안녕하세요. 우아한테크코스 6기 백엔드 낙낙입니다.

이번 글에서는 팀 프로젝트에서 겪은 동시성 문제와 MySQL의 락을 사용하다가 경험한 데드락에 관해 이야기합니다. 특히 **갭 락은 X 락이더라도 공유가 가능하므로 데드락이 발생할 수 있다는 사실**을 강조합니다.

이 과정에서 MySQL의 락 메커니즘을 깊이 있게 다룹니다. 그리고 이 문제를 락을 사용하지 않고 어떻게 해결했는지 공유합니다.

이 글을 통해 MySQL의 락 메커니즘을 이해하고 락을 사용할 때 발생할 수 있는 데드락의 원인을 파악할 수 있습니다. MySQL의 락을 더욱 안전하게 활용하는 데 도움이 되기를 바랍니다.

## 문제 발생 상황

팀에서 여행기 장소를 조회하고 저장하는 기능을 개발하던 중, 여러 사용자가 동시에 장소를 저장할 때 중복된 장소가 저장되는 문제를 겪었습니다.

문제 설명 전 알아둘 프로젝트의 사전 지식은 다음과 같습니다.
- 현재 진행 중인 프로젝트는 다녀온 여행기를 기록할 수 있는 서비스입니다.
- 여행기에 사용되는 여행 장소는 여러 여행기에서 등장할 수 있습니다.
- 따라서 여행 장소는 `place` 라는 테이블 하나로 관리되고, 여러 여행기가 이 장소를 공유합니다.

이 상황에서 문제가 발생한 코드는 다음과 같습니다:

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

위의 코드는 다음과 같이 동작합니다.
1. 사용자가 여행기 작성을 요청하면 각 여행기 장소가 존재하는지 확인합니다. 존재한다면 바로 반환합니다.
2. 만약 존재하지 않는다면 DB에 새로 추가하고 반환합니다.

장소를 저장할 때 해당 장소가 존재하는지 미리 확인하기 때문에 중복으로 저장되지 않을 것이라 예상했습니다. 하지만 **여러 명이 동시에 호출하면 중복 저장이 되는 문제**가 발생했습니다. 

## 문제 재현

테스트 코드를 통해 문제를 재현해 보면 다음과 같습니다:

```java
    @Test
    void createTraveloguePlacesWithConcurrency() throws InterruptedException {
        TraveloguePlaceRequest request = new TraveloguePlaceRequest(...);
		
	// 스레드 10개 생성
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        
        // 10개의 스레드가 동시에 getPlace() 호출
        for (int i = 0; i < 10; i++) {
            executorService.execute(() -> traveloguePlaceService.getPlace(request));
        }
        
	// 모든 작업이 끝날 때까지 최대 30초 대기
        executorService.shutdown();
        executorService.awaitTermination(30, TimeUnit.SECONDS);
		
        String placeName = request.placeName();
        TraveloguePositionRequest position = request.position();
		
	// DB에 해당 place 가 하나만 저장되었는지 확인
        assertThat(placeRepository.findByNameAndLatitudeAndLongitude(placeName, position.lat(), position.lng()))
                .isPresent();
    }
```

위 테스트에서 10개의 스레드가 동시에 `getPlace`를 호출한 후 `findByNameAndLatitudeAndLongitude` 메소드로 해당 장소가 1개만 저장되었는지 확인했습니다. 
하지만 다음과 같은 예외가 발생합니다:

```
Query did not return a unique result: 10 results were returned
```

이는 동일한 장소가 중복으로 10개 저장되어서 발생한 문제였습니다.
실제로 DB를 확인해 보면 다음과 같이 중복된 장소가 10개 저장된 것을 확인할 수 있습니다:

<img src="https://i.imgur.com/gMbzkiW.png" width="600" />

**원인은 모든 스레드가 해당 장소를 조회할 때 존재하지 않는다는 결과를 받고, 동시에 `save` 를 호출해서 발생한 것이었습니다.**

처음에는 해당 문제를 **MySQL에서 제공해 주는 락**을 이용해서 해결하려 했습니다.
하지만 그 과정에서 데드락(교착상태)이 발생하여 해당 방법으로는 문제를 해결할 수 없었습니다.

왜 데드락이 발생했는지 살펴보기 전에, 먼저 MySQL에서 제공해 주는 S/X 락에 대해 간략하게 설명하겠습니다.

## MySQL에서의 락 - S락/X락

### S 락 (Shared Lock)

**S 락**(공유 락)은 읽기 락이라고도 불리며, **여러 트랜잭션에서 동시에 획득할 수 있는 락**입니다.

예를 들어, 트랜잭션 A가 S 락을 획득한 상태에서 트랜잭션 B도 S 락을 동시에 획득할 수 있습니다.
이렇게 S 락은 트랜잭션 간에 공유가 가능하므로 여러 트랜잭션이 동시에 데이터를 읽을 수 있도록 허용됩니다.

S 락은 SELECT 문에서 사용됩니다. S 락을 명시적으로 걸고 싶을 때는 다음과 같이 `SELECT ... FOR SHARE`를 사용하면 됩니다:

```sql
SELECT * FROM place WHERE ... FOR SHARE;
```

참고로 `FOR SHARE`를 붙이지 않은 일반적인 `SELECT` 문은 아무런 락을 걸지 않고 데이터를 조회합니다.
이 경우 다른 트랜잭션에서 락을 걸어 둔 상태에서도 데이터를 읽을 수 있습니다.

### X 락 (**Exclusive Lock)**

**X 락**(배타 락)은 쓰기 락이라고도 불리며, 이름 그대로 **배타적으로만 사용할 수 있는 락**입니다.

즉, 트랜잭션 A가 X 락을 획득한 상태에서는 다른 트랜잭션 B가 **어떠한 락(S락, X락)도** 획득할 수 없습니다.
반대로 트랜잭션 A가 S 락을 획득한 상태에서도 트랜잭션 B가 **X 락**을 획득할 수 없습니다.

X 락은 주로 `INSERT`, `UPDATE`, `DELETE` 같은 쓰기 작업을 수행할 때 자동으로 설정됩니다.
만약 `SELECT` 문에서 명시적으로 X 락을 걸고 싶다면, 다음과 같이 `FOR UPDATE`를 사용할 수 있습니다:

```sql
SELECT * FROM place WHERE ... FOR UPDATE;
```


`FOR UPDATE`를 사용하면 해당 데이터를 읽어오는 동시에 X 락이 걸려, 다른 트랜잭션에서 S 락 또는 X 락을 획득하지 못하도록 방지할 수 있습니다.

표로 정리해 보면 다음과 같습니다:

|               | **S-lock 요청** | **X-lock 요청** |
| ------------- | ------------- | ------------- |
| **S-lock 보유** | 허용            | 거부            |
| **X-lock 보유** | 거부            | 거부            |

## S 락을 걸어보자

처음에는 `Place` 테이블을 조회할 때 S 락(공유 락)을 사용하면 동시성 문제를 해결할 수 있을 것이라 기대했습니다.

이유는 다음과 같습니다. 
만약 한 트랜잭션에서 `Place`를 조회한 후 존재하지 않아 `INSERT`를 수행하면, X 락(배타 락)이 걸리기 때문에 다른 트랜잭션에서 해당 레코드를 읽지 못할 것으로 생각했기 때문입니다.

이에 따라 다음과 같이 `SELECT` 시 S 락을 걸도록 설정하였습니다:

```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<Place> findByNameAndLatitudeAndLongitude(String name, String lat, String lng);
}
```

### 데드락 발생

하지만, 이전과 동일한 테스트를 수행해 보니 여전히 다음과 같은 **데드락**이 발생했습니다:

```
Exception in thread "pool-3-thread-4" org.springframework.dao.CannotAcquireLockException: could not execute statement [Deadlock found when trying to get lock; try restarting transaction] [insert into place (created_at,deleted_at,google_place_id,latitude,longitude,modified_at,name) values (?,?,?,?,?,?,?)]; SQL [insert into place (created_at,deleted_at,google_place_id,latitude,longitude,modified_at,name) values (?,?,?,?,?,?,?)]
```

지금부터 데드락이 발생한 원인을 살펴보겠습니다.

위의 상황에서 10개의 스레드가 동시에 `getPlace` 메서드를 호출하면, `findByNameAndLatitudeAndLongitude`를 통해 각 스레드가 S 락을 동시에 획득합니다.

이후 각 스레드가 `save` 메서드를 호출하여 X 락을 요청하지만, 이미 다른 스레드들이 S 락을 보유하고 있어 X 락을 획득할 수 없는 상태가 됩니다.

즉, **10개의 스레드가 모두 S 락을 획득한 채 서로 X 락을 기다리는 데드락에 빠져, 모든 스레드가 대기하는 상황이 발생한 것입니다.**

#### 참고 사항
> `(name, latitude, longitude)` 컬럼에 인덱스가 설정되어 있는지에 따라 데드락의 원인이 다소 달라집니다. 위의 상황에서 인덱스가 없는 경우 테이블 전체(정확히는 기본 키(PK) 인덱스 전체)에 락이 걸리며, 인덱스가 있는 경우 갭 락이 발생하게 됩니다.
> 
> 우선 지금은 S 락이 공유 가능하다는 특징만 알고 있어도 충분하므로, 갭 락에 대해서는 뒤에서 설명하겠습니다.

## 그렇다면 X 락을 걸어보자
### 인덱스가 없으면 잘 동작한다.
앞에서 여러 스레드가 동시에 S 락을 획득하면서 데드락이 발생하는 상황을 확인했습니다. 그렇다면 만약 SELECT 시 X 락을 걸게 된다면 결과는 어떻게 될까요?

조회 시 X 락을 거는 것은 동시성을 크게 저하할 수 있어 신중히 사용해야 하지만, 현재 문제를 해결할 수 있는지 확인하기 위해 X 락을 설정해 보았습니다:

```java
public interface PlaceRepository extends JpaRepository<Place, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Place> findByNameAndLatitudeAndLongitude(String name, String lat, String lng);
}
```

이렇게 설정하면 10개의 스레드가 동시에 `findByNameAndLatitudeAndLongitude`를 호출하더라도, X 락은 하나의 스레드만 획득하게 되고 나머지 9개 스레드는 락을 얻지 못한 상태에서 대기하게 됩니다.

이후 X 락을 획득한 스레드는 `save` 메서드를 호출할 때 다른 스레드가 추가적인 락을 걸지 않았기 때문에 저장에 성공할 수 있습니다. 이후 commit 시 X 락이 해제되고, 대기 중인 다른 스레드 중 하나가 X 락을 얻게 되면서 데드락이 발생하지 않게 됩니다.

(테이블이 비어 있는 상황에서는 데드락이 발생하는데, 해당 내용은 뒤에서 자세히 설명합니다.)

### 인덱스를 걸면 데드락이 발생한다.
우선 `(name, latitude, longitude)` 컬럼에 다음과 같이 인덱스를 추가했습니다:
```sql
create index place_idx on place(name, latitude, longitude);
```


이후 테스트를 진행해 보니, 데드락이 발생했습니다. 
분명 X 락은 동시에 획득할 수 없다고 했는데, 왜 이런 문제가 생겼을까요?

저는 데드락의 원인이 이해되지 않아 MySQL 서버에서 직접 실험해 보았습니다.
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

MySQL의 락은 트랜잭션이 커밋되거나 롤백될 때 해제됩니다. 따라서 각 콘솔에서 트랜잭션을 시작했습니다. 그 후 두 트랜잭션에서 동시에 `SELECT` 문을 호출하였습니다.

트랜잭션 A가 `SELECT ... FOR UPDATE` 쿼리를 먼저 실행하면 X 락이 걸고 조회에 성공합니다.
트랜잭션 B가 같은 `SELECT ... FOR UPDATE` 쿼리를 실행하면 A가 이미 X 락을 가지고 있기 때문에 대기할 것이라 예상했습니다.

하지만 예상과 달리 트랜잭션 B도 `SELECT` 문을 실행하자마자 결과가 바로 반환되었습니다.
혹시 락이 걸리지 않은 것인지 확인하기 위해 다음 명령어로 락 정보를 확인하였습니다:

```sql
SELECT * FROM performance_schema.data_locks;
```

결과는 다음과 같았습니다:

![](https://i.imgur.com/BVkfm8r.png)


두 트랜잭션이 `X, GAP` 락을 동시에 획득한 것을 볼 수 있습니다.


### 락은 인덱스와 밀접한 관계가 있다.
분명 X 락은 동시에 획득할 수 없다고 말했는데, 왜 이러한 결과가 나온 것일까요?

이를 위해서는 먼저 락과 인덱스 사이의 관계를 이해해야 합니다.

[MySQL 공식 문서[1]](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)를 보면, SQL 은 S/X 락을 걸 때 레코드 단위로 락을 건다고 나와 있습니다.
좀 더 정확하게 설명하면, **SQL 문을 실행할 때 스캔 되는 모든 인덱스 레코드에 락을 겁니다.**

매번 락을 걸 때마다 테이블 전체에 락을 걸면 동시성이 매우 떨어지기 때문에, MySQL에서는 인덱스를 활용한 레코드 기반의 락 메커니즘을 제공합니다.

하지만 SQL 문 실행 시 인덱스의 레코드를 단 하나도 스캔하지 못한다면 어떻게 될까요? 이 경우 **MySQL은 팬텀 리드를 방지하기 위해 갭 락(또는 Supremum pseudo-record 락)을 걸게 됩니다.**
이는 다음의 상황을 방지하기 위함입니다.

![](https://i.imgur.com/zzqO1m0.png)

트랜잭션 A가 `('place1', '12.345', '12.345')`를 조회할 때 락을 걸며 데이터를 읽습니다. 
이때 다른 트랜잭션 B가 같은 `('place1', '12.345', '12.345')`를 삽입한다고 가정해 봅시다. 
만약 트랜잭션 A가 다시 `('place1', '12.345', '12.345')`를 조회하면, 트랜잭션 B가 방금 삽입한 결과를 얻게 됩니다.

위처럼 조회한 결과에서 레코드가 새로 추가되거나 삭제되는 현상을 팬텀 리드(Phantom Read)라고 합니다. 
(참고로 MySQL은 `REPEATABLE READ` 이상의 격리 수준에서만 팬텀 리드를 방지하기 위해 갭 락을 겁니다.)

그렇다면 갭 락에 대해 자세히 살펴보도록 하겠습니다.

### 갭 락과 Supremum pseudo-record 락

**갭 락(Gap Lock)은 두 인덱스 레코드 사이의 간격에 대해 걸리는 락으로, 특정 구간에 새로운 레코드가 삽입되지 않도록 막는 락입니다.**

프로젝트는 조금 복잡하기 때문에 조금 더 쉬운 예제를 통해 설명하겠습니다.

![](https://i.imgur.com/rG78K1q.png)


위처럼 age에 대한 인덱스가 있습니다. 그리고 레코드는 총 3개의 레코드만 존재합니다.

이 상황에서 각 레코드에 대한 갭 락은 다음과 같습니다.
![](https://i.imgur.com/WTuHRK5.png)

- **age가 20인 레코드(첫 번째 레코드)** 에 대한 갭 락은 해당 레코드의 왼쪽에 락을 거는 것입니다. 그림에서 1번 구간을 의미합니다. 예를 들어 age = 18인 레코드가 삽입되는 것을 막습니다.
- **age가 22인 레코드(두 번째 레코드)** 에 대한 갭 락은 첫 번째와 두 번째 레코드 사이에 락을 거는 것입니다. 그림에서 2번 구간에 락을 겁니다. 예를 들어 age = 21인 레코드가 삽입되는 것을 막습니다.
- **age가 25인 레코드(세 번째 레코드)** 에 대한 갭 락은 두 번째와 세 번째 레코드 사이에 락을 거는 것입니다. 그림에서 3번 구간에 락을 겁니다. 예를 들어 age = 23인 레코드가 삽입되는 것을 막습니다.

그렇다면 **age가 25인 레코드(세 번째 레코드)** 의 오른쪽에 다른 레코드가 삽입되는 것을 막기 위해서는 어떻게 해야 할까요? 즉 4번 구간에 락을 걸고 싶다면 어떻게 해야 할까요?
**이를 위해 거는 것이 바로 supremum pseudo-record 락입니다.**

**supremum pseudo-record 락은 InnoDB 인덱스에서 가장 큰 레코드보다 큰 값이 삽입되지 않도록 막는 락입니다.**
앞에서 본 것처럼 갭 락은 특정 레코드 앞의 간격(왼쪽)에 대해서만 잠금을 걸 수 있기 때문에 인덱스의 가장 큰 레코드 이후의 간격에는 갭 락을 걸 수 없습니다.  

따라서 인덱스의 가장 큰 레코드 이후의 값 삽입을 막기 위해 supremum pseudo-record 락을 사용합니다. 이 락은 인덱스의 끝을 나타내는 허수 레코드에 갭 락을 거는 방식으로 동작합니다. 
허수 레코드는 개념적인 레코드로, 실제로는 존재하지 않는 레코드입니다. 이는 인덱스의 마지막 위치를 가정하고 그 위치에 갭 락을 걸어 해당 구간에 대한 삽입을 방지합니다.

<details>
<summary><h4>참고 : 갭 락이 어떻게 팬텀 리드를 방지하지?</h4></summary>
위의 예제에서 age가 23인 레코드를 X 락과 함께 조회한다고 해보겠습니다:
	
```sql
SELECT * FROM ... WHERE age = 23 FOR UPDATE
```

이 쿼리를 실행하면 3번 구간에 갭 락이 걸리게 됩니다. 
다른 트랜잭션은 이 갭 락 때문에 age가 23인 레코드를 삽입하지 못하게 되고, 이에 따라 팬텀 리드가 발생하지 않게 됩니다.

하지만 age가 23인 레코드뿐만 아니라 age가 24인 레코드도 삽입할 수 없게 됩니다. 동시성이 떨어지는 방식이라고 생각할 수 있지만, 이는 MySQL이 특정 컬럼 값에 대해서만 락을 걸지 못하기 때문입니다. MySQL의 락 메커니즘은 레코드 락과 갭 락을 조합하여 사용하므로 다소 비효율적으로 보이더라도 갭 락을 통해 팬텀 리드를 방지하는 것입니다.
</details>

### 갭 락과 supremum pseudo-record 락은 공유가 가능하다.
문제는 갭 락과 supremum pseudo-record 락이 서로 공유할 수 있다는 점입니다. 
[MySQL 공식 문서[1]](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)에 따르면, 갭 락의 주목적은 데이터를 삽입하는 것을 방지하는 것이기 때문에 서로 충돌하지 않고 공유가 가능합니다. 즉 insert가 되는 것만 막지, 같은 갭 락을 획득하는 것은 막지 않는다는 것입니다.

따라서 X 락을 걸어도 인덱스에 해당 레코드가 없다면 갭 락이 걸리게 되고, 이 갭 락은 여러 트랜잭션이 동시에 획득할 수 있어 데드락이 발생한 것입니다.


앞에서 보았듯이 `(name, latitude, longitude)` 컬럼에 인덱스가 없으면 데드락이 발생하지 않습니다.
이는 인덱스가 없는 경우 PK 인덱스를 스캔하기 때문입니다.

PK 인덱스를 스캔할 때는 모든 범위를 스캔하기 때문에 전체에 락을 걸게 됩니다. 이에 따라 여러 트랜잭션이 동시에 해당 락을 획득할 수 없게 됩니다. 

좀 더 정확히 설명하자면 각 레코드에 넥스트 키 락이 걸립니다. 여기서 넥스트 키 락이란 레코드 락과 갭 락이 결합한 것입니다. 레코드 자체를 잠그는 레코르 락과, 해당 레코드 왼쪽에 대한 삽입을 방지하는 갭 락이 동시에 걸리는 것을 의미합니다. 넥스트 키 락에 대한 설명이 추가되면 내용이 길어질 수 있으므로 여기서는 간단한 개념만 짚고 넘어가겠습니다.

결론적으로 PK 인덱스에서는 모든 레코드와 각 레코드 사이의 간격이 전부 잠기게 됩니다. 그리고 레코드 락은 X 락으로 설정될 때 서로 공유할 수 없기 때문에 위의 상황에서 데드락이 발생하지 않습니다. 


하지만 테이블이 비어 있는 경우에는 데드락이 발생합니다. PK 인덱스를 스캔해도 결과가 나오지 않게 되며, 이때 supremum pseudo-record 락이 걸리면서 여러 트랜잭션이 동시에 획득할 수 있게 됩니다:

![](https://i.imgur.com/fB6J4zc.png)

따라서 데드락이 발생하게 됩니다.


## 그렇다면 인덱스 없이 X 락을 걸면 해결인 것일까?
그렇다면 인덱스 없이 X 락을 걸면 동시성 문제가 해결된 것일까요? 
테이블이 비어 있는 경우에는 인덱스가 없더라도 데드락이 발생할 수 있지만, 실제 프로덕션 환경에서 테이블이 비어 있을 확률은 매우 낮으므로 이는 큰 고려 대상이 아닙니다.

하지만 다음과 같은 큰 단점을 안고 가야 합니다.
- 해당 컬럼에 인덱스를 더 이상 걸 수 없게 됩니다. 
- unique를 걸게 되면 인덱스가 생성되기 때문에, 해당 컬럼들에 대해 unique 제약 조건도 걸 수 없게 됩니다. 
- `SELECT ... FOR UPDATE`는 동시성이 매우 떨어지는 방식이기 때문에 성능 저하 문제도 발생할 수 있습니다.

위의 단점들이 치명적이라 생각했기 때문에 다른 방식도 고려했지만, 다음과 같은 이유로 우아하지 않다고 생각했습니다.
- `READ_UNCOMMITED` 와 함께 어플리케이션 단에서 `synchronized` 사용
    - `READ_UNCOMMITED` 는 가장 낮은 격리 수준이므로 데이터 정합성이 깨질 가능성이 높습니다.
    - `synchronized` 는 성능 저하 문제가 있고, 서버가 다중화되면 사용하기 어려워집니다.
- unique 제약 조건을 걸고 충돌 시 전체 재시도
	- Hibernate는 exception이 터졌다고 세션을 자동으로 초기화해 주지 않습니다. 따라서 JPA에 남아있는 엔티티의 id 값이 null 로 남아 예외가 발생하게 됩니다. 따라서 `entitymanager.clear()` 를 catch 문에서 수행해야 하는데, 이는 코드의 복잡성을 증가시키고 인지 비용을 발생시킵니다.
	- 여행기 장소 저장에 실패할 때, 여행기 작성을 처음부터 전부 다시 해야 합니다. 이미지 관련 처리도 포함되어 있기 때문에 성능 저하가 발생할 수 있습니다.
- 전체 재시도가 아닌 `REQUIRES_NEW` 를 이용해 부분 재시도
	- 테스트에서 커넥션 풀 부족으로 인한 데드락 문제가 발생했습니다. 물론 이는 히카리 풀 사이즈를 조정하면 데드락 문제를 예방하는 것이 가능합니다.
	- `REQUIRES_NEW` 를 사용하는 것은 코드의 복잡성을 증가시키고 인지 비용을 발생시킵니다.

## 다른 해결 방법 - `INSERT IGNORE`
다른 방법들을 추가로 모색하던 중 `INSERT IGNORE`에 대해 알게 되었습니다. 이 쿼리는 중복된 unique key 또는 primary key를 삽입하려고 할 때 해당 삽입을 무시하고 에러를 발생시키지 않는 쿼리입니다.

이 방식을 사용하면 추가적인 락을 걸지 않고, unique 제약 조건이 충돌해도 롤백 및 재시도가 필요 없으므로 성능 저하를 방지할 수 있습니다. 또한 트랜잭션 격리 수준을 낮추지 않아도 되기 때문에 데이터의 정합성도 여전히 유지할 수 있습니다.

### `INSERT IGNORE` 사용 시 주의할 점
JPQL이나 QueryDSL에서는 `INSERT IGNORE`를 직접 지원하지 않으므로, 다음과 같이 Native Query를 사용해야 합니다:

```java
@Modifying(clearAutomatically = true)  
@Transactional  
@Query(value = "INSERT IGNORE INTO place (name, latitude, longitude) VALUES (:name, :latitude, :longitude)", nativeQuery = true)
int saveWithoutDuplication(String name, String lat, String lng);
```

이 메소드는 int를 반환합니다. 이는 삽입된 레코드의 id가 아닌 삽입 성공 여부를 나타냅니다. 따라서 삽입된 데이터의 id를 얻으려면 추가적인 조회가 필요합니다.

이 과정에서 주의할 점이 있습니다. 첫 번째 조회에서 데이터가 존재하지 않아 `INSERT IGNORE`를 실행하는 동안, 다른 트랜잭션이 동일한 데이터를 먼저 삽입할 수 있다는 점입니다. 이 경우 두 번째 조회에서도 데이터가 존재하지 않는다는 결과를 반환합니다. 조회 -> 저장 -> 조회 에서 `REPEATABLE READ` 격리 수준이 적용될 경우, 첫 번째 조회와 두 번째 조회가 동일하게 유지되어 다른 트랜잭션이 삽입한 레코드를 볼 수 없기 때문입니다.

이를 방지하기 위해 첫 번째 조회를 생략하고, 데이터의 존재 여부와 관계없이 매번 INSERT IGNORE를 먼저 실행하는 방식으로 접근해야 합니다.

매번 INSERT 쿼리를 실행하는 방식이 동시성을 약간 떨어뜨릴 수는 있지만, 여행기를 저장하는 과정에서 호출되는 것이기 때문에 크게 비효율적이지는 않다고 판단했습니다.

실제로 테스트를 진행해 본 결과, 데드락이나 unique 충돌 없이 동시성 문제를 효과적으로 해결한 것을 확인할 수 있었습니다.

<img src="https://i.imgur.com/VtZbxuH.png" width="600" />

결론적으로, `INSERT IGNORE`를 활용해 동시성 문제를 해결할 수 있었습니다.

## 마무리
이 글을 통해 MySQL의 동시성 문제와 관련된 다양한 이슈를 살펴보았습니다. 특히 S 락과 X 락의 동작 원리, 여러 스레드가 동시에 S 락을 보유할 때 발생하는 데드락, 그리고 갭 락의 공유 가능성에 대한 내용을 설명하였습니다. 또한 `INSERT IGNORE`를 활용한 해결책이 중복된 unique 키나 기본 키 삽입 시 성능 저하 없이 에러를 방지하며 데이터의 정합성을 유지할 수 있는 효과적인 방법임을 확인했습니다. 

이러한 경험이 여러분이 직면할 수 있는 유사한 문제를 해결하는 데 큰 도움이 되길 바랍니다. 
질문이나 잘못된 내용이 있다면 언제든지 편하게 말씀해 주세요!

## 참고 자료
[1] https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html
