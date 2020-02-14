## Spark-sql

- 데이터프레임이란?
  - RDD를 추상화한 것.
  - RDD를 내부적으로 사용하지만 바이너리 구조로 직렬화 하기 때문에 성능이 더 좋다.
    - 스파크 2.0 부터 데이터프레임은 단순히 데이터셋의 alias이다.
      - type DataFram = dataset[Row]
  - RDB와 비슷하다.
    - Row들을 가지고 각 Row들은 여러 칼럼으로 구성된다.
  - 불변이다(RDD와 마찬가지)
  - RDD와의 차이점
    - RDD 기반위에서 생성되었음 = RDD의 추상화 버전이라 바로 사용하기 쉬운 고급(high level) API를 제공한다.
    - RDD는 데이터프레임에 비하면 로우레벨(row level)의 데이터 조작을 지원함
  - hive와 유사함
  - 데이터 프레임인 테이블이 있다면 데이터 프레임을 테이블로 등록할 수 있고, 데이터 프레임 API대신에 스파크 SQL을 사용해 데이터를 조작할 수 있다.
  - 데이터는 일반적으로 DataFrameReader를 통해 데이터 프레임으로 로그되며, DataFrameWriter를 통해 데이터 프레임에서 데이터가 저장된다.
  
- 데이터프레임 생성방법

  - RDD를 데이터프레임으로 변환

  - SQL쿼리 실행

    - 간단하게 데이터프레임 API로 변환된다.

      ```scala
      //데이터프레임 로드 (from. file.csv)
      scala> val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statePopulation.csv") //(header, inferschema, sep)를 세팅하는 부분은 스파크에 csv 헤더가 존재한다는 것을 알리고 쉼표(,)를 사용해 필드 칼럼을 구분하여 스키마를 암시적으로 유추하도록 하는 과정이다.
      
      //데이터프레임 테이블에서 5개 출력
      scala> statesDF.show(5)
      
      //Spark SQL을 통해서 5개 출력 (stateDF.show(5)와 결과가 동일함)
      //테이블로 만들기
      scala> statesDF.createOrReplaceTempView("states") 
      scala> spark.sql("select * from states limit 5").show //spark.sql()의 리턴 결과가 데이터프레임임.
      ```

      

  - Parquet, Json, CSV, 텍스트, Hive, JDBC 등의 외부 데이터 로드

    - CSV 파일 로드

      ```scala
      //데이터프레임 로드 (from. file.csv)
      scala> val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statePopulation.csv")
      
      //데이터프레임의 스키마 출력
      scala> statesDF.printSchema 
      root
       |-- State: string (nullable = true)
       |-- Year: integer (nullable = true)
       |-- Population: integer (nullable = true)
      
      //데이터프레임 실행계획 출력
      scala> statesDF.explain(true)
      ```

    

- Spark-Sql 사용

  ```scala
  //정렬
  scala> statesDF.sort(col("Population").desc).show(5)
  scala> spark.sql("select * from states order by Population desc limit 5").show()
  
  //합산 (group by)
  scala> statesDF.groupBy("state").sum("Population").show(5)
  scala> spark.sql("select State, sum(Population) from states group by State limit 5").show
  
  //alias 사용법 (AS)
  scala> statesDF.groupBy("State").agg(sum("Population").alias("Total")).show(5)
  scala> spark.sql("select State, sum(Population) as Total from states group by State limit 5").show
  scala> statesDF.groupBy("State").agg(sum("Population").alias("Total")).explain(true) //실행계획 보기
  
  //합산 + 정렬
  scala> statesDF.groupBy("State").agg(sum("Population").alias("Total")).sort(col("Total").desc).show(5)
  scala> spark.sql("select State, sum(Population) as Total from states group by State order by Total desc limit 5").show
  
  //동시에 여러 합산 + 정렬
  scala> statesDF.groupBy("State").agg(min("Population").alias("minTotal"), max("Population").alias("maxTotal"), avg("Population").alias("avgTotal")).sort(col("minTotal").desc).show(5)
  ```



- 피벗

  ```scala
  scala> statesDF.groupBy("State").pivot("Year").sum("Population").show(5)
  +---------+--------+--------+--------+--------+--------+--------+--------+
  |    State|    2010|    2011|    2012|    2013|    2014|    2015|    2016|
  +---------+--------+--------+--------+--------+--------+--------+--------+
  |     Utah| 2775326| 2816124| 2855782| 2902663| 2941836| 2990632| 3051217|
  |   Hawaii| 1363945| 1377864| 1391820| 1406481| 1416349| 1425157| 1428557|
  |Minnesota| 5311147| 5348562| 5380285| 5418521| 5453109| 5482435| 5519952|
  |     Ohio|11540983|11544824|11550839|11570022|11594408|11605090|11614373|
  | Arkansas| 2921995| 2939493| 2950685| 2958663| 2966912| 2977853| 2988248|
  +---------+--------+--------+--------+--------+--------+--------+--------+
  ```

  

- 필터

  - 새로운 데이터프레임을 만들때 로우를 필터링 함으로써 연산 성능을 향상시킬 수 있다

    ``` scala
    //캘리포니아만 추출
    scala> statesDF.filter("State == 'California'").show(5)
    ```

- UDF(User Defined Functino)

  - 직접 함수 만들기

    ```scala
    scala> import org.apache.spark.sql.functions._
    
    scala> val toUpper: String => String = _.toUpperCase
    
    scala> val toUpperUDF = udf(toUpper)
    
    //toUpper(col("State")) 를 하면 col("State")는 타입이 COLUMN인데 toUpper의 파라메터 타입은 String이라는 오류가 난다.
    scala> statesDF.withColumn("StateUpperCase", toUpperUDF(col("State"))).show(5)
    
    +----------+----+----------+--------------+
    |     State|Year|Population|StateUpperCase|
    +----------+----+----------+--------------+
    |   Alabama|2010|   4785492|       ALABAMA|
    |    Alaska|2010|    714031|        ALASKA|
    |   Arizona|2010|   6408312|       ARIZONA|
    |  Arkansas|2010|   2921995|      ARKANSAS|
    |California|2010|  37332685|    CALIFORNIA|
    +----------+----+----------+--------------+
    ```

- 암시 스키마

  