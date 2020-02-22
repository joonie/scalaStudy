## 스파크스트림

- [참고문서](http://spark.apache.org/docs/2.3.2/streaming-programming-guide.html)
- 최소 한번 처리 방식
  - 이벤트가 실제로 처리되고 결과가 어딘가에 저장된 후에 마지막 이벤트의 위치를 저장
  - 이벤트 offset(?)을 저장하기 때문에 재시작시 바로 이전 이벤트부터 실행한다.
  - 잠재적인 중복 이슈가 발생할 수 있음. (최소 한번은 처리됨)
  - 사용 예시
    - 지속적으로 뭔가를 보여주는 주식가격표시기나 측정을 하는 경우 적합하지 않음.
    - 누적 합계, 집계등 정확성을 중요시하는 처리에는 적합하나 중복될 경우 잘못 나올 수 있음
  - 처리 순서
    - 결과를 저장한다
    - 오프셋을 저장한다
- 최대 한번 처리 방식
  - 이벤트 처리전에 수신된 마지막 이벤트의 위치를 저장
  - 실패하더라도 저장된 이벤트의 다음 이벤트부터 처리한다 (저장된 이전 이벤트는 읽지 않음)
  - 이벤트가 한번만 처리되거나 안될수도 있다
  - 사용 예시
    - 지속, 순간적인 값을 보여주는데 적합 (주식가격표시기나 측정)
    - 정확도가 떨어지기 때문에 집계에는 부적합
  - 처리 순서
    - 오프셋을 저장한다 (오프셋이 이미 저장되었기 때문에 결과가 저장되지 않으면 스킵됨)
    - 결과를 저장한다
- 정확히 한번 처리 방식
  - 최소 한번 처리방식과 유사하다 중복이벤트를 제거함
  - 중복이 되면 안되는 집계 연산에 유리
  - 처리 순서
    - 결과를 저장한다
    - 오프셋을 저장한다
- 스파크스트림에서는 sparkContext 대신에 이와 비슷한 StreamingContext를 사용한다.
  - StreamingContext는 배치 간격에 대한 시간이나 기간을 지정해야한다.
  - 애플리케이션이 한번 실행하면 중지된 애플리케이션을 재시작할 수 없다.
  - 재실행이 필요한 경우 새로운 스트리밍컨텍스트를 생성해야한다.



- 스파크 실행

  ```scala
    val conf = new SparkConf().setMaster("local[1]").setAppName("StreamBasicV1")
    val ssc = new StreamingContext(conf, Seconds(10))
  
    //스트림 시작
    ssc.start()
  
    //false : 즉시 중지, (ture, ture)를 넣으면 모든 데이터가 처리되었는지 확인
    ssc.stop(false)
  ```



- 입력 스트림
  - **<중요>** 기존의 파일은 적용 대상에서 제외한다. (신규 추가된 파일만 적용됨)
  - socketTextStream
    - TCP 통신(호스트이름:포트)로 입력스트림을 생성한다.
    - TCP 소켓으로 데이터가 수신되고 바이트는 \n 구분자 라인으로 인코딩된 UTF-8로 해석된다.
  - rawSocketStream
    - 네트워크 소스인 호스트이름:포트로 입력스트림을 생성한다.
    - 역직렬화된 데이터 블록을 수신받기 때문에 socketTextStream보다 효율적이다.
  - fileStream
    - Key-value 타입과 입력 포맷을 사용해 입력 스트림을 받는다.
    - 처리하고자 하는 파일을 source 경로에 넣음으로써 시작되는 듯.
  - textFileStream
    - 해당 경로의 텍스트파일을 읽는 입력 스트림을 받는다.
    - 입력스트림은 LongWritable을 키로, Text를 값으로, TextinputFormat을 입력 포맷으로 사용한다.
  - binaryRecordsStream
    - 바이너리 파일을 읽는다
  - queueStream
    - RDD큐에서 입력 스트림을 생성한다.
    - 각 배치는 큐에서 리턴하는 하나 또는 모든 RDD를 처리한다.



- 불연속 스트림
  - 스파크 스트리밍은 불연속 스트림 또는 DStream이라는 추상화를 기반으로 생성된다.
  - DStream은 RDD 시퀀스로 표현되며 개별시간마다 개별 RDD가 생성된다.



- 스트리밍 에플리케이션 작성 과정
  - SparkContext에서 StreamingContext를 생성
  - StreamingContext에서 DStream을 생성
  - 각 RDD에 적용될 수 있는 트랜스포메이션과 액션을 제공
  - StreamingContext.start