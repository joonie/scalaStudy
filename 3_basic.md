## 3. 스칼라 기초


- **Scastie**에서 scala를 테스트해볼수도 있음

  

- 연산자 오버로딩

  - 1 + 2 는 1 .+ (2) 와 동일하다.
  - 마침표는 별표(*)보다 우선순위가 높다.
    - 1 + 2 * 3 은 7이지만 1 .+ (2) * 3 은 9이다. 

- 리스트 사용

  ```scala
  1. val list = List('a', 'b', 'c') // List[Char] = List(a, b, c)
  2. 'd' :: list // List[Char] = List(a, b, c, d)
  
  3. List(1, 2, 3).size() //List(1, 2, 3).size는 틀림.
  
  4. //아래 4가지는 인자(i)가 1개일 때 모두 결과가 같다
  List(1, 2, 3, 4).filter((i: Int) => isEven(i)).foreach((i: Int) => println(i));
  List(1, 2, 3, 4).filter(i => isEven(i)).foreach(i => println(i));
  List(1, 2, 3, 4).filter(isEven).foreach(println());
  List(1, 2, 3, 4).filter isEven foreach println
  
  ```

- if


  - 스칼라의 if는 결과를 돌려준다

  ```scala
  val configFilePath = if(configFile.exists()) {
    configFile.getAbsolutePath();
  } else {
    configFile.createNewFile();
    configFile.getAbsolutePath();
  }
  ```


  - 3항연산자는 지원하지 않는다

- for와 가드(guard), yield


  - 스칼라의 for는 break나 continue를 지원하지 않는다. 가드나 더 나은구조로 처리해야함.

  ```scala
  for(breed <- dogBreeds) { //dogBreeds의 리스트의 원소를 하나씩 가져와 breed에 넣음
  	println(breed);
  }
  
  for(breed <- dogBreeds) {
    //guard라 부르며 dogBreeds의 원소에서 "terrier"를 포함한 원소를 리턴한다.
    if breed.contains("terrier") 
    //"yorkshire"가 포함된 원소를 제외한다
    if !breed.startsWith("yorkshire")
  } println(breed) 
  
  //위의 가드를 '&&' 등으로 묶어서 처리할 수 있다
  for(breed <- dogBreeds) {
    if breed.contains("terrier") && !breed.startsWith("yorkshire")
  }
  
  //yield는 타입을 추론하여 이에 맞는 (ex. List[String])으로 리턴해준다. (.collect()와 유사)
  val filteredBreed = for {
    breed <- dogBreeds
    if breed.contains("terrier") && !breed.startsWith("yorkshire")
  } yield breed
  
  
  ```

  > For 에 식이 한개만 들어가면 괄호('(', ')')를 여러식이 들어가면 중괄호를 사용하는 것이 관례
  >
  > 괄호를 사용하려면 세미콜론(;)을 추가해야한다.



- Some(Optional)

  ```scala
  val dogBreeds = List(Some("AAA"), Some("BBB"), Some("CCC"))
  
  for {
    breedOption <- dogBreeds
  //  if breedOption != None Some을 사용하면 이곳에 None 체크를 하는 것과 동일하다
    breed <- breedOption
    upcaseBreed = breed.toupperCase()
  } println (upcaseBreed)
  
  //패턴매칭을 통해서 위와 동일한 결과를 내는 방법
  for {
    Some(breed) <- dogBreeds //breedOption이 Some인 경우만 성공
    upcaseBreed = breed.toupperCase()
  } println (upcaseBreed)
  
  
  ```

  

- 자바와 연산자의 차이

  - 대부분의 연산자는 자바와 동일
  - '=='와 '!='는 자바와 다름
    - 자바
      - 값을 비교하지 않고 객체 레퍼런스를 비교
      - 값을 비교하고 싶으면 equals
    - 스칼라
      - 값을 비교
      - 레퍼런스를 비교하고 싶으면 eq 메소드 사용

- 스칼라 try-catch (feat. 파일 인풋방법 file io)

  - 기본적으로 java와 try-catch 사용은 비슷함
  - 실행법

  ```sh
  scalac /Users/user/git/study/scala/scalaStudy/src/main/scala/progscala2/rounding/TryCatch.scala
  scala -cp . progscala2.rounding.TryCatch /Users/user/test1
  ```

  

  ```scala
  object TryCatch {
    /** 사용법: scala rounding.TryCatch 파일이름1 파일이름2 ... */
    def main(args: Array[String]) = {
      args foreach (arg => countLines(arg))                            // <1>
    }
  
    import scala.io.Source                                             // <2>
    import scala.util.control.NonFatal
  
    def countLines(fileName: String) = {                               // <3>
      println()  // 읽기 쉽게 하기 위해 빈 줄을 넣는다.
      var source: Option[Source] = None                                // <4>
      try {                                                            // <5>
        source = Some(Source.fromFile(fileName))                       // <6>
        val size = source.get.getLines.size
        println(s"file $fileName has $size lines")
      } catch {
        case NonFatal(ex) => println(s"Non fatal exception! $ex")      // <7>
      } finally {
        for (s <- source) {                                            // <8>
          println(s"Closing $fileName...")
          s.close
        }
      }
    }
  }
  ```

- 열거값

  - 자바의 enum과 관련이 없다.

  ```scala
  object Breed extends Enumeration {
    type Breed = Value //Value 대신에 Breed를 참고할 수 있게 하기 위한 일종의 별명(alias)
    val doberman = Value("Doberman Pinscher")
    val yorkie   = Value("Yorkshire Terrier")
    val scottie  = Value("Scottish Terrier")
    val dane     = Value("Great Dane")
    val portie   = Value("Portuguese Water Dog")
  }
  import Breed._
  
  // 품종과 품종의 ID 목록을 표시한다.
  println("ID\tBreed")
  for (breed <- Breed.values) println(s"${breed.id}\t$breed")
  
  /* 출력결과
  ID	Breed
  0	Doberman Pinscher
  1	Yorkshire Terrier
  2	Scottish Terrier
  3	Great Dane
  4	Portuguese Water Dog
  */
  
  // 테리어 품종의 목록을 표시한다.
  println("\nJust Terriers:")
  Breed.values filter (_.toString.endsWith("Terrier")) foreach println
  
  /* 출력결과
  Yorkshire Terrier
  Scottish Terrier
  */
  
  //type Breed = Value를 선언했기 때문에 Value.toString 대신 b.tostring을 사용할 수 있는 것
  def isTerrier(b: Breed) = b.toString.endsWith("Terrier")
  
  println("\nTerriers Again??")
  Breed.values filter isTerrier foreach println
  
  /* 출력결과
  Yorkshire Terrier
  Scottish Terrier
  */
  ```

  

- 문자열 인터폴레이션 ($ 사용)

  ```scala
  val name = "StarBucks"
  println(s"Hello, $name") //Hello StarBucks
  
  val percentFloat = (10000F / 600F) * 100
  println(f"result : ${percentFloat}%.1f%%") //%% 두번 쓰면 %를 출력함
  ```

  - '\$'를 사용하고 싶다면 '\$\$'를 사용하면 된다

  

- 인터페이스와 비슷한 trait

  ```scala
  -java style
  trait Logging {
    def info (message: String): Unit
    def warn (message: String): Unit
    def err (message: String): Unit
  }
  
  -scala style
  trait StdoutLogging extends Logging {
    def info (message: String) = println(s"INFO : $message")
    def warn (message: String) = println(s"WARN : $message")
    def err (message: String) = println(s"ERR : $message")
  }
  
  val service2 = new ServiceImportance("dos") with StdoutLogging {
    override def work(i: Int): Int = {
      info(s"Starting work: i = $i")
      val result = super.work(i)
      info(s"Ending work: i = $i, result = $result")
      result
    }
  }
  (1 to 3) foreach (i => println(s"Result: ${service2.work(i)}"))
  ```

  


