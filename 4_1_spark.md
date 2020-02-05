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

