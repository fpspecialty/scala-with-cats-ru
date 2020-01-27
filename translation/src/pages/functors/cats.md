## Функторы в Cats

Давайте рассмотрим реализацию функторов в Cats.
Мы изучим аспекты, которые встречались и у моноидов:
*тайпкласс*, *экземпляры*, и *синтаксис*.

### Тайпкласс Функтор

Тайпкласс функтор — это [`cats.Functor`][cats.Functor].
Получить его экземпляры можно с помощью метода `Functor.apply`,
который находится в объекте-компаньоне.
Как обычно, готовые экземпляры сгруппированы по типу в пакете
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

### Синтаксис функторов

Главный метод предоставляемый синтаксисом для `Functor` — это `map`.
Это тяжело продемонстрировать на примере `Option` или `List`,
так как у них есть свои собственные встроенные методы `map`,
и компилятор Scala всегда предпочтёт
использовать встроенный метод, нежели метод расширения.
Поэтому мы рассмотрим в качестве примера пару других типов.

Сначала рассмотрим отображение между функциями.
В Scala для типа `Function1` не определён метод `map`,
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
На этот раз мы абстрагируем функторы,
не привязываясь таким образом ни к одному конкретному типу.
Мы можем написать метод, который применяет выражение к числу,
независимо от того, в контексте какого функтора оно находится:

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

Компилятор может использовать этот метод расширения,
чтобы добавить метод `map` всюду, где отсутствует встроенная реализация `map`:

```scala
foo.map(value => value + 1)
```

Если предположить, что у `foo` отсутствует собственная реализация `map`,
компилятор обнаруживает потенциальную ошибку и
оборачивает выражение в `FunctorOps` для исправления кода:

```scala
new FunctorOps(foo).map(value => value + 1)
```

Метод `map` у `FunctorOps` требует
`Functor` в качестве неявного параметра.
Это означает, что данный код будет скомпилирован
только если в области видимости есть экземпляр `Functor` для `F`.
Иначе мы получаем ошибку компиляции:

```tut:book:silent
final case class Box[A](value: A)

val box = Box[Int](123)
```

```tut:book:fail
box.map(value => value + 1)
```

### Экземпляры пользовательских типов

Мы можем определить функтор, просто определив его метод map.
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

Иногда нам нужно внедрить зависимости для экземпляров.
Например, если бы нам пришлось определить свой `Functor` для `Future`
(другой гипотетический пример Cats предоставляют в `cats.instances.future`),
нам бы пришлось запрашивать неявный параметр `ExecutionContext` для `future.map`.
Мы не можем добавлять дополнительные параметры к `functor.map`,
поэтому придётся запрашивать нужные зависимости там, где мы объявляем экземпляр:

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
либо напрямую используя `Functor.apply`,
либо косвенно, с помощью метода расширения `map`,
компилятор обнаружит `futureFunctor` путём поиска значений неявных параметров
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

### Упражнение: Разветвляемся с функторами

Напишите `Functor` для следующего бинарного дерева.
Проверьте, что код ожидаемо работает для экземпляров `Branch` и `Leaf`:

```tut:book:silent
object wrapper {
  sealed trait Tree[+A]

  final case class Branch[A](left: Tree[A], right: Tree[A])
    extends Tree[A]

  final case class Leaf[A](value: A) extends Tree[A]
}; import wrapper._
```

<div class="solution">
Семантика `Functor` для дерева схожа с таковой для `List`.
Рекурсивно обходим структуру данных, применяя функцию к каждому `Leaf`, который найдём.
Законы функтора интуитивно подталкивают нас к сохранению прежней структуры 
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

Давайте опробуем наш `Functor` на примере некоторых `Tree`:

```tut:book:fail
Branch(Leaf(10), Leaf(20)).map(_ * 2)
```

Упс! Перед нами встаёт всё та же проблема инвариантности, что и в разделе [@sec:variance].
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
