## Отступление: Частичная унификация {#sec:functors:partial-unification}

В разделе [@sec:functors:more-examples]
мы увидели любопытную ошибку компилятора.
Компиляция приведенного ниже кода пройдёт успешно
при условии, что был включен флаг компилятора `-Ypartial-unification`:

```tut:book:silent
import cats.Functor
import cats.instances.function._ // для Functor
import cats.syntax.functor._     // для map

val func1 = (x: Int)    => x.toDouble
val func2 = (y: Double) => y * 2
```

```tut:book
val func3 = func1.map(func2)
```

но завершится с ошибкой, если флаг был пропущен:

```scala
val func3 = func1.map(func2)
// <console>: error: value map is not a member of Int => Double
//        val func3 = func1.map(func2)
                            ^
```

Очевидно, что «частичная унификация» — 
это некое необязательное поведение компилятора, 
без которого наш код не будет компилироваться.
Мы должны уделить время, чтобы описать это поведение,
обсудить некоторые подводные камни и обходные пути.

### Унифицирующие конструкторы типов

Чтобы скомпилировать выражение, такое как `func1.map (func2)` выше, 
компилятор должен искать `Functor` для` Function1`.
Однако, `Functor` принимает конструктор типа с одним параметром:

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(func: A => B): F[B]
}
```

и `Function1` имеет два параметра типа
(аргумент функции и тип результата):

```scala
trait Function1[-A, +B] {
  def apply(arg: A): B
}
```

Компилятор должен исправить один из двух параметров
`Function1` для создания конструктора типа
правильного рода, чтобы передать в `Functor`.
У него есть два варианта на выбор:

```tut:book:silent
type F[A] = Int => A
type F[A] = A => Double
```

*Мы* знаем, что первый вариант — правильный выбор.
Однако более ранние версии компилятора Scala 
не могли сделать такой вывод.
Это печально известное ограничение, 
известное как [SI-2712] [link-si2712], 
не позволяло компилятору «унифицировать» конструкторы типов 
разных арностей.
Это ограничение компилятора теперь исправлено, 
но мы должны включить его 
с помощью флага компилятора в `build.sbt`:

```scala
scalacOptions += "-Ypartial-unification"
```

### Исключение параметров слева направо

Частичная унификация в компиляторе Scala 
работает через исправление параметров типа слева направо.
В приведенном выше примере компилятор исправляет 
`Int` в `Int => Double` 
и ищет `Functor` для функций типа `Int =>?`:

```tut:book:silent
type F[A] = Int => A

val functor = Functor[F]
```

Такое исключение параметров слева направо работает для 
широкого спектра распространенных сценариев, 
включая поиск `Functor` для 
таких типов, как` Function1` и `Either`:

```tut:book:silent
import cats.instances.either._ // для Functor
```

```tut:book
val either: Either[String, Int] = Right(123)

either.map(_ + 1)
```

Однако существуют ситуации, 
где исключение слева направо не является правильным выбором.
Одним из таких примеров является тип `Or` в [Scalactic][link-scalactic], 
который условно является левосторонним эквивалентом `Either`:

```scala
type PossibleResult = ActualResult Or Error
```

Другим примером является функтор `Contravariant` для `Function1`.

В то время, как ковариантный `Functor` для` Function1` реализует 
композицию функций в стиле `andThen` слева направо, 
функтор `Contravariant` реализует композицию справа налево 
в стиле `compose`.
Другими словами, все следующие выражения эквивалентны:

```tut:book:silent
val func3a: Int => Double =
  a => func2(func1(a))

val func3b: Int => Double =
  func2.compose(func1)
```

```tut:book:fail:silent
// Hypothetical example. This won't actually compile:
val func3c: Int => Double =
  func2.contramap(func1)
```

Однако, если мы попробуем это реализовать, 
наш код не скомпилируется:

```tut:book:silent
import cats.syntax.contravariant._ // для contramap
```

```tut:book:fail
val func3c = func2.contramap(func1)
```

Проблема здесь в том, что `Contravariant` для `Function1` 
исправляет тип возвращаемого значения и оставляет тип параметра изменяющимся, 
требуя, чтобы компилятор исключал параметры типа 
справа налево, как показано ниже и на рисунке [@fig:functors:function-contramap-type-chart]:

```scala
type F[A] = A => Double
```

![Диаграмма: применение contramap к Function1](src/pages/functors/function-contramap.pdf+svg){#fig:functors:function-contramap-type-chart}

Ошибки компилятора происходят из-за его ориентированности слева направо. 
Мы можем доказать это, создав псевдоним типа, 
который меняет местами параметры в Function1:

```tut:book:silent
type <=[B, A] = A => B

type F[A] = Double <= A
```

Если мы переопределим `func2` в качестве экземпляра` <= `, 
мы сбросим требуемый порядок исключения и 
можем вызвать `contramap` как хотели:

```tut:book:silent
val func2b: Double <= Double = func2
```

```tut:book
val func3c = func2b.contramap(func1)
```

Разница между `func2` и `func2b` является 
чисто синтаксической — оба ссылаются на одно и то же значение, 
а псевдонимы типов полностью совместимы.
Невероятно, однако, что 
этого простого изменения достаточно, чтобы 
дать компилятору подсказку, необходимую 
для решения проблемы.

Нам редко приходится реализовывать
исключение справа налево. 
Большинство многопараметрических конструкторов типов 
право-ориентированы, 
что требует исключения слева направо, 
которое поддерживается компилятором из коробки. 
Тем не менее, полезно знать о `-Ypartial-unification`
и этой причуде порядка исключения 
в случае, если вы когда-нибудь столкнетесь 
со странным сценарием, подобным приведенному выше.
