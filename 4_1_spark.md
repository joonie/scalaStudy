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

  
- 집계

  - 주 별 인구별 상위 연도를 구하고 싶다

    - 주를 key로 잡고 해당 key에 대한 인구별 상위 연도를 value로 잡으면 된다.

    ```scala
    scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv")
    scala> val pairRDD = statesPopulationRDD.map(record => record.split(",")).map(t => (t(0), (t(1), t(2)))) //split하고 0번 index를 key로 잡고 (2번, 3번) tuple을 value로 잡는다.
    ```

  - 연도 별 인구 별 상위 주를 구하고 싶다

    - 연도를 key로 잡고 해당 key에 대한 인구 별 상위 주 들을 value로 잡으면 된다.

    ```scala
    scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv")
    scala> val pairRDD = statesPopulationRDD.map(record => record.split(",")).map(t => (t(1), (t(0), t(2)))) //split하고 2번 index를 key로 잡고 (1번, 3번) tuple을 value로 잡는다.
    ```

  - 튜플의 pairRDD에 대한 집계함수

    - groupByKey

      - RDD의 각 키 값을 하나의 시퀀스로 그룹핑한다.
      - 셔플링을 진행하기 때문에 비싼 연산이다, 너무 많은 셔플링을 하게 된다.
      - 메모리에 모든 키-값 쌍을 보유해야함으로 OOM이 발생할 수 있다.
      - 키를 통해 해시코드를 만들고 이것을 기준으로 셔플링해서 동일한 파티션의 각 키에 대한 값을 수집.

    - reduceByKey

      - 모든 엘리먼트를 셔플링하지 않고 먼저 로컬에서 일부 기본 집계를 수행한 후에 groupByKey 수행을 진행한다.
      - 모든 데이터를 전송할 필요가 없기 때문에 전송데이터가 크게 줄어든다.
      - 결과를 groupByKey와 같지만 성능이 더 좋음.
      - 하둡 맵리듀스에 익숙하다면 맵리듀스 프로그래밍의 컴바이너와 매우 유사하다.

    - aggregateByKey

      - reduceByKey와 매우 유사하지만 기능이 더 강력함.
      - 동일한 데이터 타입에서 동작할 필요가 없고 파티션간에 다른 집계를 수행할 수 있다.
      - key를 합치고 value를 aggregate함.

    - combineByKey

      - 컴바이너를 생성하는 초기 함수를 제외하고 aggregateByKey와 성능이 매우 비슷.

        

  - ```scala
    
    //1단계: RDD를 초기화한다.
    scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv").filter(_.split(",")(0) != "State") //split 한 것의 0번째가 "State"가 아닌 라인들 추출
    scala> statesPopulationRDD.take(10)
    res1: Array[String] = Array(Alabama,2010,4785492, Alaska,2010,714031, Arizona,2010,6408312, Arkansas,2010,2921995, California,2010,37332685, Colorado,2010,5048644, Delaware,2010,899816, District of Columbia,2010,605183, Florida,2010,18849098, Georgia,2010,9713521)
    
    //2단계: pairRDD로 변환한다. (지역 별 년도와 일구수 튜블을 생성)
    scala> val pairRDD = statesPopulationRDD.map(record =>
         | record.split(",")).map(t => (t(0), (t(1).toInt, t(2).toInt)))
    scala> pairRDD.take(10)
    res2: Array[(String, (Int, Int))] = Array((Alabama,(2010,4785492)), (Alaska,(2010,714031)), (Arizona,(2010,6408312)), (Arkansas,(2010,2921995)), (California,(2010,37332685)), (Colorado,(2010,5048644)), (Delaware,(2010,899816)), (District of Columbia,(2010,605183)), (Florida,(2010,18849098)), (Georgia,(2010,9713521)))
    
    //3단계: 값을 그룹핑하고 인구를 더한다 (지역별 인구수의 합)
    scala> val groupedRDD = pairRDD.groupByKey.map(x => {
         | var sum=0;
         | x._2.foreach(sum += _._2); //x._2 : x 튜플의 2번째 원소, _._2 : 임의의 앨리먼터의 2번째 원소
         | (x._1, sum)})
    
    //4단계: 키로 값을 리듀싱하고 인구를 더한다.
    //여기서 reduceByKey는 키로 묶는다는걸 의미하고 x._2+y,_2는 value의 2번째 인자를 모두 더한다는 의미
    //즉 키값을 기준으로 하나의 year와 합쳐진 인구가 만들어진다. (Montana,(2010,7105432):총 7105432개)
    scala> val reduceRDD = pairRDD.reduceByKey((x,y) => (x._1, x._2+y._2))
    .map(x => (x._1, x._2._2)) //(Montana,7105432)...
    
    //5단계
    scala> val initialSet = 0
    val addToSet = (s: Int, v: (Int, Int)) => s + v._2
    val mergePartitionSets = (p1: Int, p2: Int) => p1 + p2
    
    val aggregatedRDD = pairRDD.aggregateByKey(initialSet)(addToSet, mergePartitionSets)
    
    scala> aggregatedRDD.take(10) //(Montana,7105432)...
    
    //6단계
    scala> val createCombiner = (x:(Int, Int)) => x._2
    scala> val mergeValues = (c:Int, x:(Int, Int)) => c + x._2
    scala> var mergeCombiners = (c1:Int, c2:Int) => c1 + c2
    scala> val combinedRDD = pairRDD.combineByKey(createCombiner, mergeValues, mergeCombiners)
    scala> combinedRDD.take(10) //(Montana,7105432)...
    ```

- 파티션

  - 파티션 개수 : 스파크 설정 파라미터 이며 spark.default.parallelism 또는 클러스터의 코어 개수 중 하나
  - 파티션이 적으면 일부의 worker만 동작하기 때문에 성능이 저하된다.
  - 파티션이 너무 많으면 실제보다 많은 자원을 사용하기 때문에 자원 부족 현상이 나타날 수 있다.

  - RDD의 파티션 개수 확인

    ```scala
    scala> val rdd_one = sc.parallelize(Seq(1,2,3))
    scala> rdd_one.getNumPartitions //2
    ```

  - 파티션 방법

    - hashCode 파티션

    - Range 파티션

      ```scala
      scala> import org.apache.spark.RangePartitioner
      scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv")
      scala> val pairRDD = statesPopulationRDD.map(record => (record.split(",")(0), 1))
      scala> pairRDD.mapPartitionsWithIndex((i,x) => Iterator(""+i+":"+x.length)).take(10)
      res1: Array[String] = Array(0:177, 1:174) //partition 2개 (pairRDD.getNumPartitions)
      
      scala> val rangePartitioner = new RangePartitioner(5, pairRDD)
      scala> val rangePartitionedRDD = pairRDD.partitionBy(rangePartitioner)
      scala> rangePartitionedRDD.mapPartitionsWithIndex((i,x) => Iterator(""+i+":"+x.length)).take(10) //파티션 5개로 변경
      res3: Array[String] = Array(0:70, 1:63, 2:77, 3:70, 4:71)
      ```

      - pairRDD.getNumPartitions가 만약 default로 6이었다면 pairRDD는 6개 중 3개의 파티션에 데이터가 모여있었을 것.
      - 리파티셔닝을 하면 파티션 개수가 6->5로 바뀌고 데이터가 5개의 파티션에 골고루 들어가게 된다.

- 셔플링
  - 많은 연산을 처리하다보면 새로운 파티션이 생성되거나 축소되거나 병합될 수 있다. -> 리파티셔닝
    - 조인, 리듀스, 그룹핑, 집계 연산 등이 발생할 때 리파티셔닝 및 셔플링을 유발함.
      - aggregateBykey, reduceByKey등이 유발.
      - filter, map, flatMap 등은 셔플링을 유발하지 않음.
  - 리파티셔닝에 필요한 모든 데이터의 이동을 셔플링(shuffling)이라 한다.
  - 익스큐터간에 데이터를 교환하기 때문에 이 셔플링 과정에서 많은 성능 지연이 발생한다.
    - groupby 를 하면 각 익스큐터에서 하나의 익스큐터로 데이터가 모이게 됨.



- 브로드캐스트 변수

  - 모든 익스큐터에서 사용할 수 있는 공유 변수

  - 모든 익스큐터 메모리에 올라가게 된다.

  - 제거 : unpersist()

  - 정리 : destroy 정리 후 해당 변수를 사용하게 되면 예외가 발생한다.

  - 변수 생성

    ```scala
    scala> val rdd_one = sc.parallelize(Seq(1,2,3))
    scala> val i=5
    scala> val bi = sc.broadcast(i) //브로드캐스트 설정
    scala> bi.value //5
    scala> rdd_one.take(5)//1,2,3
    scala> rdd_one.map(j => j + bi.value).take(5)//6,7,8
    scala> bi.unpersist //브로드캐스트 제거
    
    scala> val m = scala.collection.mutable.HashMap(1->2, 2->3, 3->4)
    scala> val bm = sc.broadcast(m) //브로드캐스트 설정
    scala> rdd_one.map(j => j * bm.value(j)).take(5)//1*2, 2*3, 3*4
    scala> m.destroy //브로드캐스트 정리
    
    ```

    

- 누산기

  - 카운팅 하는데 사용.

  - LongAccumulator, DoubleAccumulator, CollectionAccumulator[T]

    ```scala
    scala> val acc1 = sc.longAccumulator("acc1")
    scala> val someRDD = statesPopulationRDD.map(x => {acc1.add(1); x})
    scala> acc1.value //0
    scala> someRDD.count //Long = 351
    scala> acc1.value //351 someRDD.count를 한 뒤로 351이 할당됨
    scala> acc1
    res14: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 364, name: Some(acc1), value: 351)
    ```

  - AccumulatorV2 클래스를 확장해서 사용할 수 있다.

    ```scala
    import org.apache.spark.util.AccumulatorV2
    
    class MyAccumulator extends AccumulatorV2[Int, Int] {
      //boolean인지 확인
         override def isZero: Boolean = ???
      //누산기를 복사해 다른 누산기를 생성
         override def copy(): AccumulatorV2[Int, Int] = ???
      //값 재설정
         override def reset(): Unit = ???
      //누산기에 특정 값 추가
         override def add(v: Int): Unit = ???
      //2개의 누산기를 병합
         override def merge(other: AccumulatorV2[Int, Int]): Unit = ???
      //누산기 값 리턴
         override def value: Int = ???
         }
    
    case class YearPopulation(year: Int, population: Long)
    
    class StateAccumulator extends AccumulatorV2[YearPopulation, YearPopulation] { 
    
          private var year = 0 //연도
          private var population:Long = 0L //인구
     
    //인구나 연도가 0이면 true, 그렇지 않으면 false
          override def isZero: Boolean = year == 0 && population == 0L
     
    //누산기를 복사하고 새로운 누산기 리턴
          override def copy(): StateAccumulator = {  
               val newAcc = new StateAccumulator  
               newAcc.year =     this.year  
               newAcc.population = this.population  
               newAcc 
           }
    
    //주와 인구를 0으로 재설정
           override def reset(): Unit = { year = 0 ; population = 0L }
     
    //누산기에 값 추가
           override def add(v: YearPopulation): Unit = { 
               year += v.year 
               population += v.population 
           }
     
    //2개의 누산기 병합
           override def merge(other: AccumulatorV2[YearPopulation, YearPopulation]): Unit={  
               other match {               
                   case o: StateAccumulator => {     
                           year += o.year 
                           population += o.population    
                   }    
                   case _ =>   
               } 
            }
     
    //누산기 값을 접근하기 위해서 스파크에서 호출할 수 있는 함수
           override def value: YearPopulation = YearPopulation(year, population)
    }
    
    
    scala> val statePopAcc = new StateAccumulator
    scala> sc.register(statePopAcc, "statePopAcc")
    scala> val statesPopulationRDD = sc.textFile("/user/irteamsu/input/statePopulation.csv").filter(_.split(",")(0) != "State")
    
    scala> statesPopulationRDD.map(x => {
         val toks = x.split(",")
         val year = toks(1).toInt
         val pop = toks(2).toLong
         statePopAcc.add(YearPopulation(year, pop)) x
         }).count
    ```
    
        