## 5. 스파크 

- 서버 : dev-shop-collection-nn-ncl

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
      scala> val rdd_two = sc.textFile("wiki.txt")
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

    