## 5. 스파크 

- 서버 : dev-shop-collection-nn-ncl

- cluster manager : http://dev-shop-collection-nn-ncl.nfra.io:7180/cmf/home

- 실행 스크립트 : home1/irteamsu/run-spark-client-mode.sh

- web-ui

  - http://dev-shop-collection-nn-ncl.nfra.io:4040

- RDD란

  - 스파크에서 사용하는 기본객체
  - 파티셔닝 되어있는 정수 배열이라고 생각하면 쉽다
  - 불변 컬렉션이며 트랜스포메이션 시 새로운 RDD를 생성하기 때문에 저장된 계보로 복구가 가능하다

- RDD 생성 방법

  - 컬렉션 병렬화

    - sc.parallelize(...)

      ```scala
      val rdd_one = sc.parallelize(Seq(1,2,3)) //1,2,3 순서열에 저장
      rdd_one.take(10) //10개 가져옴
      ```

  - 외부에서 데이터 로드

    - sc.textFile(...)

    - textFile은 입력 데이터를 텍스트 파일로 로드하는 함수인데 
      각각의 Line은 \n으로 끝나며 이 부분이 모두 RDD의 엘리먼트가 된다.

      

      ```scala
      hdfs dfs -mkdir /user/irteamsu/input
      hdfs dfs -put ./wiki1.txt /user/irteamsu/input/
      scala> val rdd_two = sc.textFile("/user/irteamsu/input/wiki1.txt")
      scala> rdd_two.count //9가 출력 
      scala> rdd_two.first
      
      res2: String = Apache Spark provides programmers with an application programming interface centered on a data structure called the resilient distributed dataset (RDD), a read-only multiset of data items distributed over a cluster of machines, that is maintained in a fault-tolerant way.
      ```

      

  - 기존 RDD를 트랜스포메이션 하는 경우

    ```scala
    scala> val rdd_one_x2 = rdd_one.map(i => i * 2)
    scala> rdd_one_x2.take(10) //2,4,6
    ```

  - 스트리밍 API

    - 스파크 스트리밍을 이용해 RDD를 생성할 수 있고 이런 RDD를 불연속 스트림 RDD 혹은 Dstream RDD라 부른다

- 트랜스포메이션과 액션 

  - RDD는 불변이고 모든 연산은 새로운 RDD를 생성

  - RDD에서 수행할 수 있는 두가지 주요 연산이 (1) 트랜스포메이션과 (2) 액션이다.

    - 트랜스포메이션
      - 입력 엘리먼트를 분리, 필터링 등으로 RDD 엘리먼트를 변경한다
      - 계획중에는 트랜스포메이션 실행이 일어나지 않는다.
      - 일반 트랜스포메이션
        - Map, filter, flatMap, groupByKey, sortByKey, combineByKey ...
      - 수학/통계 트랜스포메이션
        - sampleByKey, randomSplit ...
      - 집합 이론 / 관계형 트랜프포메이션
        - Cogroup, join, subtractByKey, fullOuterJoin, leftOuterJoin, rigthOuterJoin ...
      - 데이터 구조 기반 트랜스포메이션
        - partitionBy, repartition, zipwithIndex, coalesce ...

    > DAG에 트랜스포메이션을 추가하고 드라이버가 데이터를 요청할 때만 해당 DAG가 실제로 실행된다.
    >
    > 이를 느긋한 계산 (lazy-evaluation) 이라한다. 
    >
    > 스파크는 드라이버가 가진 모든 연산을 한번 훑어보고 효율적인 방법으로 최적화하여 실행한다.

    

    - 액션
      - 실제로 계산이 수행되는 연산
      - 액션 연산이 발생하기 전까지 스파크 프로그램의 실행계획은 DAG 형태로만 만들어지며 실제 수행은 없다.



- map

  ```scala
  scala> val rdd_two = sc.textFile("/user/irteamsu/input/wiki1.txt")
  scala> val rdd_three = rdd_two.map(line => line.length) //map의 결과로 새로운 RDD가 만들어진다
  scala> rdd_three.take(10) //데이터 총 9줄 각각의 문자열 길이를 리턴
  ```

- flatMap

  - map과 비슷하지만 RDD 엘리먼트의 모든 컬렉션을 평평(flat)하게 한다

  ```scala
  scala> var rdd_three = rdd_two.map(line => line.split(" ")) //띄어쓰기 별로 모두 끊어 array에 저장
  scala> rdd_three.take(1) //1줄만 가져옴 ex. array(array(apache, spark, provides ...))
  
  scala> var rdd_three = rdd_two.flatMap(line => line.split(" "))
  scala> rdd_three.take(1) //단어 하나나옴 ex. array(apache)
  scala> rdd_three.take(10) //단어 하나나옴 ex. array(apache, spark, provides)
  ```

- filter

  ```scala
  scala> val rdd_three = rdd_two.filter(line => line.contains("Spark"))
  scala> rdd_three.count //5
  ```

- coalesce

  - 입력 파티션(RDD는 파티션의 조합)을 출력 RDD의 더 작은 **파티션**으로 **결합**한다 

  ```scala
  scala> rdd_two.partitions.length //2
  scala> val rdd_three = rdd_two.coalesce(1)
  scala> rdd_three.partitions.length //1, coalesce(2)를 하면 2가 나옴
  ```

- repartition

  - 입력 RDD를 출력 RDD에서 더 적거나 더 많은 출력 파티션으로 다시 파티셔닝

  ```scala
  scala> val rdd_three = rdd_two.repartition(5)
  scala> rdd_three.partitions.length //5
  ```

- 액션

  - reduce

    - RDD의 모든 엘리먼트에 reduce함수를 적용하고 적용한 결과를 드라이버에 전달한다.
    - 마지막의 연산 결과를 수집한다.

    ```scala
    scala> val rdd_one = sc.parallelize(Seq(1,2,3,4,5,6))
    scala> rdd_one.take(10)
    scala> rdd_one.reduce((a,b) => a+b) //21
    ```

  - count

    - RDD의 엘리먼트 개수를 구한다

    ```scala
    scala> rdd_one.count //6
    ```

  - collect

    - RDD의 모든 엘리먼트를 얻은 결과를 드라이버에 보낸다. 
    - 드라이버가 모든 파티션의에서 RDD의 모든 엘리먼트를 수집한다.
    - 큰 RDD에 collect()를 호출하면 드라이버에서 메모리 부족 문제가 발생한다.

    ```scala
    scala> val rdd_two = sc.textFile("/user/irteamsu/input/wiki1.txt")
    scala> rdd_two.collect
    ```



- 캐싱

  - RDD를 메모리에 저장한다 (LRU 정책을 따름)

  - persist(newLevel: StorageLevel)

    - MEMORY_ONLY (default : 가장 빠름)
    - MEMORY_AND_DISK
    - MEMORY_ONLY_SER : 객체를 더 작게 만들기 위해 직렬화 가능한 객체가 저장됨
    - MEMORY_AND_DISK_SER
    - DISK_ONLY
    - MEMORY_ONLY_2, MEMORY_AND_DISK_2 등...
    - OFF_HEAP (가장 느림)

  - unpersist는 캐싱한 내용을 단순히 해제한다.

    ```scala
    scala> import org.apache.spark.storage.StorageLevel
    scala> rdd_one.persist(StorageLevel.MEMORY_ONLY)
    scala> rdd_one.unpersist()
    scala> rdd_one.persist(StorageLevel.DISK_ONLY)
    scala> rdd_one.unpersist()
    
    scala> rdd_one.count //1초
    scala> rdd_one.cache
    scala> rdd_one.count //83ms
    ```

    

- 데이터로드

  - textFile, whileTextFiles, JDBC 데이터소스에서 로드

    - whileTextFiles

      - 파일 이름과 파일 전체 내용을 나타내는 <filename, textOfFile> 쌍을 가진 RDD에 여러 텍스트 파일을 로드
      - textFile과는 다르게 파일 전체 내용을 단일 레코드로 로드한다.

      ```scala
      scala> val rdd_whole = sc.wholeTextFiles("/user/irteamsu/input/wiki1.txt")
      scala> rdd_whole.take(1)
      ```

    - Jdbc 로드 (안되네...?)

      ```scala
      scala> sqlContext.load(path=None, source=None, schema=None, **options)
      scala> val dbContent = sqlContext.load(source="jdbc", url="jdbc:mysql://10.105.197.53:3306/test", dbtable="test", partitionColumn="id")
      ```

- RDD 저장

  - saveAsTextFile
  - saveAsObjectFIle

  ```scala
  scala> rdd_one.saveAsTextFile("out.txt")
  ```



- RDD

  - map의 쓰임

  ```scala
  scala> val rdd_one = sc.parallelize(Seq(1,2,3,4,5,6))
  scala> val rdd_two = rdd_one.map(i => i * 3)
  scala> val rdd_three = rdd_two.map(i => i + 2) // Array(5, 8, 11, 14, 17, 20)
  scala> val rdd_four = rdd_three.map(i => ("str"+(i+2).toString, i-2)) //(x,y)로 만듬
  scala> rdd_four.take(10) //res23: Array[(String, Int)] = Array((str7,3), (str10,6), (str13,9), (str16,12), (str19,15), (str22,18))
  ```

  - 대문자 치환

  ```scala
  scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv")
  scala> statesPopulationRDD.first // State,Year,Population
  scala> val upperCaseRDD = statesPopulationRDD.map(_.toUpperCase)
  scala> upperCaseRDD.take(10)
  ```

  - 쌍 RDD

    - 쌍(Pair) RDD는 집계, 정렬, 데이터 조인에 적합한 key-value 튜플로 구성된 RDD이다.

    ```scala
    scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv")
    scala> val pairRDD = statesPopulationRDD.map(record =>
         | (record.split(",")(0), record.split(",")(2))) //split한 0번째, 2번째 index 튜플 생성
    ```

  - Mean, min, max, stdev

    ```scala
    scala> val rdd_one = sc.parallelize(Seq(1.0, 2.0, 3.0))
    scala> rdd_one.mean
    res29: Double = 2.0
    
    scala> rdd_one.min
    res30: Double = 1.0
    
    scala> rdd_one.max
    res31: Double = 3.0
    
    scala> rdd_one.stdev
    res32: Double = 0.816496580927726
    ```

  - SequenceFileRDD

    - SequenceFileRDD는 하둡의 파일포맷인 SequenceFile에서 생성된다.
    - SequenceFile은 압축될 수 있고, 압축이 해제될 수 있다.
    - 하둡의 맵리듀스 프로세스는 키-값 쌍인 SequenceFiles를 사용할 수 있다.
    - Text, IntWritable 등과 같은 하둡쓰기 가능한 데이터 타입이다.

    ```scala
    scala> pairRDD.saveAsSequenceFile("seqfile") //pairRDD를 sequenceFile로 만듬
    scala> val seqRDD = sc.sequenceFile[String, String]("seqfile") //sequenceFile로 sequenceRDD를 만든다
    scala> seqRDD.take(10) //결과적으로 pairRDD와 seqRDD는 같음
    ```

- CoGroupedRDD

  - CoGroupedRDD는 RDD의 부모와 함께 그룹핑 되는 RDD 

  - 기본적으로 공통 키와 양 부모RDD의 값 리스트로 구성된 pairRDD를 생성하기 때문에 양 부모 RDD는 pairRDD여야 한다.

  - 쉽게 말해 그룹의 그룹

  - pairRDD가 (Alabama, 12345), (Delaware, 67890)이고 pairRDD2가 (Alabama, 2010), (Delaware, 2011)이라면 coGroup은 (Alabama, (2010, 12345)), (Delaware, (2011, 67890))이 된다.

    

  ```scala
  scala> val pairRDD = statesPopulationRDD.map(record =>
       | (record.split(",")(0), record.split(",")(2)))
  scala> val pairRDD2 = statesPopulationRDD.map(record =>
       | (record.split(",")(0), record.split(",")(1)))
  scala> val cogroupRDD = pairRDD.cogroup(pairRDD2)
  scala> cogroupRDD.take(10)
  ```

- ShuffledRDD

  - RDD 엘리먼트를 섞어 동일한 익스큐터에서 동일한 키에대한 값을 누적해 집계하거나 로직을 결합할 수 있다.

    ```scala
    scala> val pairRDD = statesPopulationRDD.map(record =>
         | (record.split(",")(0), 1))
    scala> pairRDD.take(5)
    scala> val shuffledRDD = pairRDD.reduceByKey(_+_) //같은 key로 value를 sum
    scala> shuffledRDD.take(5) 
    // 결과 : res43: Array[(String, Int)] = Array((Montana,7), (California,7), (Washington,7), (Massachusetts,7), (Kentucky,7))
    
    //Montana, California 각각이 7개 있음.
    ```

- UnionRDD

  - ```scala
    scala> val rdd_one = sc.parallelize(Seq(1,2,3))
    scala> val rdd_two = sc.parallelize(Seq(4,5,6))
    scala> val unionRDD = rdd_one.union(rdd_two)
    scala> unionRDD.take(10) //1,2,3,4,5,6
    ```

- HadoopRDD 1.0

  - 하둡 MR API를 사용해 HDFS에 저장된 데이터를 읽는다 
  - sc.textFile("하둡 Path")

- HadoopRDD 2.0

  - 하둡 MR API를 사용해 HDFS, HBase, C3에 저장된 데이터를 읽는다

  ```scala
  scala> 
  import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat
  import org.apache.hadoop.io.Text
  
  val newHadoopRDD = sc.newAPIHadoopFile("/user/irteamsu/input/statePopulation.csv", classOf[KeyValueTextInputFormat], classOf[Text], classOf[Text])
  
  scala> newHadoopRDD.take(3) //Array((Alaska,2010,714031,), (Alaska,2010,714031,), (Alaska,2010,714031,))
  ```

  