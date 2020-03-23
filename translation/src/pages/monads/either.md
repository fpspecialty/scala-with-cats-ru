## Either

Рассмотрим ещё одну полезную монаду:
`Either` из стандартной библиотеки Scala.
В Scala версии 2.11 и ниже
многие не считали `Either` монадой,
так как для этого типа не были определены методы `map` и `flatMap`.
Однако в Scala 2.12 `Either` стал *право-ориентированным (right biased)*.

### Лево-ориентированный и право-ориентированный Either

В Scala 2.11 `Either` не имел методов
`map` или `flatMap`.
Из-за этого `Either`
был неудобен для использования в for-выражениях.
Приходилось вставлять `.right`
в каждой конструкции:

```tut:book:silent
val either1: Either[String, Int] = Right(10)
val either2: Either[String, Int] = Right(32)
```

```tut:book
for {
  a <- either1.right
  b <- either2.right
} yield a + b
```

В Scala 2.12, `Either` был переосмыслен.
Было решено, что правая часть означает некий «успешный результат»,
и благодаря этому даны прямые определения `map` и `flatMap`.
А использование в for-выражениях стало гораздо приятнее:

```tut:book
for {
  a <- either1
  b <- either2
} yield a + b
```

в Cats реализована обратная совместимость этого поведения для Scala 2.11
через импорт `cats.syntax.either`,
позволяющая использовать право-ориентированный `Either`
во всех поддерживаемых версиях Scala.
В Scala 2.12+ можно как опускать этот импорт,
так и использовать его:

```tut:book:silent
import cats.syntax.either._ // for map and flatMap

for {
  a <- either1
  b <- either2
} yield a + b
```

### Создание экземпляров

В дополнение к созданию экземпляров `Left` и `Right` напрямую,
мы так же можем импортировать методы расширения `asLeft` и `asRight`
из [`cats.syntax.either`][cats.syntax.either]:

```tut:book:silent
import cats.syntax.either._ // for asRight
```

```tut:book
val a = 3.asRight[String]
val b = 4.asRight[String]

for {
  x <- a
  y <- b
} yield x*x + y*y
```

Эти смарт-конструкторы
лучше, чем `Left.apply` и `Right.apply`,
потому что они производят значения типа `Either`,
а не `Left` или `Right`.
Это помогает избежать ошибок механизма вывода типов,
связанных с чрезмерно узким определением (over-narrowing),
как в примере ниже:

```tut:book:fail
def countPositive(nums: List[Int]) =
  nums.foldLeft(Right(0)) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

Этот код не компилируется по двум причинам:

1. компилятор выводит тип аккумулятора
   как `Right`, а не как `Either`;
2. мы не указали типовый параметр для `Right.apply`,
   поэтому компилятор выводит тип левого параметра как `Nothing`.

Использование `asRight` решает обе проблемы.
`asRight` возвращает тип `Either`,
и мы можем полностью указать тип
всего одним типовым параметром:

```tut:book:silent
def countPositive(nums: List[Int]) =
  nums.foldLeft(0.asRight[String]) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

```tut:book
countPositive(List(1, 2, 3))
countPositive(List(1, -2, 3))
```

`cats.syntax.either` добавляет
несколько полезных методов расширения
в объект-компаньон `Either`.
Методы `catchOnly` и `catchNonFatal`
великолепны для захвата `Exceptions`
в экземпляры `Either`:

```tut:book
Either.catchOnly[NumberFormatException]("foo".toInt)
Either.catchNonFatal(sys.error("Badness"))
```

Также добавлены методы для конструирования `Either`
из других типов данных:

```tut:book
Either.fromTry(scala.util.Try("foo".toInt))
Either.fromOption[String, Int](None, "Badness")
```

### Преобразования Either

Ещё `cats.syntax.either` добавляет
несколько полезных методов для экземпляров `Either`.
Можно использовать `orElse` и `getOrElse` для вытаскивания
значений из правой части или получения дефолтного значения:

```tut:book:silent
import cats.syntax.either._
```

```tut:book
"Error".asLeft[Int].getOrElse(0)
"Error".asLeft[Int].orElse(2.asRight[String])
```

С помощью метода `ensure`
можно проверить, удовлетворяет ли значение правой части
предикату:

```tut:book
-1.asRight[String].ensure("Must be non-negative!")(_ > 0)
```

С помощью методов `recover` и `recoverWith`
можно обработать ошибки аналогично `Future`:

```tut:book
"error".asLeft[Int].recover {
  case str: String => -1
}

"error".asLeft[Int].recoverWith {
  case str: String => Right(-1)
}
```

Методы `leftMap` и `bimap` дополняют метод `map`:

```tut:book
"foo".asLeft[Int].leftMap(_.reverse)
6.asRight[String].bimap(_.reverse, _ * 7)
"bar".asLeft[Int].bimap(_.reverse, _ * 7)
```

Метод `swap` меняет местами левую и правую часть:

```tut:book
123.asRight[String]
123.asRight[String].swap
```

Наконец, Cats определяет множество методов для преобразований:
`toOption`, `toList`, `toTry`, `toValidated`, и так далее.

### Обработка ошибок

`Either` часто используется для fail-fast обработки ошибок.
Обычно мы строим цепочку вычислений через `flatMap`.
Если какое-нибудь вычисление выдаст ошибку,
оставшиеся вычисления не запустятся:

```tut:book
for {
  a <- 1.asRight[String]
  b <- 0.asRight[String]
  c <- if(b == 0) "DIV0".asLeft[Int]
       else (a / b).asRight[String]
} yield c * 100
```

При использовании `Either` для обработки ошибок
нам надо определить,
каким типом представлять ошибки.
Можно было бы использовать `Throwable`:

```tut:book:silent
type Result[A] = Either[Throwable, A]
```

Получается семантика, аналогичная `scala.util.Try`.
Но есть проблема: `Throwable`
- слишком широкий тип.
Мы (почти) не сможем определить, какая именно ошибка произошла.

Другой подход - сделать алгебраический тип
для представления возможных ошибок:

```tut:book:silent
object wrapper {
  sealed trait LoginError extends Product with Serializable

  final case class UserNotFound(username: String)
    extends LoginError

  final case class PasswordIncorrect(username: String)
    extends LoginError

  case object UnexpectedError extends LoginError
}; import wrapper._
```

```tut:book:silent
case class User(username: String, password: String)

type LoginResult = Either[LoginError, User]
```

Этот подход решает проблемы, возникающие с `Throwable`.
Получается фиксированное множество ожидаемых типов ошибок
и catch-all для остальных неучтенных ошибок.
Также мы получаем полную проверку
в pattern matching:

```tut:book:silent
// Choose error-handling behaviour based on type:
def handleError(error: LoginError): Unit =
  error match {
    case UserNotFound(u) =>
      println(s"User not found: $u")

    case PasswordIncorrect(u) =>
      println(s"Password incorrect: $u")

    case UnexpectedError =>
      println(s"Unexpected error")
  }
```

```tut:book
val result1: LoginResult = User("dave", "passw0rd").asRight
val result2: LoginResult = UserNotFound("dave").asLeft

result1.fold(handleError, println)
result2.fold(handleError, println)
```

### Упражнение: Какой же вариант лучший?

Является ли стратегия обработки ошибок в предыдущих примерах
подходящей для всех случаев?
Что ещё может понадобиться в процессе обработки ошибок?

<div class="solution">
Это открытый вопрос.
Также это trick question---ответ
зависит от нужной семантики.
Пункты для размышления:

- Восстановление после ошибок важно при выполнении больших задач.
  Мы не хотим, чтобы после исполнения задачи длиной в день
  возникла ошибка на последнем этапе.

- Репортинг ошибок не менее важен.
  Нужно знать, что именно пошло не так,
  а не просто факт ошибки.

- В ряде случаев нужно собрать все ошибки,
  а не только первые случившиеся.
  Типичный пример - валидация веб-форм.
  Намного лучше
  указать на все ошибки пользователю во время отправки формы,
  чем показывать их по одной.
</div>
