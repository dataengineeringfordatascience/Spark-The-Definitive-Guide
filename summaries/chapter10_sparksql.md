spark sql 이란?
(정의)

- 데이터에 대해 crud작업 할 수 있는 언어
- 대용량 데이터에서 sql작업을 할 수 있도록 지원하는 spark기반의 쿼리 언어

(특징)

- Hive와 연동이 된다. 메타스토어에 저장된 메타데이터를 사용가능.(hive의 metastore  = spark의 catalog)
- tableau와 연동이 된다. BI시각화에 필요한 작업을 spark에서하고 바로 결과물 시각화 편리(thrift jdbc/odbc server 덕분에)

(사용법)

- command line 사용가능
- spark sql interface 사용가능 : sql을 통해 추출한 데이터를 바로 데이터프레임으로 받을 수 있어 좋다.

(spark sql의 작업영역) 의 구성요소

1. 카탈로그 (hive의 metastore  = spark의 catalog)
2. 데이터베이스

   데이테베이스 리스트 확인하기
   데이터베이스 신규생성하기
   사용할 데이터베이스 정하기
   현재 사용중인 데이터베이스 보기
   현재위치에서 다른 데이터베이스 사용하기

3. 뷰

   데이터베이스에 등록됨.
   -임시뷰
   현재세션에서만 사용가능하며, 데이터베이스에 등록되지 않음.(**임시테이블 지원안하기 때문에 뷰를 많이 사용함)
   -글로벌 임시뷰
   데이터베이스에 상관없이 사용가능, 세션종료전까지 사용가능

*explain ? 쿼리 최적화를 위한 쿼리성능 체크!

```https://snowple.tistory.com/377```

explain table_name
=describe table_name
=show columns from table_name

- 예제: 업뎃/삽입/삭제
  explain select user_name from a
  <==> update a set user_name ='cb' (컬럼 select으로 쿼리전환하기)

- 예제: 조인
  explain select a.user_name, b.product_name 
  from a join b 
  on a.user_id = b.user_id

- 결과물
  쿼리결과의 1행을 출력하는데 사용된 테이블에 관한 정보를 출력한다.

  <결과 테이블의 컬럼들>

  - select_type : 사용된 테이블을 선택한 방법 : simple(union, subquery없을 때),primary(first and most outer), subquery(subquery in select), derived(subquery in from), union(not primary and union), union_result(result)
  - type : 행 스캔방식 : all(행순서)/index(인덱스순서)/range(제한인덱스)/ref(비식별자 매칭)/eq_ref(식별자 매칭)/const(상수조건)/system(컬럼1개인 테이블)
  - possible_key : 자동으로 선택한 인덱스
  - key : 사용한 키(사용한 인덱스)
  - key_len : 인덱스 필드의 최대가능 길이

4. 테이블

[metadata]

- managed / unmanaged table
  : depends on having spark manage the metadata for a set of files as well for the data !
- describe / refresh  

[data]

- internal / external ?

  : external 테이블은 DB 외부에 저장된 data 파일을 조작하기 위한 접근 방법의 하나로 **읽기 전용 테이블**이다. external 테이블의 실제 데이터는 DB 외부에 존재하지만, external 테이블에 대한 **metadata는 DB 내부에 존재**하는 일종의 가상 테이블이다.

  출처: <https://studyharo.tistory.com/15> [하로의 공부방]

- drop/cache  ?

  : Cache the contents of the table in memory using the RDD cache. This enables subsequent queries to avoid scanning the original files as much as possible

  : caching the tables put the whole table in memory compressed if you use this setting(**spark**.sql.inMemoryColumnarStorage.compressed = true). When doing caching on a DataFrame it is Lazy caching which means **it will only cache what rows are used in the next processing event**. So if you do a query on that DataFrame and only scan 100 rows, those will only be cached, not the entire table. If you do CACHE TABLE MyTableName in SQL though, it is defaulted to be eager caching and will cache the entire table.

  +table cache : http://small-dbtalk.blogspot.com/2013/09/mysql-table-cache.html

5.Standard SQL 기본제공 기능

- complex types : Struct, List
- subquries [서브쿼리] https://ttend.tistory.com/624 

: correlated / uncorrelated ?
  : scalar / predicate?

  

  서브쿼리란?  내부쿼리 !

- 쿼리안에 포함된 쿼리

- SELECT,ORDER BY절 위치: 스칼라 서브쿼리

  *데이터건수가 적을때, 조인보다 빠름. 
  *반드시 하나의 결과만 리턴
  *where문을 뽑아서 새로운 select문을  만들면 끝

select a.id () from a

- FROM 절 위치: 인라인 뷰

- WHERE 절 위치 : 서브쿼리

  

  *상관서브쿼리란? 조인같음…조건 2개씩 걸어도 됨.

  : 내부쿼리가 외부쿼리의 컬럼을 이용한 결과를 외부에 반출한다

  

  (순서반복)

  외부 쿼리의 행을 가져온다.

  이를 이용해 서브쿼리를 수행한다.(동일 테이블을 **중복사용**가능)

  서브쿼리의 결과값으로 외부쿼리의 행 선택여부를 결정한다.

  

  *predicate란?
  : a logical condition being applied to rows in a table. SQL Predicates are found on the tail end of clauses, functions, and SQL expressions in existing query statements. It is an expression that evaluates to **TRUE**, **FALSE**, or **UNKNOWN**. 

  

  **How to filter 'unique and used' value of a column** :

  ​	select a.name from a 

  ​	where exist (select 1 from b where id =a.id)

  **How to deal with null value** : 

  ​	nvl (colname, default value if colname is null)

  ​	nvl2 (colname, set value if not null, default value if colname is null)

  ​	coalesce(colname1, colname2 if colname1 is null, default value if both are null)

- fuctions(+user defined fuctions)
