## NL 조인

조인의 기본 전략은 NL 조인이다. 

Index 를 이해했다면 NL 조인은 어렵지 않다. NL 조인에서 인덱스를 사용하지 않는다면 성능 상에 이슈가 있기 때문에 거의 인덱스를 무조건 사용하기 때문이다. 

그리고 NL 조인을 이해하고 나면 뒤에서 설명할 Hash Join 과 Sort Merge Join 도 그렇게 어렵지 않다. 조인 프로세싱 자체는 다르지 않기 때문이다. 다만 어떤 자료구조를 사용하느냐의 차이는 있다. 

그럼 지금부터 NL 조인에 대해서 알아보자. 

***

### 1. 기본 매커니즘 

예시로 바로 설명하겠다.

사원 테이블과 고객 테이블이 있고 1996 년 1월 1일 이후에 입사한 사원으로 사원이 관리하는 고객 데이터를 추출하는 쿼리를 만든다고 해보자. 

아래 SQL 문을 작성하면 결과를 쉽게 만들 수 있다. 

```sql
SELECT e.사원명, c.고객명, c.전화번호
FROM 사원 e, 고객 c 
WHERE e.입사일자 >= '19960101'
AND c.관리사원번호 = e.사원번호 
``` 

사원 테이블에 있는 사원번호를 기반으로 조인을 수행한다. NL 조인이 사용하는 수행방식은 간단한데 사원 테이블에서 조건에 맞는 레코드를 찾아서 일일히 고객 테이블로 가서 매칭이 되는 레코드를 찾는 방식이다.

이런 방식이 두 개의 중첩문이 있는 구조랑 비슷하다고 해서 NL (Nested Loop) 조인 이라고 한다.

그러니까 NL 조인을 for 문으로 풀면 다음과 같을 것이다.

```java
for (int i=0;i<100;i++) {
    for (int j=0;j<100;j++) {
        // Do anything
    }
}
```

일반적으로 NL 조인은 외각에 있는 Outer Table 과 안쪽에 있는 테이블인 Inner Table 모두에서 인덱스를 사용한다.

여기서 중요한 건 Outer Table 에서는 한번만 딱 결과집합을 만들면 되므로 사이즈가 크지 않다면 인덱스를 굳이 걸지 않아도 된다.

하지만 Inner Table 에서는 Outer Table 에 있는 레코드 하나당 결과 집합을 찾기 위해 조회를 해야하므로 인덱스가 없다면 그만큼 테이블 풀 스캔을 해야한다. 

이러한 이유로 Inner Table 에서는 인덱스를 이용하도록 하는게 좋다.

그러면 이제 세세하게 실재 조인의 수행과정을 보자. SQL 문은 이전과 같고 사원 테이블에 입사일자에 인덱스가 걸려있고 고객 테이블에 관리사원번호도 인덱스가 걸려있다고 가정해보자. 

```sql 
SELECT e.사원명, c.고객명, c.전화번호
FROM 사원 e, 고객 c 
WHERE e.입사일자 >= '19960101'
AND c.관리사원번호 = e.사원번호 
```

처리 과정은 다음과 같다. 

1. 사원 테이블에서 인덱스를 이용해 입사일자 >= 19960101 인 첫 레코드르 찾는다. 

2. 인덱스에서 읽은 ROWID 를 바탕으로 사원 테이블에 엑세스한다. 

3. 사원 테이블에서 읽은 사원번호를 바탕으로 고객 인덱스를 탐색한다. 

4. 고객 인덱스에서 읽은 ROWID 를 바탕으로 고객 테이블에 엑세스한다. 

5. 고객 인덱스에서 조건에 맞는 값이 더 있는지 스캔한다. 있다면 테이블에 엑세스해서 결과 집합을 만들고 없다면 스캔을 종료한다.   

6. 한 조인이 끝났으므로 다음 조인을 재개하기 위해서 사원 인덱스에서 조건에 맞는 다음 레코드를 찾는다. 

7. Outer Table 에서 찾은 레코드 수를 바탕으로 1 ~ 6 까지의 과정을 반복한다. 

***

### 2. NL 조인 실행계획 제어 

아래는 NL 조인의 실행 계획이다. 상대적으로 위쪽에 있는 사원 테이블을 기준으로 고객 테이블과 NL 조인을 한다고 해석하면 된다.

각 테이블에 엑세스 할 때 인덱스도 사용한다는 사실을 실행계획에서 찾을 수 있다. 

```text
Execution Plan
--------------------------------------------
SELECT STATEMENT Optimizer = ALL_ROWS
    NESTED LOOPS
        TABLE ACCESS BY INDEX ROWID OF '사원' (TABLE)
            INDEX RANGE SCAN OF '사원_X1' (INDEX)
        TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
            INDEX RANGE SCAN OF '고객_X1' (INDEX) 
```

NL 조인을 제어할 땐 아래와 같이 use_nl 힌트를 사용하면 된다. 

```sql
SELECT /*+ ordered use_nl(c) */ 
    e.사원명, c.고객명, c.전화번호
FROM 사원 e, 고객 c 
WHERE e.입사일자 >= '19960101'
AND c.관리사원번호 = e.사원번호 
```

ordered 힌트는 FROM 절에서 기술한 순서대로 조인을 수행하라는 뜻이며 use_nl 은 NL 방식으로 해당 테이블을 조인 하라는 뜻이다. 

세 개 이상의 테이블을 조인할 땐 힌트를 아래처럼 적으면 된다. 

```sql
SELECT /*+ ordered use_nl(B) use_nl(C) use_hash(D) */ 
    * 
FROM A, B, C, D
```

여기서 use_hash 힌트는 Hash Join 을 수행하라는 뜻이다. 

그리고 ordered 대신에 leading 이라는 힌트를 넣을 수 있는데 이는 FROM 절에 종속되지 않고도 순서를 제어하는 방식이다. 아래와 같이 사용한다. 

```sql
SELECT /*+ leading(C, A, B, D) use_nl(A) use_nl(B) use_hash(D) */ 
    * 
FROM A, B, C, D
```

***

### 3. NL 조인 수행 과정 분석 

간단한 조건절을 통해서 NL 조인이 어떻게 수행하는지를 봤는데 좀 더 자세한 NL 조인 수행 방식을 보기 위해서 아래와 같이 조건절을 추가해보자. 

```sql
SELECT /*+ ordered use_nl(c) index(e) index(c) */ 
    e.사원명, e.사원번호, e.입사일자 
    c.고객명, c.전화번호, c.최종주문금액
FROM 사원 e, 고객 c 
WHERE e.입사일자 >= '19960101'
AND c.관리사원번호 = e.사원번호 
AND e.부서코드 = '2123'
AND c.최종주문금액 >= 20000
```

인덱스 구성은 다음과 같다. 

```text
사원_PK : 사원번호
사원_X1 : 입사일자
고객_PK : 고객번호
고객_X1 : 관리사원번호
고객_X2 : 최종주문금액  
```

고객 테이블로 조인을 수행하도록 힌트를 줬고 사원 테이블과 고객 테이블 모두 인덱스르 사용하도록 힌트를 줬다. 

실행 계획과 사용할 인덱스는 다음과 같다. 

```text
Execution Plan
--------------------------------------------
SELECT STATEMENT Optimizer = ALL_ROWS
    NESTED LOOPS
        TABLE ACCESS BY INDEX ROWID OF '사원' (TABLE)
            INDEX RANGE SCAN OF '사원_X1' (INDEX)
        TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
            INDEX RANGE SCAN OF '고객_X1' (INDEX)
```

처리 과정은 다음과 같다. 

1. 입사일자 조건을 만족시키기 위해 사원_X1 인덱스에서 Range Scan 을 수행한다. 

2. 사원_X1 에서 읽은 ROWID 를 바탕으로 사원 테이블에 엑세스한다. 그 후 부서코드를 기반으로 필터링 작업을 수행한다. 

3. 사원 테이블에서 찾은 레코드에 있는 사원 번호를 바탕으로 고객_X1 인덱스에서 Range Scan 을 수행한다. 

4. 고객_X1 인덱스에서 읽은 ROWID 값을 바탕으로 고객 테이블에 엑세스한다. 

5. 고객 테이블에서 최종주문금액을 바탕으로 필터링 처리해서 최종 결과 집합에 포함시킨다. 

여기서 주의할 점은 각 단계별 작업을 모두 끝내고 다음 단계로 넘어가는 워터폴 방식이 아니라 하나하나 레코드 별로 1 ~ 5번 작업을 수행한다는 것이다. 

***

### 4. NL 조인 튜닝 포인트 

위의 SQL 문을 튜닝한다고 해보자. 먼저 사원_X1 인덱스를 읽고 테이블에 엑세스 하는데 단일 칼럼 같은 경우는 범위 검색 조건이라도 상관 없으므로 인덱스 스캔에서 비효율이 발생하지는 않았다. 

하지만 이 조건 하나로 테이블 랜덤 엑세스를 했으니 테이블 랜덤 엑세스 비율이 많고 '부서코드' 기반의 필터가 많다면 그만큼 비효율적이다는 의미니 부서코드를 인덱스에 추가하는 방법이 있다.

두 번째 튜닝 포인트는 고객_X1 인덱스를 탐색하는 부분이다. 고객_X1 인덱스를 탐색 수는 사원 테이블에서 '부서코드' 필터링을 거치고 난 후 레코드 수를 말한다. 

만약 사원 테이블 필터링을 거친 후 레코드 수가 10만 개이고 고객_X1 인덱스 Depth 가 3 이라면 30만개의 레코드를 찾아봤다는 의미다. 조인의 수가 너무 많다면 조인 순서를 바꾸던지 다른 조인 방식을 사용는 방법을 고려해볼 수 있다.  

세 번째 튜닝 포인트는 고객_X1 인덱스를 읽고나서 고객 테이블로 엑세스하는 과정이다. 여기서도 고객 테이블 엑세스 비율이 높고 '최종주문금액' 을 가지고 필터링 하는 비율도 높다면 고객_X1 인덱스에 '최종주문금액' 을 인덱스에 추가하는 과정을 고려해볼 수 있다. 

마지막으로 알고 있어야 하는 사실은 조인의 과정은 조인의 시작 테이블에서 얻은 결과 건수에 의해 전체 조인의 량이 결정된다는 사실이다.

사원_X1 인덱스를 스캔하면서 사원 테이블로 가서 추출한 레코드의 수가 많다면 그만큼 고객 테이블에서 조인을 하는 수가 많다라는 사실이다.

#### 조인의 순서

앞서 말한 조인의 시작하는 테이블에서 추출한 레코드의 수에 따라서 조인 횟수에 영향을 준다는 사실을 알았다. 

좀 더 자세하게 비교를 해보자 두 개의 테이블이 있고 OneToMany 관계를 이룬다고 했을 때 (팀과 멤버) 팀을 가지고 조인을 하는게 나을지 멤버를 가지고 조인을 하는게 나을지 한번 생각해보자. 

먼저 멤버(Many) 를 가지고 조인을 한다고 했을 때 멤버의 수만큼 멤버 테이블로 엑세스를 할 것이고 그 수 만큼 팀 인덱스를 거쳐서 팀 테이블로 엑세스 할 것이다. 

팀을 가지고 멤버로 조인을 한다고 하면 팀 테이블로 가서 멤버 인덱스로 가서 멤버 테이블로 실제적으로 엑세스를 할 것이다.

둘이 비교를 하자면 팀 으로 조인을 시작하는 경우에는 뒷 과정인 멤버 테이블로 엑세스하는 수가 많을 것이다. 아무래도 멤버가 많을것이니. 

멤버로 조인을 시작하는 경우에는 멤버 테이블로 엑세스 하는 과정과 팀 인덱스를 스캔하는 과정 팀 테이블로 엑세스 하는 과정 모두가 많을 것이다.

그러므로 어떤 테이블로 조인을 시작하느냐에 따라서 성능 차이는 있을 수 있다. 

만약 조인의 순서를 바꿔도 그렇게 큰 차이가 없다면 다른 조인 방법 (Hash Join, Sort Merge join) 같은 걸 이용해보자.


***

### 5. NL 조인 특징 요약 

NL 조인의 첫 번째 특징은 테이블 랜덤 엑세스 위주의 조인 방식이라는 점이다. 레코드 하나를 읽기 위해서 블록을 통째로 읽어야 하는 방식 때문에 설령 메모리에서 버퍼 캐시에 히트를 한다고 하더라도 비효율은 분명 존재한다. 

그래서 인덱스 구성이 아무리 완벽하더라도 대용량 데이터를 NL 조인한다고 하면 성능이 나오진 않을 것이다.

두 번째 특징은 조인은 한 레코드씩 순차적으로 진행된다는 점이다. 이 말은 부분 처리가 NL 조인으로 가능하다는 뜻으로 매우 빠른 응답속도를 내는게 가능하다. 

마지막으로 인덱스 구성 전략이 다른 조인들 보다 확실히 중요하다. 일단 조인 칼럼에 인덱스가 있느냐 없느냐 부터 성능에 영향을 많이 미치고 INNER 테이블에 있는 인덱스는 특히나 더 중요하다.

종합적으로 이러한 요소들을 고려했을 때 NL 조인은 소량의 데이터를 처리하거나 부분 처리가 가능한 OLTP 환경에서의 트랜잭션 처리에 좋다. 

***

### 6. NL 조인 튜닝 실습

실제로 이제 예제를 가지고 튜닝을 해보자.

```sql
SELECT /*+ ordered use_nl(c) index(e) index(c) */ 
    e.사원명, e.사원번호, e.입사일자 
    c.고객명, c.전화번호, c.최종주문금액
FROM 사원 e, 고객 c 
WHERE e.입사일자 >= '19960101'
AND c.관리사원번호 = e.사원번호 
AND e.부서코드 = '2123'
AND c.최종주문금액 >= 20000
```

여기서 SQL 트레이스 결과가 다음과 같이 나왔다고 한다. 

```text
Rows    Row Source Operation
5       Nested Loops
3           TABLE ACCESS BY INDEX ROWID OF 사원
2780            INDEX RANGE SCAN OF 사원_X1
5           TABLE ACCESS BY INDEX ROWID OF 고객 
8               INDEX RANGE SCAN OF 고객_X1
```

사원_X1 에서 찾은 레코드 수는 2780 개인데도 불구하고 실제 테이블에 엑세스해서 가지고 온 데이터는 3개에 불과하다. 

이는 불필요한 테이블 랜덤 엑세스가 일어난다는 뜻이므로 인덱스에 테이블 필터 조건을 추가하는게 좋아보인다. 

사원_X1 인덱스에 필요한 칼럼을 추가하고 트레이스를 다시 보니 다음과 같다. 

```text
Rows    Row Source Operation
5       Nested Loops
3           TABLE ACCESS BY INDEX ROWID OF 사원
3               INDEX RANGE SCAN OF 사원_X1
5           TABLE ACCESS BY INDEX ROWID OF 고객 
8               INDEX RANGE SCAN OF 고객_X1
```

이러면 이제 튜닝은 끝난 것일까? 불필요한 테이블 엑세스 수는 많이 줄었지만 인덱스 스캔도 효율적으로 하고 있는지는 아직 확인하지 않았다. 

그러므로 트레이스에서 각 단계의 처리 건수(Processing count) 를 확인하는게 중요하다.

이를 한번 확인해보자. 

```text
Rows    Row Source Operation
5       Nested Loops (cr=112 pr=34 pw=0 time=122 us)
3           TABLE ACCESS BY INDEX ROWID OF 사원 (cr=105 pr=32 pw=0 time=118 us)
3               INDEX RANGE SCAN OF 사원_X1 (cr=102 pr=31 pw=0 time=16 us)
5           TABLE ACCESS BY INDEX ROWID OF 고객 (cr=7 pr=2 pw=0 time=4 us)
8               INDEX RANGE SCAN OF 고객_X1 (cr=5 pr=1 pw=0 time=0 us)
```

cr 은 블록의 논리적 요청 횟수, pr 은 디스크에서 읽은 블록의 개수, pw 는 디스크에 저장한 블록 수 이다.

한 블록에 평균 레코드가 500개 정도 있다고 잡는다면 사원 인덱스에서 탐색할 때 50,000 개의 레코드를 읽은 후 3개의 결과를 추출했다는 뜻이다. 

그러므로 사원_X1 인덱스 스캔의 효율성을 위한 튜닝이 조금 필요하다. 

이번에는 트레이스 결과가 다음과 같다고 했을 때 병목지점은 어디일까?  

```text
Rows    Row Source Operation
5       Nested Loops (cr=112 pr=34 pw=0 time=122 us)
2780        TABLE ACCESS BY INDEX ROWID OF 사원 (cr=166 pr=2 pw=0 time=... us)
2780            INDEX RANGE SCAN OF 사원_X1 (cr=4 pr=0 pw=0 time=... us)
5           TABLE ACCESS BY INDEX ROWID OF 고객 (cr=2566 pr=3 pw=0 time=... us)
8               INDEX RANGE SCAN OF 고객_X1 (cr=2558 pr=383 pw=0 time=... us)
```

이 결과에서는 사원 테이블에서 불필요한 Index Range Scan 을 하지도 않고 테이블 엑세스 필터도 없다. 

그치만 이제 고객 테이블에서 너무나 많은 블록을 읽어오지만 실제로 얻은 결과의 수는 적다. 

이런 경우에는 조인 순서를 뒤집어 보는 걸 고려해볼 수 있다. 조인 순서를 뒤집어봐도 첫 번째 테이블에서 읽은 결과가 많을 수도 있다. 

왜냐하면 지금 현재 이 고객 테이블은 사원 테이블로 인해 어느정도 필터가 된 집합들이기 떄문에 적은 것일수도 있어서이다. 

조인 순서를 바꿔도 효과가 없다면 다른 조인 방법을 생각해보자. 

***

### 7. NL 조인 확장 매커니즘

버전이 올라가면 NL 조인의 성능을 개선하기 위해서 나온 방식은 테이블 Prefetch, 배치 I/O 등이 있다. 

이 방식들은 디스크 I/O 의 성능을 개선하기 위한 방법인데 개별적으로 I/O 를 하는 것보다 I/O 를 한번에 많이 하는 방식이다.

테이블 Prefetch 는 I/O 가 발생할 때 다음 블록들까지 그냥 미리가지고 오는 걸 말하며 배치 I/O 는 I/O 가 필요할 때 마다 모아놨다가 임계치를 충족한다면 그때 I/O 를 시전하는 방법이다.

이들을 실행 계획을 보면 조금 다르다. 

#### 전통적인 실행계획

오라클 데이터베이스 기준으로 NL 조인을 표현하기 위해서 전통적으로 사용하는 방식이다. 

```text
NESTED LOOPS
    TABLE ACCESS BY INDEX ROWID OF '사원' (TABLE)
        INDEX RANGE SCAN OF '사원_X1' (INDEX)
    TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
        INDEX RANGE SCAN OF '고객_X1' (INDEX)
```

#### 테이블 Prefetch 실행계획

Inner 테이블 쪽에서 테이블 Prefetch 가 발생한 실행계획이다. 

```text
TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
    NESTED LOOPS
        TABLE ACCESS BY INDEX ROWID OF '사원' (TABLE)
            INDEX RANGE SCAN OF '사원_X1' (INDEX)
        INDEX RANGE SCAN OF '고객_X1' (INDEX)
```

#### 배치 I/O 실행계획    

Inner 테이블 쪽에서 배치 I/O 가 발생한 실행게획이다. 

nlj_batching, no_nlj_batching 힌트를 이용해서 이 실행계획이 나오게 또는 안나오게 할 수도 있다.  

```text
NESTED LOOPS
    NESTED LOOPS
        TABLE ACCESS BY INDEX ROWID OF '사원' (TABLE)
            INDEX RANGE SCAN OF '사원_X1' (INDEX)
        INDEX RANGE SCAN OF '고객_X1' (INDEX)
    TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
```