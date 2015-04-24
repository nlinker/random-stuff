# Почему сhecked exceptions в Java не работают
... и какая альтернатива им в Scala

## Когда они нужны
Один человек достаточно ёмко [объяснил](http://stackoverflow.com/a/19061110) случай, когда их имеет смысл использовать
> Checked Exceptions should be used to declare for __expected__, but __unpreventable__ errors that are __reasonable to recover from__.

Увы, это достаточно узкий случай, который волей случая натянулся на создание вообще любого софта в Java.

## В чём проблема?

1. Это экспериментальная фича, и java - единственный язык, который имеет её. В то время никто ещё не думал, что java будет везде, и была иллюзия, что если что-то окажется плохо, то фичу можно будет быстренько выпилить.

1. Исключения становятся частью интерфейса, что неправильно - разные реализации могут кардинально различаться по поводу того, какие исключения они будут бросать.

1. Необходимо постоянно оборачивать библиотечные исключения в исключения своего приложения, что раздувает количество рутинного кода.

1. throws Exception просачиваются везде в стандартном API, что фактически не даёт никакого профита, а только засоряет как само API, так и будущие реализации (Например, AutoCloseable, Callable):
    ```java
    public interface AutoCloseable {
        void close() throws *Exception*;
    }
    public interface Callable<V> {
        V call() throws *Exception*;
    }
    ```
    Аналогично, весь код jdbc насквозь пронизан java.sql.SQLException, что не даёт ничего, кроме необходимости создавать рутинный код.

1. Невозможно написать правильный throws для функций высшего порядка (не годится даже бесполезный Exception):
    ```java
        interface Func<T, R> { R apply(T t) throws *???*; }

        List<R> map(List<T> list, Func<T, R> f) throws *???* {}
    ```
    Если `f` аннотирован исключением, то это же исключение должно быть и у `map` (снова `Exception`?) и вдобавок семантика `map` становится намного сложнее - ведь процесс уже может прерваться в любой момент, что вызывает вопрос "А как тогда корректно обработать это исключение?" порождая вопросы вроде доступа к уже обработанным и ещё не обработанным элементам и прочее. Непоследний момент ещё и в том, что `map` становится непараллелизуемой. Если же аннотации различаются, то данные функции (`map` и `f.apply`) вообще невозможно сочетать. Единственный выход - избавиться вообще от `throws`, и сделать функцию `map` чистой, а исключительные ситуации возложить на код где-то наверху.

1. Характерен пример с интерфейсом Appendable в JDK:
    ```java
    Appendable.append(CharSequence csq) throws IOException;
    ```
    Что говорит этот `throws`? Каждый раз, когда я добавляю строку к чему-нибудь (билдеру, логам, консоли) я должен ловить это `IOException`. Почему? Потому что эта операция _теоретически_ может совершать IO (`Writer` тоже реализует `Appendable`). Следовательно в каждом месте я должен делать
    ```java
    try {
      log.append(message)
    }
    catch (IOException e) {
      // swallow it
    }
    ```
    Да, придётся проглатывать исключение, потому что никакого другого разумного способа его обработать нет.

1. Аналогичная проблема вообще с любым кодом общего назначения. Например, коллбэки также невозможно аннотировать нужным исключением:
    ```java
        class Component implements Listenable {
            Listener getListener() {...}
            interface Listener {
                void changed(Component c) throws *???*;
            }
        }

        /** библиотека общего назначения, предполагается,
            что она будет использоваться для кучи проектов */
        class Library {
            Component c = ...;
            void genericMethod() throws *???* {
                if (...) с.getListener().changed(c);
            }
        }
    ```
    Никакого разумного исключения, кроме `Exception`, вместо *???* поставить не получится, но если воткнуть `Exception`, то снова получим необходимость создания большого количества рутинного кода.

1. Если для кого-то имеют значения мнения авторитетов, то вот:

   [Bruce Eckel](http://www.mindview.net/Etc/Discussions/CheckedExceptions)
> Examination of small programs leads to the conclusion that requiring exception specifications could both enhance developer productivity and enhance code quality, but experience with large software projects suggests a different result – decreased productivity and little or no increase in code quality.

   [Rod Waldhoff](http://radio-weblogs.com/0122027/stories/2003/04/01/JavasCheckedExceptionsWereAMistake.html)
> Checked exceptions are pretty much disastrous for the connecting parts of an application's architecture however. These middle-level APIs don't need or generally want to know about the specific types of failure that might occur at lower levels, and are too general purpose to be able to adequately respond to the exceptional conditions that might occur.

   [Anders Hejlsberg](http://www.artima.com/intv/handcuffs.html)
> I see two big issues with checked exceptions: scalability and versionability.

## Как правильно?

В Java все исключения обернуть в RuntimeException и создать общий код где-то наверху, который корректно и полно обрабатывает все исключения. Так сделано в Tapestry, Spring, Joda-time и наверное ещё многих других фреймворках.

В Скале можно использовать scala.util.Try, сделать алгебраический тип, аннотировать метод исключениями или (есть совершенно фатальные причины) использовать соответствующий плагин для Scala.

### Использовать Try

```scala
def perform(input: String): Try[...] = {
  for {
    a <- Try(operation1(input))
    b <- Try(operation2(a))
    c <- Try(operation3(b))
  } yield doSomethingWith(a, b, c)
}
```
Если какая-нибудь из operation1..3 кинет исключение, оно будет обёрнуто в Failure, если же всё будет нормально, то вызов doSomethingWith(a, b, c) будет обёрнут в Success. И потом это значение можно использовать как обычное значение, например, преобразовать в нужный json-чик.

### Создать свой тип, и разбирать его на каждом шаге

```scala
  sealed trait MyResult
  case class  Success(value: Int) extends MyResult
  case class  FileNotFound(file: java.io.File) extends MyResult
  case object IllegalState extends MyResult

  def perform(input: String): Int = {
    val r = operation1(input) match {
      case Success(a) => operation2(a) match {
        case Success(b) => operation3(b) match {
          case Success(c) => doSomethingWith(a, b, c)
          case failure ⇒ failure
        }
        case failure ⇒ failure
      }
      case failure ⇒ failure
    }
    r match {
      case Success(a) ⇒ a
      case FileNotFound(file) ⇒
        Console.println(s"File $file not found")
        0  // default value
      case IllegalState ⇒
        Console.println(s"The illegal state encountered")
        0  // default value
    }

  }

  def operation1(input: String): MyResult = ???
  def operation2(a: Int): MyResult = ???
  def operation3(b: Int): MyResult = ???
  def doSomethingWith(a: Int, b: Int, c: Int): MyResult = ???
```

### Аннотировать (эта возможность сделана для совместимости с Java):

```scala
    @throws(classOf[IOException])
    @throws(classOf[LineUnavailableException])
    @throws(classOf[UnsupportedAudioFileException])
    def playSoundFileWithJavaAudio {
      // exception throwing code here ...
    }
```
Вызов этого метода из Java будет неотличим от throws аннотации в Java-методах.

### Для Scala есть наконец плагин для компилятора, который реализует checked exceptions:

[No Exceptions: Checked Exceptions for Scala](https://opensource.plausible.coop/src/projects/SNX/repos/nx/browse)

----

Источники:

http://java.dzone.com/articles/tragedy-checked-exceptions
http://radio-weblogs.com/0122027/stories/2003/04/01/JavasCheckedExceptionsWereAMistake.html
http://stackoverflow.com/questions/10818427/is-either-the-equivalent-to-checked-exceptions
http://stackoverflow.com/questions/27578/when-to-choose-checked-and-unchecked-exceptions
http://programmers.stackexchange.com/questions/177806/decision-for-unchecked-exceptions-in-scala
http://stackoverflow.com/questions/613954/the-case-against-checked-exceptions
https://opensource.plausible.coop/src/projects/SNX/repos/nx/browse
