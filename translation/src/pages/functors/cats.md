## Функторы в Cats

Давайте рассмотрим реализацию функторов в Cats.
Мы изучим аспекты которые встречались и у моноидов:
*тайпкласс*, *экземпляры*, и *синтаксис*.

### Тайпкласс Функтор

Тайпкласс функтор [`cats.Functor`][cats.Functor].
Получить его экземпляры можно с помощью метода `Functor.apply`,
который находится в объекте-компаньоне.
Как обычно, готовые экземпляры упорядочены по типу в пакете
[`cats.instances`][cats.instances]:

```tut:book:silent
import scala.language.higherKinds
import cats.Functor
import cats.instances.list._   // for Functor
import cats.instances.option._ // for Functor
```

```tut:book
val list1 = List(1, 2, 3)
val list2 = Functor[List].map(list1)(_ * 2)

val option1 = Option(123)
val option2 = Functor[Option].map(option1)(_.toString)
```

`Functor` также предоставляет метод `lift`,
который преобразует функцию типа `A => B`
в функцию, которая имеет тип `F[A] => F[B]`:

```tut:book
val func = (x: Int) => x + 1

val liftedFunc = Functor[Option].lift(func)

liftedFunc(Option(1))
```

### Синтаксис Функтора

Главный метод предоставляемый синтаксисом для `Functor` это `map`.
Трудно продемонстрировать это с помощью `Options` и `Lists`,
так как у них есть свои собственные встроенные методы `map`
и компилятор Scala всегда будет предпочитать
встроенный метод в сравнении с методом расширения.
Мы рассмотрим это на двух примерах.

Сначала рассмотрим маппинг функций.
В типе `Function1` в Scala нет метода `map`,
(вместо этого он называется `andThen`)
поэтому и нет конфликтов с именами:

```tut:book:silent
import cats.instances.function._ // for Functor
import cats.syntax.functor._     // for map
```

```tut:book:silent
val func1 = (a: Int) => a + 1
val func2 = (a: Int) => a * 2
val func3 = (a: Int) => a + "!"
val func4 = func1.map(func2).map(func3)
```

```tut:book
func4(123)
```

Рассмотрим другой пример.
В этот раз мы будем абстрагироваться от функторов,
так чтобы не пришлось работать ни с одним конкретным типом.
Мы можем написать метод, который применяет уравнение к числу,
независимо от того, в каком контексте функтора оно находится:

```tut:book:silent
def doMath[F[_]](start: F[Int])
    (implicit functor: Functor[F]): F[Int] =
  start.map(n => n + 1 * 2)

import cats.instances.option._ // for Functor
import cats.instances.list._   // for Functor
```

```tut:book
doMath(Option(20))
doMath(List(1, 2, 3))
```

Чтобы проиллюстрировать, как это работает,
давайте посмотрим на определение метода
`map` в `cats.syntax.functor`.
Вот упрощённая версия кода:

```scala
implicit class FunctorOps[F[_], A](src: F[A]) {
  def map[B](func: A => B)
      (implicit functor: Functor[F]): F[B] =
    functor.map(src)(func)
}
```

Компилятор может использовать этот метод расширения
чтобы добавить метод `map` везде, где не доступен встроенный метод `map`:

```scala
foo.map(value => value + 1)
```

Если предположить что у `foo` нет встроенного метода `map`,
компилятор обнаруживает потенциальную ошибку и
обертывает выражение в `FunctorOps` для исправления кода:

```scala
new FunctorOps(foo).map(value => value + 1)
```

Метод `map` у `FunctorOps` требует
неявный `Functor` в качестве параметра.
Это означает, что данный код будет скомпилирован
если у нас есть `Functor` для `F` в области видимости.
Иначе, мы получаем ошибку компиляции:

```tut:book:silent
final case class Box[A](value: A)

val box = Box[Int](123)
```

```tut:book:fail
box.map(value => value + 1)
```

### Экземпляры пользовательских типов

Мы можем определить функтор просто определив его метод map.
Ниже представлен пример `Functor` для `Option`,
хотя подобный код уже существует в [`cats.instances`][cats.instances].
Реализация тривиальна — мы просто вызываем метод `map` у `Option`:

```tut:book:silent
implicit val optionFunctor: Functor[Option] =
  new Functor[Option] {
    def map[A, B](value: Option[A])(func: A => B): Option[B] =
      value.map(func)
  }
```

Иногда нам нужно внедрить зависимости для наших экземпляров.
Например, если бы нам пришлось определить свой `Functor` для `Future`,
(другой гипотетический пример Cats предоставляют в `cats.instances.future`)
нам бы пришлось запрашивать неявный параметр `ExecutionContext` для `future.map`.
Мы не можем добавлять дополнительные параметры к `functor.map`,
поэтому нам придётся запрашивать нужные нам зависимости в момент создания экземпляра:

```tut:book:silent
import scala.concurrent.{Future, ExecutionContext}

implicit def futureFunctor
    (implicit ec: ExecutionContext): Functor[Future] =
  new Functor[Future] {
    def map[A, B](value: Future[A])(func: A => B): Future[B] =
      value.map(func)
  }
```

Всякий раз, когда мы требуем `Functor` для `Future`,
либо напрямую используя `Functor.apply`
или косвенно с помощью метода расширения `map`,
компилятор обнаружит `futureFunctor` с помощью поиска неявных параметров
и рекурсивного поиска `ExecutionContext` по месту вызова.
Вот как может выглядеть такое расширение:

```scala
// Мы пишем:
Functor[Future]

// Сперва компилятор расширяет это до:
Functor[Future](futureFunctor)

// А потом до:
Functor[Future](futureFunctor(executionContext))
```

### Упражнение: Разветвляемся с Функторами

Напишите `Functor` для следующего бинарного дерева.
Проверьте что код ожидаемо работает для экземпляров `Branch` и `Leaf`:

```tut:book:silent
object wrapper {
  sealed trait Tree[+A]

  final case class Branch[A](left: Tree[A], right: Tree[A])
    extends Tree[A]

  final case class Leaf[A](value: A) extends Tree[A]
}; import wrapper._
```

<div class="solution">
Семантика похожа на написание `Functor` для `List`.
Рекурсивно обходим структуру данных, применяя функцию к каждому `Leaf`, который найдём.
Законы функтора интуитивно требуют от нас сохранения прежней структуры 
с таким же расположением узлов `Branch` и `Leaf`:

```tut:book:silent
import cats.Functor

implicit val treeFunctor: Functor[Tree] =
  new Functor[Tree] {
    def map[A, B](tree: Tree[A])(func: A => B): Tree[B] =
      tree match {
        case Branch(left, right) =>
          Branch(map(left)(func), map(right)(func))
        case Leaf(value) =>
          Leaf(func(value))
      }
  }
```

Давайте используем наш `Functor` для переобразования некоторых `Trees`:

```tut:book:fail
Branch(Leaf(10), Leaf(20)).map(_ * 2)
```

Упс! Перед нами встаёт таже проблема инвариантности, которую мы обсуждали в разделе [@sec:variance].
Компиллятор может найти экземпляр `Functor` для `Tree` но не для `Branch` или `Leaf`.
Давайте добавим смарт-конструкторы для решения этой проблемы:

```tut:book:silent
object Tree {
  def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
    Branch(left, right)

  def leaf[A](value: A): Tree[A] =
    Leaf(value)
}
```

Теперь мы можем использовать наш `Functor` правильно:

```tut:book
Tree.leaf(100).map(_ * 2)

Tree.branch(Tree.leaf(10), Tree.leaf(20)).map(_ * 2)
```
</div>
