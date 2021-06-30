# DBMS 와 버퍼 

***

데이터를 메모리에서 관리한다면 디스크에 접근하지 않아도 되고 성능도 빨라진다. 

그러므로 메모리에 어떤 데이터를 적재하느냐의 전략에 따라서 성능이 결정되기도 한다. 

메모리에 자주 접근하는 데이터를 올려놓는다면 디스크 I/O 가 줄어들고 메모리에서 바로 가지고 올 수 있다.

이런 역할을 하는걸 캐싱이라고도 한다. 

이러한 데이터 관리를 어떻게? 언제까지? 관리해주는 것이 버퍼 매니저가 하는 역할이다. 

#### 메모리 위에 있는 두 개의 버퍼 

메모리 위에는 사용 목적에 따라 다른 두 개의 버퍼가 있다. 

- 데이터 캐시 

  - 데이터 캐시는 디스크에 있는 데이터를 메모리에 올려놔서 이 메모리에 있는 데이터를 원하는 요청이 오면 디스크에 접근하지 않고 
  메모리에서 반환하도록 한다. 이를 통해서 성능을 높일 수 있다.  
  
  - MySQL InnoDB 에서는 buffer pool 이라고 하며 매개변수는 `innodb_buffer_pool_size` 라고 한다. 
  
  - 초기값은 128MB 이다.
  
  - 설정 값을 확인하기 위해서는 `SHOW VARIABLES LIKE 'innodb_buffer_pool_size'` 라고 하면 된. 

- 로그 버퍼 

  - 데이터베이스는 write-back 방식을 통해서 데이터 갱신을 지원한다. 그래서 갱신 요청이 오면 바로 업데이트 하는게 아니라 
  로그 버퍼에 쌓아놓고 다차면 업데이트를 비동기식으로 한다.
  
    - 이 경우에 문제점이 로그 버퍼에 있는 내용이 디스크에 저장되기전에 장애가 난다면 데이터 갱신이 일어나지 않는다는 문제가 생긴다. 
    그래서 이걸 해결하기 위해서 매번 커밋 시점에 갱신 정보를 담고 있는 로그 파일을 하나 쓴다. 이 로그 파일은 메모리에 저장되지 않고 영속적인 
    저장소에 쓰는 것이다. 오로지 append-only 로 로그 파일을 쓴다면 성능 상에 문제는 없다.  
    
  - MySQL InnoDB 에서는 log buffer 라고 하며 매개변수는 `innodb_log_buffer_size` 이다.
  
  - 초기값은 8MB 이다.
  
  - 설정 값을 확인하기 위해서는 `SHOW VARIABLES LIKE 'innodb_log_buffer_size'` 이다.  
  
이러한 버퍼는 사용자의 요구에 따라 사이즈를 변경하는게 가능하다.  

#### 데이터 캐시와 로그 버퍼 튜닝

이건 각 어플리케이션 마다 다를텐데 일반적으로는 데이터 조회 요청이 데이터 업데이트 요청보다 더 많다. 

하지만 만약 데이터 갱신 요청이 더 많다면 로그 버퍼의 사이즈를 키우는 걸 추천한다. 

#### 추가적인 메모리 영역 워킹 메모리
 
DBMS 는 로그 버퍼와 데이터 캐시 말고도 워킹 메모리(working memory) 라는 추가적인 메모리 영역을 가지고 있다.

이 메모리는 임시 메모리로 데이터를 정렬 하거나 해시 테이블로 사용할 때 사용되는 메모리다. 

정렬 같은 경우는 order_by 나 group_by, 집합 연산, 윈도우 함수 등을 사용할 때 사용된다. 

MySQL 같은 경우는 이 영역을 `sort_buffer_size` 라고 부른다. 

이 영역의 사이즈가 중요한 이유는 데이터 정렬을 할 때 이 부분의 영역이 부족하면 저장소의 스왑 영역을 사용하는데 

이는 디스크 기반의 영역이므로 성능이 나오지 않는다. 

주로 하나의 SQL 문을 실행할때는 메모리 부족이 일어나는 경우는 거의 없다. 이 경우를 재현하려면 여러개의 SQL 문을 실행하고
이게 잘 작동하는지 알아봐야 한다.     