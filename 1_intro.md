## 1. 스칼라 소개



- 실행

 - REPL(read, eval, print, loop)
 - sc(스크립트) 파일은 변수나 함수등을 클래스로 감쌀 필요가 없는 매우 간단한 작업을 위해 사용한다.
 - 스크립트 파일을 실행하려면 익명 객체로 감싸서 처리하는 일종의 트릭(trick)이 필요.


  - 스크립트 실행하기
  
    ```sh
    $scala
    scala> load: src/main/scala/progscala2/introscala/upper1.sc
    ```
    
  - Scala 파일을 scalac로 컴파일 한 후에 생성된 class파일을 실행 (편법)

  - ```sh
    # 컴파일
    scalac src/main/scala/progscala2/introscala/upper1.scala
    
    # 실행
    scala -cp . progscala2.introscala.Upper hello world!
    > HELLO WORLD!
    ```

    

  - SBT로 실행 

    ```sh
    #따로 컴파일 할 필요가 없음
    run-main progscala2.introscala.Upper hello world!
    > HELLO WORLD!
    ```

    

  - scala 명령에 스칼라 소스코드를 넘기면 컴파일을 한다.

  - scala 명령에 main이 포함된 JAR파일을 넘기거나 클래스파일의 이름을 넘기면 그 파일을 실행한다.



- 예제

  - ToUpperClase(), mkString()

  ```scala
  object Upper2 {
    def main(args: Array[String]) = {
      val output = args.map(_.toUpperCase()).mkString("[", ", ", "]");
      println(output)
    }
  }
  
  > run-main progscala2.introscala.Upper2 hello world! asdf asdf
  > [HELLO, WORLD!, ASDF, ASDF]
  ```

  

- 동시성

  - **액터**라는 직관적인 모델을 사용하여 작성하는 **아카(AKKA)** API에 주목!

  - 액터모델

    - 서로 아무것도 공유하지 않는 액터라는 독립적인 소프트웨어 요소 (thread-safe)

      