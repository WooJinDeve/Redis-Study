## 개요
- [[NHN FORWARD 2021] Redis 야무지게 사용하기](https://www.youtube.com/watch?v=92NizoBL4uA) 를 통해 레디스의 이론적인 부분을 정리한 레퍼지토리입니다.

## Redis 캐시 사용

- **What Is Caching?**
    - `Temporary Location For Speed`
        - 데이터의 원래 소스보다 더 빠르고 효율적으로 액세스할 수 있는 임시 데이터 저장소.
- **서비스에서 유용한 cache 사용**
    - 원본보다 빠른 접근 속도를 제공해야 한다.
    - 동일한 데이터 반복적 엑세스시 `cache` 사용이 유용하다.
    - 변하지 않는 데이터일수록 `cache`사용시 효율적이다.
- **Redis as a cache**
    - 단순한 `Key-value` 구조
    - `In-memory` 데이터 저장소(`RAM`)
    - 빠른 성능 제공 : `평균 작업속도 < 1 ms` , `초당 수백만 건의 작업 가능`
        - *지연 시간 ↓, 처리량 ↑*
    - 서버 재시작 시 모든 데이터 유실
- **캐싱 전략 ( Caching Stategies )**
    - 캐싱 전략은 데이터의 유형과 데이터에 엑세스 패턴을 고려하여 선택
    - **Look-Aside (Lazy Loading)**
        - 레디스를 cache 사용할 때 가장 많이 사용하는 방법
        - `Application`은 데이터를 `cache`에 확인 후 데이터를 가지고 오는 작업 반복
        - 데이터가 존재하지 않을 경우 데이터베이스에서 데이터를 추출 후 `cache`에 저장
        - `Redis` 서버가 다운되어도 데이터를 원할하게 추출함
            - 데이터의 많은 `connection` 이 몰림
            - `Redis` 에 `Cache Miss` 가 발생해 성능 저하를 일으킴.
        - `DB` 에서 `cache`에 데이터를 미리 저장하는 방법 : `cache warming`
    - **Write-Around**
        - `DB`에만 데이터 저장 후 `cache miss`가 발생한 경우에만 `cache`의 데이터 추출
            - `cache`의 데이터와 `DB`의 데이터 정합성이 다를 수 있음
    - **Write-Thorugh**
        - `DB`, `cache` 모두 데이터 저장, 상대적으로 느린 저장 시간, 리소스 낭비
            - `expire time` 설정으로 문제 해결

## Redis 데이터 타입 활용

<img width="700" alt="레디스" src="https://user-images.githubusercontent.com/106054507/190864311-023a8dd8-53df-4f79-970b-48aaa8a1da01.PNG">

- **Strings** : `set command`를 통해 `put`한 데이터는 모두 `String`으로 저장
- **Bitmaps** : `bit` 단위의 연산 제공
- **Lists** : 데이터를 순서대로 저장하기 떄문에 큐로 사용하기 용의
- **Hashes** : 하나의 쌍에 여러개의 필드와 밸류 쌍으로 데이터를 저장
- **Sets** : 중복되지 않는 문자열의 집합
- **Sorted Sets** : 중복되지 않는 값을 저장하지만, 모든 값은 `Score`라는 숫자값으로 저장 및 정렬
- **HyperLogLogs** : 많은 값 및 중복되지 않는 값의 개수를 저장시 용의
- **Streams** : 로그 저장시 가장 용의한 자료구조

- **Best Practice - Counting**
    - **Strings**
        - 단순 증감 연산
        - *INCR / INCRBY / INCRBTFLOAT / HINCRBY / HINCRBYFLOAT / ZINCRBY*
    - **Bits**
        - 데이터 저장공간 절약
        - 정수로 된 데이터만 카운팅 가능
    - **HyperLogLogs**
        - 대용량 데이터를 카운팅 할 때 적절(오차 0.81%)
        - `set`과 비슷하지만 저장되는 용량은 매우 작음(12KB 고정)
        - 저장된 데이터 확인 불다.
- **Best Practice - Messaging**
    - **Lists**
        - `Blocking` 기능을 이용한 `Event Queue`로 사용
        - 키가 있을 때만 데이터 저장 가능 - *LPUSHX / RPUSHX*
            - 이미 캐싱되어 있는 피드에만 신규 트윗 저장
    - **Streams**
        - `append-only`
        - *시간 범위로 검색  / 신규 추가 데이터 수신 / 소비자별 다른 데이터 수신*

## Redis 데이터 영구 저장 (RDB vs AOF)

- Redis는  In-memory 데이터 스토어, 서버 재시작 시 모든 데이터 유실
- 복제 기능을 사용해도 사람의 실수 발생 시 데이터 복원 불가
- 캐시 이외의 용도를 사용한다면 데이터 백업이 필요함

- **Redis Persistence Option**
    - **RDB - snapshot**
        - 저장 당시 메모리에 있는 데이터 그대로 파일로 저장
        - 바이너리 파일 형태로 저장
    
    ```
    자동 : redis.conf 파일에서 **SAVE** 옵션
    수동 : BGSAVE 커맨드를 이용해 cli 창에서 수동으로 RDB 파일 저장
    ```
    
    - **AOF - Append Only File**
        - 데이터를 변경하는 커맨드가 실행시 모든 커맨드 저장
        - Redis 프로토콜 형태로 저장
    
    ```
    자동 : redis.conf 파일에서 auto-aof-rewrite-percentage 옵션
    수동 : BGREWRITWAOF 커맨드를 이용해 cli 창에서 수동으로 AOF 파일 재작성
    ```
    
    - **RRB vs AOF 선택 기준**
        - 백업을 필요하지만 어느 정도의 데이터 손실이 발생해도 괜찮은 경우
            - `RDB` 단독 사용
        - 장애 상황 직전까지의 모든 데이터가 보장되어야 할 경우
            - `AOF` 사용
        - 제일 강력한 내구성이 필요한 경우
            - `RDB & AOF` 동시 사용

## Redis 아키텍처 선택 노하우(Replication vs Sentinel vs Cluster)

<img width="700" alt="아키텍처" src="https://user-images.githubusercontent.com/106054507/190864320-577905d1-41cb-4675-9067-48d9b4349d1d.PNG">

- **Replication (복제 구성)**
    - `Master`, `Replica` 만 존재하는 간단한 구조
    - 단순한 복제 연결
        - `replicaof command`를 이용해 간단하게 복제 연결
        - 비동기식 복제
        - `HA` 기능이 없으므로 장애 상황 시 수동 복구
            - `replicaof no one`
            - 애플리케이션에서 연결 정보 변경
- **Sentinel**
    - `Master`, `Replica` 외의 추가의 `Sentinel` 노드가 필요
    - `Sentinel`을 일반 노드를 모니터링하는 역할 수행
    - `Master` 가 비정상 상태일 때 자동으로 페일오버
    - 연결 정보 변경 필요 없음
    - `sentinel` 노드는 항상 3대 이상의 홀수로 존재해야 함
        - 과반수 이상의 `sentinel` 동의 시 페일오버 진행
- **Cluster**
    - 최소 3개의 `Master` 노드 사용 및 샤딩 기능 제공
    - **스케일 아웃과 HA (High Avaliability)**
        - 키를 여러 노드에 자동으로 분할해서 저장 ( 샤딩 )
        - 모든 노드가 서로를 감시하여, `Mater`가 비정상 상태일 때 자동 페일오버
    

- **아키텍처 선택 기준**



<img width="700" alt="아키텍처 선택 기준" src="https://user-images.githubusercontent.com/106054507/190864328-c5166de2-7ec8-424d-8541-37fd630c588b.PNG">


## Redis 운영 Tips + 장애 포인트

- **Redis는 Single Thread로 동작**
    - 한 요청이 오랜 처리시간을 가질 경우 대기 요청 늦어짐. ( 장애 가능성 ↑ )
    - `keys *` → `scan` : 재귀적으로 키 값 추출
    - `Hash`나 `Sorted Set`등의 자료구조
        - 키 나누기 ( 최대 100만개 )
        - `hgetall` → `hscan`
        - `del` → `unlink` : 백그라운드로 요청 실행
- **변경시 장애를 막을 수 있는 기본 설정 값**
    - **STOP-WRITES-ON-BGSAVE-ERROR = NO**
        - `yes(default)`
        - `RDB` 파일 저장 실패시 `redis`로의 모든 `write` 불가능
    - **MAXMEMORY-POLICY = ALLKEYS-LRU**
        - `redis`를 캐시로 사용할 때 `Expire Time` 설정 권장
        - 메모리가 가득 찾을 때 `MAXMEMORY-POLICY` 정책에 의해 키 관리
            - `noeviction(default) : Delete none`
            - `volatile-lru`
            - `allkeys-lru`
- **Cache Stamped  TTL 값을 너무 작게 설정한 경우**
    - `Key Expires`가 실행될때 많은 `request`가 이 `Key`를 엑세스할때에 `Duplicate Write`, `Duplicate Read`가 발생 ( 장애 가능성 ↑ )
- **MaxMemory 값 설정 : Persistence / 복제 사용시 MaxMemory 설정 주의**
    - `RDB` 저장 & `AOF` `rewrite` 시 `fork()`
    - `Copy-on-Write`로 인해 메모리를 두배로 사용하는 경우 발생
    - `Persistence` / 복제 사용시 `MaxMemory`는 실제 메모리의 절반으로 설정
