# Chapter 9. Data Sources

스파크에서 다룰 수 있는 6가지 주요 데이터 소스

- CSV
- JSON
- Parquet
- ORC
- JDBC/ODBC connections
- Plain-text files

이외에도 Cassandra, HBase, MongoDB, AWS Redshift, XML 등등을 이용할 수 있음





### 1. Data Sources API의 구조

Data Sources API의 기본 구조는 Read, Write 등으로 나누어 볼 수 있음

#### (1) Read API 의 구조

```scala
// 데이터 읽기 예시
spark.read.format("csv")
.option("mode", "FAILFAST")
.option("inferSchema", "true")
.option("path", "path/to/file(s)")
.schema(someSchema)
.load()
```

- `format` : default = parquet
- `option`: 어떤 방식으로 데이터를 읽을 것인지 설정. key - value 구조로 넣음
- `schema`: 데이터의 스키마 
- `read mode` : 데이터를 다루다보면, 구조화되지 않은 데이터를 흔히 볼 수 있음(malformed data). 이런 데이터를 읽어들일 때 어떻게 읽어들여야 할지 스파크에게 알려줘야 함. read mode의 종류는 다음과 같음

![1554536640195](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554536640195.png)

#### (2) Write API의 구조

```scala
// 데이터 쓰기 예시 
dataframe.write.format("csv")
.option("mode", "OVERWRITE")
.option("dateFormat", "yyyy-MM-dd")
.option("path", "path/to/file(s)")
.save()
```

- format` : default = parquet
- `option`: 어떤 방식으로 데이터를 쓸 것인지 설정. key - value 구조로 넣음
  - `save mode` : 지정 경로에 데이터를 어떻게 쓸 것인가 설정. default = `errorIfExists`

![1554536905822](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554536905822.png)

- `PartitionBy, bucketBy, sortBy`: 파일 기반 데이터 소스에 대해서만 설정 가능.  



이제 개별 데이터 소스마다 특징을 알아보자



### 2. CSV Files

- csv 파일, 만만하게 보지 말고 주의하자

  > *CSV files, while seeming well structured, are actually one of the trickiest file formats you will encounter because not many assumptions can be made in production scenarios about what they contain or how they are structured.*

- 그래서 csv reader에는 많은 옵션을 주어서 문제를 방지하게끔 하고 있음

#### (1) Reading CSV Files

```scala
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}

//Schema를 설정해주자
val myManualSchema = new StructType(Array(
new StructField("DEST_COUNTRY_NAME", StringType, true),
new StructField("ORIGIN_COUNTRY_NAME", StringType, true),
new StructField("count", LongType, false)
))

//Schema에 따라 데이터를 불러오자 
spark.read.format("csv")
.option("header", "true")
.option("mode", "FAILFAST")
.schema(myManualSchema)
.load("/data/flight-data/csv/2010-summary.csv")
.show(5)

//schema를 잘못지정했다면, 데이터를 불러올 때 오류가 날 것(스키마 정의할 때가 아니라)
```



#### (2) Writing CSV Files

```python
# in Python
# csv 파일을 읽어서 tsv 파일로 써보자 
csvFile = spark.read.format("csv")\
.option("header", "true")\
.option("mode", "FAILFAST")\
.option("inferSchema", "true")\
.load("/data/flight-data/csv/2010-summary.csv")

csvFile.write.format("csv").mode("overwrite").option("sep", "\t")\
.save("/tmp/my-tsv-file.tsv")
```

```pseudocode
# 설정했던 디렉토리 안에 보면 파티션이 여럿으로 나뉘어져 있을 것
# 리파티션 하면 다시 파티션 갯수는 나뉘어 질 것. 
$ ls /tmp/my-tsv-file.tsv/
/tmp/my-tsv-file.tsv/part-00000-35cf9453-1943-4a8c-9c82-9f6ea9742b29.csv
```





### 3. JSON Files

- JavaScript Object Notation. 
- Spark 에서 얘기하는 JSON 파일은 *line-delimited JSON 파일*임. (large JSON object 또는 array per
  file이 아님!)
-  장점 1:  전체 데이터를 로드 하지 않고도 뒤에 새로운 레코드를 붙일 수 있음 

> *it allows you to append to a file with a new record (rather than having to read in an entire file and then write it out)*

- 장점2: JSON 오브젝트는 structure를 가지고 있으며, basic types이 설정되어 있음
- 스파크가 데이터를 이해하고 다루는 데 csv보다 더 도움이 되는 정보를 가지고 있음. 그래서 옵션도 csv 보다 적음

#### (1) Reading JSON Files

```python 
spark.read.format("json").option("mode", "FAILFAST")\
.option("inferSchema", "true")\
.load("/data/flight-data/json/2010-summary.json").show(5)
```



#### (2) Writing JSON Files

```python 
# in Python
csvFile.write.format("json").mode("overwrite").save("/tmp/my-json-file.json")
$ ls /tmp/my-json-file.json/
/tmp/my-json-file.json/part-00000-tid-543....json
```



### 4. Parquet Files

>  *open source column-oriented data store that provides a variety of storage optimizations, especially for analytics workloads.*

- 압축되어 있어 저장 공간 절약
- 전체 파일 읽지 않고 일부 칼럼만 읽어올 수 있음 
- 스파크에서 특히 잘 돌아감(디폴트 파일 포멧!)

- csv, json 보다 long-term storage 에 적합. 리딩하는 게 훨씬 효율적이어서.
- 옵션이 몇 가지 없음

![1554538676949](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554538676949.png)



#### (1) Reading Parquet Files

- 데이터 저장할 때 스키마도 같이 저장하시 때문에 읽을 때 따로 설정할 필요가 없음

> *the schema is built into the file itself (so no inference needed)*

```python 
# in Python
spark.read.format("parquet")\
.load("/data/flight-data/parquet/2010-summary.parquet").show(5)
```



#### (2) Writing Parquet Files

```python 
# in Python
csvFile.write.format("parquet").mode("overwrite")\
.save("/tmp/my-parquet-file.parquet")
```



### 5. ORC Files

> *self-describing, type-aware columnar file format designed for Hadoop workloads*.

- 스트리밍 데이터 읽기에 최적화 되어있음
- 옵션이 따로 없음. 스파크가 알아서 데이터를 잘 이해해서.
- 그럼 ORC와 Parquet 파일의 차이점은 ? 
  - ORC 는 Hive에 최적화
  - Parquet은 Spark 에 최적화!

#### 6. SQL Databases

- MySQL database, a PostgreSQL database, or an Oracle database 등에 연결 가능
- 책에서는 SQLite 에 연결
- "how you connect to the database"는 설정해줘야 함
- ***authentication*** 과 ***connectivity***를 고려해야 함. 스파크 클러스터들이 데이터베이스 시스템의 네트워크에 연결되어 있는지 확인해야 함 

![1554539336936](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554539336936.png)

- SQL 데이터베이스를 쓰려면? 

  > *you need to do two things: (1) the Java Database Connectivity (JDBC) driver for you particular database on the spark class path, and (2) provide the proper JAR for the driver itself.*



#### (1) Reading from SQL databases

```python
# in Python
driver = "org.sqlite.JDBC"
path = "/data/flight-data/jdbc/my-sqlite.db"
url = "jdbc:sqlite:" + path
tablename = "flight_info"
```

- 데이터를 가져오기 전에, 연결이 잘 되어있는지 확인해야 함. 아래 코드 활용 (SQLite에서는 필요 없지만, 일반 RDB에서 가져올때는 해줘야 함)

```scala
import java.sql.DriverManager
// 연결이 잘 되어있는지 확인 먼저 한다
val connection = DriverManager.getConnection(url)
connection.isClosed()
connection.close()

// 데이터를 불러온다 -- SQLite 버전
val dbDataFrame = spark.read.format("jdbc").option("url", url)
.option("dbtable", tablename).option("driver", driver).load()


// 데이터를 불러온다 -- PostgreSQL 버전
val pgDF = spark.read
.format("jdbc")
.option("driver", "org.postgresql.Driver")
.option("url", "jdbc:postgresql://database_server")
.option("dbtable", "schema.tablename")
.option("user", "username").option("password","my-secret-password").load()
```



- 스키마는 따로 지정할 필요가 없음

> *Spark gathers this information from the table itself and maps the types to Spark data types.*



?? 이해가 안가요

![1554540242416](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554540242416.png)

![1554540255369](C:\Users\nrchu\AppData\Roaming\Typora\typora-user-images\1554540255369.png)

#### (2) Query Pushdown