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

  - header를 보고 스키마를 추론할 수 있고 텍스트 파일 라인을 분리하는데 구분자를 지정할 수 있다
  
  - ```scala
    scala> val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statePopulation.csv")
    ```
  
- 명시 스키마

  - 모든 데이터 타입은 types 패키지에 속해있다.

    ```scala
    import org.apache.spark.sql.types._
    ```

    

  - StructField 객체의 컬렉션인 StructType을 사용해 스키마를 표현할 수 있다.

    ```scala
    scala> import org.apache.spark.sql.types.{StructType, IntegerType, StringType}
    scala> val schema = new StructType().add("i", IntegerType).add("s", StringType)
    scala> schema.printTreeString
    root
     |-- i: integer (nullable = true)
     |-- s: string (nullable = true)
    
    scala> schema.prettyJson
    res3: String =
    {
      "type" : "struct",
      "fields" : [ {
        "name" : "i",
        "type" : "integer",
        "nullable" : true,
        "metadata" : { }
      }, {
        "name" : "s",
        "type" : "string",
        "nullable" : true,
        "metadata" : { }
      } ]
    }
    ```

  - 인코더

    - 복잡한 데이터 타입을 정의할 때 인코더를 사용한다.

      ```scala
      scala> import org.apache.spark.sql.Encoders
      scala> Encoders.product[(Integer, String)].schema.printTreeString
      root
       |-- _1: integer (nullable = true)
       |-- _2: string (nullable = true)
      
      scala> case class Record(i: Integer, s: String)
      scala> Encoders.product[Record].schema.printTreeString
      root
       |-- i: integer (nullable = true)
       |-- s: string (nullable = true)
      
      scala> import org.apache.spark.sql.types.DataTypes
      scala> val arrayType = DataTypes.createArrayType(IntegerTypes)
      ```



- 데이터셋 저장

  - 스파크SQL은 DataFrameWriter 인터페이스를 이용해 외부 저장소 시스템에 저장할 수 있다.

    - Parquet, ORC, Text, Hive, Json, CSV, JDBC

    ```scala
    //statePopulation.csv 복사
    scala> val statesPopulationDF = spark.read.option("header", "true").option("inferschema", "true").option("seq", ",").csv("/user/irteamsu/input/statePopulation.csv")
    
    scala> statesPopulationDF.write.option("header", "true").csv("/user/irteamsu/input/statePopulation_dup.csv")
    
    //statesTaxRates.csv 복사
    scala> val statesTaxRatesDF = spark.read.option("header", "true").option("inferschema", "true").option("seq", ",").csv("/user/irteamsu/input/statesTaxRates.csv")
    
    scala> statesPopulationDF.write.option("header", "true").csv("/user/irteamsu/input/statesTaxRates_dup.csv")
    ```

- 암시 스키마

  - header를 보고 스키마를 추론할 수 있고 텍스트 파일 라인을 분리하는데 구분자를 지정할 수 있다
  
  - ```scala
    scala> val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statePopulation.csv")
    ```
  
- 명시 스키마

  - 모든 데이터 타입은 types 패키지에 속해있다.

    ```scala
    import org.apache.spark.sql.types._
    ```

    

  - StructField 객체의 컬렉션인 StructType을 사용해 스키마를 표현할 수 있다.

    ```scala
    scala> import org.apache.spark.sql.types.{StructType, IntegerType, StringType}
    scala> val schema = new StructType().add("i", IntegerType).add("s", StringType)
    scala> schema.printTreeString
    root
     |-- i: integer (nullable = true)
     |-- s: string (nullable = true)
    
    scala> schema.prettyJson
    res3: String =
    {
      "type" : "struct",
      "fields" : [ {
        "name" : "i",
        "type" : "integer",
        "nullable" : true,
        "metadata" : { }
      }, {
        "name" : "s",
        "type" : "string",
        "nullable" : true,
        "metadata" : { }
      } ]
    }
    ```

  - 인코더

    - 복잡한 데이터 타입을 정의할 때 인코더를 사용한다.

      ```scala
      scala> import org.apache.spark.sql.Encoders
      scala> Encoders.product[(Integer, String)].schema.printTreeString
      root
       |-- _1: integer (nullable = true)
       |-- _2: string (nullable = true)
      
      scala> case class Record(i: Integer, s: String)
      scala> Encoders.product[Record].schema.printTreeString
      root
       |-- i: integer (nullable = true)
       |-- s: string (nullable = true)
      
      scala> import org.apache.spark.sql.types.DataTypes
      scala> val arrayType = DataTypes.createArrayType(IntegerTypes)
      ```


- 데이터셋 저장

  - 스파크SQL은 DataFrameWriter 인터페이스를 이용해 외부 저장소 시스템에 저장할 수 있다.

    - Parquet, ORC, Text, Hive, Json, CSV, JDBC

    ```scala
    //statePopulation.csv 복사
    scala> val statesPopulationDF = spark.read.option("header", "true").option("inferschema", "true").option("seq", ",").csv("/user/irteamsu/input/statePopulation.csv")
    
    scala> statesPopulationDF.write.option("header", "true").csv("/user/irteamsu/input/statePopulation_dup.csv")
    
    //statesTaxRates.csv 복사
    scala> val statesTaxRatesDF = spark.read.option("header", "true").option("inferschema", "true").option("seq", ",").csv("/user/irteamsu/input/statesTaxRates.csv")
    
    scala> statesPopulationDF.write.option("header", "true").csv("/user/irteamsu/input/statesTaxRates_dup.csv")
    ```

- 집계

  - 스파크 세션 생성 테스트

    ```scala
    val conf = new SparkConf()
    val spark = SparkSession
    .builder()
    .appName("aggregation")
    .config(conf)
    .getOrCreate()
    
    val inputPath = args(0)
    
    val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv(inputPath)
    statesDF.show(5)
    +----------+----+----------+
    |     State|Year|Population|
    +----------+----+----------+
    |   Alabama|2010|   4785492|
    |    Alaska|2010|    714031|
    |   Arizona|2010|   6408312|
    |  Arkansas|2010|   2921995|
    |California|2010|  37332685|
    +----------+----+----------+
    
    //column이 State인 데이터 개수 출력
    statesDF.select(col("*")).agg(count("State")).show 
    statesDF.select(count("State")).show
    
    //column이 중복 제외하고 State 개수 출력
    statesDF.select(col("*")).agg(countDistinct("State")).show
    statesDF.select(countDistinct("State")).show
    
    //State의 첫번째 값 출력
    statesDF.select(first("State")).show //Alabama
    
    //State의 마지막 값 출력
    statesDF.select(last("State")).show //Wyoming
    
    //colume이 State인 것들의 대략적인 개수를 구한다
    statesDF.select(col("*")).agg(approx_count_distinct("State")).show //48
    statesDF.select(col("*")).agg(approx_count_distinct("State", 0.2)).show//49
    
    //Population Column에 대해서 최대 최소 
    statesDF.select(min("Population")).show
    statesDF.select(max("Population")).show
    
    //avg, sum, sumDistinct
    statesDF.select(avg("Population")).show
    statesDF.select(sum("Population")).show
    statesDF.select(sumDistinct("Population")).show
    
    //첨도 : 분포의 중간과 끝의 가중치
    statesDF.select(kurtosis("Population")).show
    
    //비대칭도 : 평균 근처의 데이터 값에 대한 비대칭 측정
    statesDF.select(skewness("Population")).show
    
    //분산 : 개별값에서 평균의 차르 제곱한 후의 평균
    statesDF.select(var_pop("Population")).show
    
    //표준편차
    statesDF.select(stddev("Population")).show
    
    //공분산 (두 랜덤 변수의 결합 가변성을 측정)
    statesDF.select(covar_pop("Year", "Population")).show
    
    ```

  - 실행

    ```shell
    ./build.sh sparkcollection.aggregation.AggregationMain . /user/irteamsu/input/statePopulation.csv
    ```

  - groupby

    ```scala
    //groupby
    statesDF.groupBy("State").count.show(5)
    +---------+-----+
    |    State|count|
    +---------+-----+
    |     Utah|    7|
    |   Hawaii|    7|
    |Minnesota|    7|
    |     Ohio|    7|
    | Arkansas|    7|
    +---------+-----+
    
    //groupby 응용
    statesDF.groupBy("State").agg(min("Population"), avg("Population")).show(5)
    ```

  - rollup (중첩계산, 주+연도 그룹의 레코드 개수와 주별 레코드 개수를 표시하고 싶다면)

    ```scala
    scala> statesDF.rollup("State", "Year").count.show(5)                                            
    //California + 2014를 기준으로 계산한다
    +------------+----+-----+
    |       State|Year|count|
    +------------+----+-----+
    |South Dakota|2010|    1|
    |    New York|2012|    1|
    |  California|2014|    1|
    |     Wyoming|2014|    1|
    |      Hawaii|null|    7|
    +------------+----+-----+
    ```
  
  - cube

    - 롤업처럼 계층이나 중첩계산을 수행하는데 사용되는 다차원 집계지만 모든 차원에 동일한 연산을 수행한다는 차이점이 있다.

    ```scala
    scala> statesDF.cube("State", "Year").count.show(5)
    +------------+----+-----+
    |       State|Year|count|
    +------------+----+-----+
    |South Dakota|2010|    1|
    |    New York|2012|    1|
    |  California|2014|    1|
    |     Wyoming|2014|    1|
    |      Hawaii|null|    7|
    +------------+----+-----+
    ```

  - 윈도우 함수

    ```scala
    scala> import org.apache.spark.sql.expressions.Window
    scala> import org.apache.spark.sql.functions.col
    scala> import org.apache.spark.sql.functions.max
    
    scala> val windowSpec = Window.partitionBy("State").orderBy(col("Population").desc).rowsBetween(Window.unboundedPreceding, Window.currentRow)
    
    scala> statesDF.select(col("State"), col("Year"), max("Population").over(windowSpec), rank().over(windowSpec)).sort("State", "Year").show(10)
    |  State|Year| ..| ..
    |Alabama|2010|4863300|6|
    |Alabama|2011|4863300|7|
    |Alabama|2012|4863300|5|
    |Alabama|2013|4863300|4|
    |Alabama|2014|4863300|3|
    |Alabama|2015|4863300|2|
    |Alabama|2016|4863300|1|
    | Alaska|2010|741894|7|
    | Alaska|2011|741894|6|
    | Alaska|2012|741894|5|
    
    //ntiles 사용 (입력 데이터셋을 n개의 부분집합으로 나눈다, 공평하게 데이터를 나누는데 사용)
    statesDF.select(col("State"), col("Year"), ntile(2).over(windowSpec), rank().over(windowSpec)).sort("State", "Year").show(10)
    |Alabama|2010|2|6|
    |Alabama|2011|2|7|
    |Alabama|2012|2|5|
    |Alabama|2013|1|4|
    |Alabama|2014|1|3|
    |Alabama|2015|1|2|
    |Alabama|2016|1|1|
    | Alaska|2010|2|7|
    | Alaska|2011|2|6|
    | Alaska|2012|2|5|
    ```

- 조인

  - leftanti

    - 오른쪽에 존재하지 않는 것을 기반으로 왼쪽의 로우만 제공
    - 왼쪽 중에 오른쪽에 없는 애들 추출 (leftouter에서 null을 제거한 애들 -> 가장 많이 쓰일듯)

  - leftouter

    - 왼쪽의 모든 로우와 오른쪽의 공통 로우를 추가한다. 오른쪽에 값이 없으면 NULL을 채운다
    - 왼쪽과 오른쪽 모두 추출하지만 오른쪽에 없으면 NULL

  - leftsemi

    - 오른쪽에 존재하는 값을 기반으로 왼쪽에 있는 로우만 제공(오른쪽에 있는 값은 포함하지 않는다)
    - 왼쪽 오른쪽 둘다 있는 것들과 무슨차이?

  - rightouter

    - 오른쪽의 모든 로우와 왼쪽 및 오른쪽의 공통 로우를 추가해 제공한다. 왼쪽에 로우가 없다면 NULL을 채운다.
    - 왼쪽 오름쪽 모두 추출하지만 왼쪽에 없으면 NULL

  - outer

    - 왼쪽 오른쪽 모두 합침 (UNION과의 차이점?)

  - cross

    - 왼쪽과 오른쪽 모두 일치시켜 카테시안 교차곱 생성

  - sample

    - INNER JOIN

    ```scala
    val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statePopulation.csv")
    
    val statesDF = spark.read.option("header", "true").option("inferschema", "true").option("sep", ",").csv("/user/irteamsu/input/statesTaxRates.csv")
    
    //application에서 처리
    statesDF.count //350
    taxDF.count //47
    var joinDF = statesDF.join(taxDF, statesDF("State") === taxDF("State"), "inner")
    //inner를 leftouter, rightouter, fullouter, leftanti, leftsemi로 바꿀 수 있음
    joinDF.show //Dateframe 보여줌
    joinDF.count //322
    
    //spark-sql에서 처리
    statesDF.createOrReplaceTempView("statesDF") //테이블로 올리기
    taxDF.createOrReplaceTempView("taxDF") //테이블로 올리기
    var joinSqlDF = spark.sql("SELECT * FROM statesDF INNER JOIN taxDF ON statesDF.State = taxDF.State")
    joinDF.show //Dateframe 보여줌
    joinDF.count //322
    ```

