## Монада Writer {#writer-monad}

[`cats.data.Writer`][cats.data.Writer] —
это монада, которая позволяет нам вести журнал во время вычислений.
Мы можем использовать `Writer` для записи сообщений, ошибок 
или дополнительных данных о вычислениях 
и извлекать журнал вместе с конечным результатом.

Одним из распространенных применений `Writer` является 
запись последовательностей шагов в многопоточных вычислениях, 
где стандартные методы императивного ведения журнала 
могут приводить к чередующимся сообщениям из разных контекстов.
С `Writer` журнал вычислений привязан к результату,
поэтому мы можем запускать параллельные вычисления без смешивания журналов.

<div class="callout callout-info">
*Типы данных Cats*

`Writer` — это первый тип данных, который мы увидели 
в пакете [`cats.data`][cats.data.package].
Этот пакет предоставляет экземпляры различных тайпклассов, 
которые создают полезную семантику.
Другие примеры из `cats.data` включают 
монадные трансформеры, которые мы увидим в следующей главе,
и тип [`Validated`][cats.data.Validated], 
с которым мы столкнемся в главе [@sec:applicatives].
</div>

### Создание и распаковка Writer

`Writer[W, A]` содержит два значения:
*log* типа `W` и *result* типа `A`.
Мы можем создать `Writer` из значений каждого типа следующим образом:

```tut:book:silent
import cats.data.Writer
import cats.instances.vector._ // для Monoid
```

```tut:book
Writer(Vector(
  "It was the best of times",
  "it was the worst of times"
), 1859)
```

Обратите внимание, что тип, сообщаемый на консоли,
на самом деле `WriterT[Id, Vector[String], Int]`
вместо `Writer[Vector[String], Int]`, как мы могли ожидать.
Для повторного использования кода
Cats реализует `Writer` в терминах другого типа, `WriterT`.
`WriterT` - это пример новой концепции, называемой *монадный трансформер*,
о которой мы расскажем в следующей главе.

Попробуем пока проигнорировать эту деталь.
`Writer` является псевдонимом типа для `WriterT`,
поэтому мы можем читать `WriterT[Id, W, A]` как `Writer[W, A]`:

```scala
type Writer[W, A] = WriterT[Id, W, A]
```

Для нашего удобства, Cats предоставляет способ создания `Writer`
через определение только журнала или результата.
Если у нас есть только результат, мы можем использовать стандартный синтаксис `pure`.
Для этого у нас должен быть `Monoid[W]` в области видимости,
чтобы Cats знал, как создать пустой журнал:

```tut:book:silent
import cats.instances.vector._   // для Monoid
import cats.syntax.applicative._ // для pure

type Logged[A] = Writer[Vector[String], A]
```

```tut:book
123.pure[Logged]
```

Если у нас есть только журнал,
мы можем создать `Writer[Unit]`, используя синтаксис `tell`
из [`cats.syntax.writer`][cats.syntax.writer]:

```tut:book:silent
import cats.syntax.writer._ // для tell
```

```tut:book
Vector("msg1", "msg2", "msg3").tell
```

Если у нас есть и результат, и журнал,
мы можем использовать метод `Writer.apply`
или синтаксис `writer`
из [`cats.syntax.writer`][cats.syntax.writer]:

```tut:book:silent
import cats.syntax.writer._ // для writer
```

```tut:book
val a = Writer(Vector("msg1", "msg2", "msg3"), 123)
val b = 123.writer(Vector("msg1", "msg2", "msg3"))
```

Мы можем извлечь результат и журнал из `Writer`,
используя методы `value` и `writing` соответственно:

```tut:book
val aResult: Int =
  a.value
val aLog: Vector[String] =
  a.written
```

Мы можем извлечь оба значения одновременно, используя метод `run`:

```tut:book
val (log, result) = b.run
```

### Композиция и трансформация Writer

Лог в `Writer` сохраняется, когда мы вызываем `map` или `flatMap` над ним.
`flatMap` добавляет логи из исходного `Writer`
и результата пользовательской функции порядка.
По этой причине рекомендуется использовать тип журнала,
который эффективно реализует операции добавления и объединения,
такой как `Vector`:

```tut:book
val writer1 = for {
  a <- 10.pure[Logged]
  _ <- Vector("a", "b", "c").tell
  b <- 32.writer(Vector("x", "y", "z"))
} yield a + b

writer1.run
```

В дополнение к преобразованию результата с помощью `map` и` flatMap`,
мы можем преобразовать журнал `Writer` с помощью метода` mapWritten`:

```tut:book
val writer2 = writer1.mapWritten(_.map(_.toUpperCase))

writer2.run
```

Мы можем преобразовать как лог, так и результат одновременно, используя методы `bimap` или `mapBoth`.
Метод `bimap` принимает два параметра: для журнала и для результата.
Метод `mapBoth` принимает одну функцию, которая принимает два параметра:

```tut:book
val writer3 = writer1.bimap(
  log => log.map(_.toUpperCase),
  res => res * 100
)

writer3.run

val writer4 = writer1.mapBoth { (log, res) =>
  val log2 = log.map(_ + "!")
  val res2 = res * 1000
  (log2, res2)
}

writer4.run
```

Наконец, мы можем очистить журнал с помощью метода `reset`
и поменять местами результат и журнал с помощью метода` swap`:

```tut:book
val writer5 = writer1.reset

writer5.run

val writer6 = writer1.swap

writer6.run
```

### Упражнение: показать свою работу

`Writer` полезен для регистрации операций в многопоточных средах.
Давайте подтвердим это путем вычисления (и записи в журнал) некоторых факториалов.

Функция `factorial` ниже вычисляет факториал
и выводит промежуточные шаги по мере выполнения.
Вспомогательная функция `slow` гарантирует, что это займет некоторое время,
даже на очень маленьких примерах ниже:

```tut:book:silent
def slowly[A](body: => A) =
  try body finally Thread.sleep(100)

def factorial(n: Int): Int = {
  val ans = slowly(if(n == 0) 1 else n * factorial(n - 1))
  println(s"fact $n $ans")
  ans
}
```

Вот вывод — последовательность монотонно возрастающих значений:

```tut:book
factorial(5)
```

Если мы запустим вычисление нескольких факториалов параллельно,
то сообщения журнала могут чередоваться при стандартном выводе.
Это затрудняет просмотр того,
какие сообщения из каких вычислений приходят:

```tut:book:silent
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
```

```scala
Await.result(Future.sequence(Vector(
  Future(factorial(3)),
  Future(factorial(3))
)), 5.seconds)
// fact 0 1
// fact 0 1
// fact 1 1
// fact 1 1
// fact 2 2
// fact 2 2
// fact 3 6
// fact 3 6
// res14: scala.collection.immutable.Vector[Int] =
//   Vector(120, 120)
```

<!--
HACK: tut не захватывает стандартный вывод из вышеперечисленных тем,
так что я все сделал это, взломав его.
-->

Перепишите `factorial` так, чтобы он записывал
сообщения журнала в `Writer`.
Продемонстрируйте, что это позволяет нам
надежно разделять журналы
для параллельных вычислений.

<div class="solution">
Начнем с определения псевдонима типа для `Writer`,
чтобы мы могли использовать его с синтаксисом `pure`:

```tut:book:silent
import cats.data.Writer
import cats.syntax.applicative._ // для pure

type Logged[A] = Writer[Vector[String], A]
```

```tut:book
42.pure[Logged]
```

Мы также импортируем синтаксис `tell`:

```tut:book:silent
import cats.syntax.writer._ // для tell
```

```tut:book
Vector("Message").tell
```

Наконец, мы импортируем
экземпляр `Semigroup` для` Vector`.
Нам это нужно для вызова `map` и `flatMap` над `Logged`:

```tut:book:silent
import cats.instances.vector._ // для Monoid
```

```tut:book
41.pure[Logged].map(_ + 1)
```

С учетом этого, определением `factorial` становится:

```tut:book:silent
def factorial(n: Int): Logged[Int] =
  for {
    ans <- if(n == 0) {
             1.pure[Logged]
           } else {
             slowly(factorial(n - 1).map(_ * n))
           }
    _   <- Vector(s"fact $n $ans").tell
  } yield ans
```

Теперь, когда мы вызываем `factorial`,
мы должны у возвращаемого значения вызвать метод `run`,
чтобы извлечь журнал и наш факториал:

```tut:book
val (log, res) = factorial(5).run
```

Мы можем запустить несколько `factorial` параллельно следующим образом,
записывая их журналы независимо и
не опасаясь чередования записей:

```tut:book
val Vector((logA, ansA), (logB, ansB)) =
  Await.result(Future.sequence(Vector(
    Future(factorial(3).run),
    Future(factorial(5).run)
  )), 5.seconds)
```
</div>
