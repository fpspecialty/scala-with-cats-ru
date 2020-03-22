## Монады в Cats

А теперь перейдём к реализации монад в Cats.
Как обычно, мы рассмотрим тайпкласс, экземпляры, и синтаксис.

### Тайпкласс «Монада» {#monad-type-class}

Тайпкласс «Монада» - это [`cats.Monad`][cats.Monad].
`Monad` расширяет два других тайпкласса:
`FlatMap`, который предоставляет метод `flatMap`,
и `Applicative`, предоставляющий метод `pure`.
`Applicative` также расширяет тайпкласс `Functor`,
который даёт всем монадам метод `map`, 
как мы увидели в упражнении выше.
Мы рассмотрим `Applicative` в разделе [@sec:applicatives].

Вот несколько примеров использования `pure`, `flatMap` и `map`:

```tut:book:silent
import cats.Monad
import cats.instances.option._ // для Monad
import cats.instances.list._   // для Monad
```

```tut:book
val opt1 = Monad[Option].pure(3)
val opt2 = Monad[Option].flatMap(opt1)(a => Some(a + 2))
val opt3 = Monad[Option].map(opt2)(a => 100 * a)

val list1 = Monad[List].pure(3)
val list2 = Monad[List].
  flatMap(List(1, 2, 3))(a => List(a, a*10))
val list3 = Monad[List].map(list2)(a => a + 123)
```

`Monad` предоставляет много других методов,
включая все методы тайпкласса `Functor`.
Подробную информацию смотрите в [scaladoc][cats.Monad].

### Экземпляры по умолчанию

Cats предоставляет экземпляры для всех монад в стандартной библиотеке
(`Option`, `List`, `Vector` и т.д.) в составе пакета [`cats.instances`][cats.instances]:

```tut:book:silent
import cats.instances.option._ // для Monad
```

```tut:book
Monad[Option].flatMap(Option(1))(a => Option(a*2))
```

```tut:book:silent
import cats.instances.list._ // для Monad
```

```tut:book
Monad[List].flatMap(List(1, 2, 3))(a => List(a, a*10))
```

```tut:book:silent
import cats.instances.vector._ // для Monad
```

```tut:book
Monad[Vector].flatMap(Vector(1, 2, 3))(a => Vector(a, a*10))
```

Cats также предоставляет экземпляр `Monad` для `Future`.
В отличие от методов класса `Future`,
у монад методы `pure` и `flatMap`
не могут принимать неявный параметр `ExecutionContext`
(потому что он не входит в определение трейта `Monad`).
Чтобы использовать `Future` как монаду несмотря на это ограничение, Cats требует, чтобы `ExecutionContext` был доступен в области видимости:

```tut:book:silent
import cats.instances.future._ // для Monad
import scala.concurrent._
import scala.concurrent.duration._
```

```tut:book:fail
val fm = Monad[Future]
```

Добавление `ExecutionContext` в область видимости
удовлетворяет все потребности в неявных параметрах, позволяя использовать экземпляр:

```tut:book:silent
import scala.concurrent.ExecutionContext.Implicits.global
```

```tut:book
val fm = Monad[Future]
```

Экземпляр `Monad` использует предоставленный `ExecutionContext`
для последующих вызовов методов `pure` и `flatMap`:

```tut:book:silent
val future = fm.flatMap(fm.pure(1))(x => fm.pure(x + 2))
```

```tut:book
Await.result(future, 1.second)
```

В дополнение к сказанному выше,
Cats предоставляет основу для новых монад, которых нет в стандартной библиотеке.
Мы познакомимся с некоторыми из них в дальнейшем.

### Синтаксис монад

Синтаксис для монад находится в трёх местах:

 - [`cats.syntax.flatMap`][cats.syntax.flatMap]
   предоставляет синтаксис для `flatMap`;
 - [`cats.syntax.functor`][cats.syntax.functor]
   предоставляет синтаксис для `map`;
 - [`cats.syntax.applicative`][cats.syntax.applicative]
   предоставляет синтаксис для `pure`.

На практике легче всего бывает импортировать всё за один раз из
пакета [`cats.implicits`][cats.implicits].
Однако, мы будем использовать индивидуальные импорты для ясности.

Мы можем использовать метод `pure` для создания экземпляров монады.
Нам будет часто требоваться предоставлять типовый параметр для однозначного указания нужного экземпляра.

```tut:book:silent
import cats.instances.option._   // для Monad
import cats.instances.list._     // для Monad
import cats.syntax.applicative._ // для pure
```

```tut:book
1.pure[Option]
1.pure[List]
```

Было бы сложно продемонстрировать монадическое поведение как нечто особенное, используя в качестве примеров `Option` или `List`, поскольку в Scala для них уже реализованы методы `flatMap` и `map`.
Вместо этого мы напишем обобщённую функцию, которая производит вычисления над значениями, передаваемыми в произвольной монаде, по выбору пользователя:

```tut:book:silent
import cats.Monad
import cats.syntax.functor._ // для map
import cats.syntax.flatMap._ // для flatMap
import scala.language.higherKinds

def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  a.flatMap(x => b.map(y => x*x + y*y))

import cats.instances.option._ // для Monad
import cats.instances.list._   // для Monad
```

```tut:book
sumSquare(Option(3), Option(4))
sumSquare(List(1, 2, 3), List(4, 5))
```

Мы можем переписать этот код, используя `for`-выражения.
Компилятор преобразует наше выражение, используя методы `flatMap` и `map`,
и вставит корректные неявные преобразования для использования `Monad`:

```tut:book:silent
def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  for {
    x <- a
    y <- b
  } yield x*x + y*y
```

```tut:book
sumSquare(Option(3), Option(4))
sumSquare(List(1, 2, 3), List(4, 5))
```

Это всё, что нам нужно знать
о большинстве монад в Cats.
Теперь давайте посмотрим на некоторые полезные экземпляры монад,
которые отсутствуют в стандартной библиотеке Scala.
