## 2. 스칼라의 특징

- 변수의 정의

  - val

    - val로 지정하는 스칼라의 변수는 불변이다. variable(var)이 아니라 val임을 기억.

      ```scala
      > var array: Array[String] = new Array(5); //초기화 필수!
      > array = new Array(2) // ERROR
      > array(0) = "Hello"
      > array
      Array[String] = Array(Hello, null, null, null, null)
      ```

    - val은 선언 시 반드시 초기화해야 한다.

  - var

    - 변경이 가능하므로 나중에 바꿀 수 있다

    - var도 val과 마찬가지로 반드시 선언할 때 초기화 해줘야한다.

      ```scala
      > var stockPrice: Double = 100.0 //초기화 필수!
      > stockPrice = 200.0 (OK) //값 자체는 바꿀 수 있지만 stockPrice가 가르키는 객체는 불변
      ```

  - 생성자 매개변수에 val이나 var을 사용할 수 있다. 이때 val은 변경 불가능, var은 변경가능 필드다.

    ```scala
    > class Person(val name: String, var age: Int)
    > val p = new Person("test", 31)
    > p.name = "change" //ERROR
    > p.age = 32 //OK
    ```

- 부분 함수

  - 부분이란 모든 입력에 대해 결과를 정의하지 않음을 의미한다.

  - ```scala
    //String과 일치하는 함수
    var pf1: PartialFunction[Any, String] = {case s:String => "YES"}
    //Double와 일치하는 함수 
    var pf2: PartialFunction[Any, String] = {case d:Double => "YES"}
    //두 함수를 묶어서 String과 Double에 모두 일치하는 새 부분함수 생성
    val pf = pf1 orElse pf2
    
    //부분함수를 호출하고 발생하는 MatchError를 잡아내는 함수
    //성공 여부와 관계없이 문자열 반환
    def tryPf(x: Any, f:PartialFunction[Any, String]) : String = 
    	try { 
        f(x).toString
      } catch {
        case _: MatchError => "ERROR!" 
      }
    
    //isDefinedAt을 호출해서 문자열 결과를 반환하는 함수
    def d(x: Any, f: PartialFunction[Any, String]) =
    	f.isDefinedAt(x).toString
    
    println("      |   pf1 - String  |   pf2 - Double  |    pf - All")   // <6>
    println("x     | def?  |  pf1(x) | def?  |  pf2(x) | def?  |  pf(x)")
    println("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    List("str", 3.14, 10) foreach { x =>
      printf("%-5s | %-5s | %-6s  | %-5s | %-6s  | %-5s | %-6s\n", x.toString, 
        d(x,pf1), tryPF(x,pf1), d(x,pf2), tryPF(x,pf2), d(x,pf), tryPF(x,pf))
    }
    ```

    ```sh
    > 결과
    			|   pf1 - String  |   pf2 - Double  |    pf - All
    x     | def?  |  pf1(x) | def?  |  pf2(x) | def?  |  pf(x)
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    str   | true  | YES     | false | ERROR!  | true  | YES
    3.14  | false | ERROR!  | true  | YES     | true  | YES
    10    | false | ERROR!  | false | ERROR!  | false | ERROR!
    ```

  - x가 "str" 문자열 일 때

    - d(x, pf1) : pf1.isDefinedAt을 호출한다 -> true
    - tryPf(x, pf1) : 인자로 전달한 pf1의 결과를 호출한다. -> YES
    - d(x, pf2) : pf2.isDefinedAt을 호출한다 -> false
    - tryPf(x, pf2) : 인주로 전달한 pf2의 결과를 호출한다 -> 타입 매치 실패로 exception catch -> ERROR

    

- 메소드 선언

  ```scala
  case class Point(x: Double = 0.0, y: Double = 0.0) { //Point의 기본 초기값을 지정한다
  
    def shift(deltax: Double = 0.0, deltay: Double = 0.0) =
      copy (x + deltax, y + deltay) //shift는 케이스클래스에서 자동으로 생성하는 copy 메소드를 활용함
  }
  
  abstract class Shape() {
    /**
     * 두 인자 목록을 받는다.
     * 1번 인자 : 그림을 그릴 때, x, y 축 방향으로 이동시킬 오프셋 값이고,
     * 2번 인자 : 함수인자
     */
    def draw(offset: Point = Point(0.0, 0.0))(f: String => Unit): Unit =
      f(s"draw(offset = $offset), ${this.toString}")
  }
  
  case class Circle(center: Point, radius: Double) extends Shape
  
  case class Rectangle(lowerLeft: Point, height: Double, width: Double)
    extends Shape
  
  ```

  - 인자를 두개 가진 메소드 선언

  - ```scala
    def draw(offset: Point = Point(0.0, 0.0))(f: String => Unit): Unit =
        f(s"draw(offset = $offset), ${this.toString}")
    
    //1번_ 기본 사용법
    draw(Point(1.0, 2.0))(str => println(s"ShapesDrawingActor : $str"))
    
    //2번_ 인자를 둘러싼 괄호('(')를 중괄호('{')로 바꿀 수 있음
    draw(Point(1.0, 2.0)){str => println(s"ShapesDrawingActor : $str")}
    
    //3번_ 2번을 보기좋게 바꿀 수 있음 
    draw(Point(1.0, 2.0)) {
      str => println(s"ShapesDrawingActor : $str")
    }
    
    //4번_ 기본 생성자를 사용하는 경우
    draw() {
      str => println(s"ShapesDrawingActor : $str")
    }
    
    //5번_ JAVA 스타일
    draw(Point(1.0,2.0), str => println(s"ShapesDrawingActor : $str"))
    
    //6번_ 5번 스타일에서 기본 생성자를 사용하려는 경우
    draw(f = str => println(s"ShapesDrawingActor : $str"))
    
    ```

  - 즉 위의 draw 함수는 인자가 2개이며 두번째 중괄호도 인자임.




- Future

  - 아카가 Future를 사용하지만 액터의 모든 기능이 필요하지 않은 경우 Future만 별도로 사용할 수 있다.

  - 구행할 작업 중 일부를 Future로 감싸면 그 작업을 비동기적으로 수행하며 Future API는 결과가 준비된 경우 콜백을 호출해주는 등 결과를 처리할 수 있는 다양한 방법을 제공한다.

    ```scala
    def sleep(millis: Long) = {
      Thread.sleep(millis)
    }
    
    // 쓸모는 없는데 바쁜 일 ;)
    def doWork(index: Int) = {
      sleep((math.random * 1000).toLong)
      index
    }
    
    (1 to 5) foreach { index =>
      val future = Future { //Future.apply에 함수를 전달한다.
        doWork(index) //별도 쓰래드에게 할당한다
      }
      future onSuccess { //partialFunction
        case answer: Int => println(s"Success! returned: $answer")
      }
      future onFailure { //partialFunction
        case th: Throwable => println(s"FAILURE! returned: $th")
      }
    }
    
    sleep(1000)  // '작업' 이 끝날 때까지 충분히 기다린다.
    println("Finito!")
    
    > 결과는 1~5가 섞여서 찍힘.
    ```



- 타입 추론

  - jdk 7부터 지원이 되는 다이아몬드 연산자와 비슷하게 처리 가능

  - ```scala
    1. val map: HashMap[String, Object] = new HashMap
    2. val map = new HashMap[String, Object]
    
    위의 1,2번은 동일하다
    ```

    

  - 명적으로 타입을 지정해야하는 경우

    - var나 val로 변수 선언시 값을 대입하지 않는 경우

      - ex. var name**: String**, val age**: Int**

    - 매개변수

      - ex. def depositFunction(amount**: Money**) = {...}

    - 메서드 안에서 return을 명시적으로 호출하는 경우

    - 메서드가 재귀적인 경우

    - 오버로딩한 둘 이상의 메서드가 있고 그중 한 메서드가 다른 메서드를 호출하는 경우 호출하는 메서드에는 반환타입을 표기해야함

    - ```scala
      object stringUtils {
        //인자의 개수가 변할 수 있는 메소드
        //String*는 0 또는 그 이상의 String이라는 의미
        def joiner(strings: String*): String = string.mkString("-") 
      
        //List를 인자로 받는 메소드
        def joiner(strings: List[String]) = joiner(strings :_*) //컴파일 오류
        def joiner(strings: List[String]): String = joiner(strings :_*) //ok
      }
      ```

      > strings :_*란 컴파일러에게 주는 힌트.*_
      >
      > '_* strings'라는 리스트를 가변 인자목록(*)으로 다루되 타입은 알 수 없지만 추론한 타입(:_)을 사용하라는 뜻.
      >
      > ':' 와 '_*' 는 띄어써도 되지만 '\_'와 '\*'는 반드시 붙여써야한다.

    

    - 컴파일러가 추론한 타입이 우리의 의도보다 더 일반적인 경우 (ex. Any)



- 흔히 하는 실수

  - ```scala
    scala > def double(i: Int) {2 * i}
    scala > println(double(2))
    > 결과는 ()
    
    scala > def double(i: Int) = {2 * i} //메서드 앞에 반드시 등호(=)를 붙여줘야한다.
    scala > println(double(2))
    > 결과는 4
    ```

    - 만약 메서드 앞에 등호가 없다면 스칼라는 프로시저로 인식한다.

- 예약어

  | 예약어  | 설명                                                         |
  | ------- | :----------------------------------------------------------- |
  | case    | 매치식에서 케이스 절을 시작한다, 케이스 클래스를 정의한다    |
  | def     | 메서드 선언을 시작한다                                       |
  | forSome | 사용할 수 있는 구체적인 타입을 제한하기 위한 존재 타입 선언에 사용한다 |
  | lazy    | val의 계산을 늦춘다                                          |
  | object  | 싱글턴 선언을 시작한다.                                      |
  | sealed  | 부모 타입에 적용한다. 그 타입의 모든 파생 타입이 같은 소스 파일에 선언되어야 한다 |
  | trait   | 상태나 동작을 새로 추가하는 믹스인모듈이다.  또한 자바 인터페이스처럼 메서드를 선언하되 정의는 하고 싶지 않은 경우에도 사용한다 |

  