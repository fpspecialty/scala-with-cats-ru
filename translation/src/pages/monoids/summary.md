## Итог

В этой главе мы сделали большой прогресс — мы рассмотрели
наши первые классы с причудливыми именами из функционального программирования:

- `Semigroup`, представляющая собой операцию сложения или комбинации;
- `Monoid`, расширяющий полугруппу, добавляя тождественный (нейтральный) элемент.

Мы можем использовать `Semigroup` и `Monoid`, импортируя три вещи:
сами классы типов, экземпляры для нужных нам типов,
и синтаксис полугруппы, чтобы использовать оператор `|+|`:


```tut:book:silent
import cats.Monoid
import cats.instances.string._ // для Monoid
import cats.syntax.semigroup._ // для |+|
```

```tut:book
"Scala" |+| " with " |+| "Cats"
```

С правильными экземплярами в области видимости
мы можем складывать что угодно:

```tut:book:silent
import cats.instances.int._    // для Monoid
import cats.instances.option._ // для Monoid
```

```tut:book
Option(1) |+| Option(2)
```

```tut:book:silent
import cats.instances.map._ // для Monoid

val map1 = Map("a" -> 1, "b" -> 2)
val map2 = Map("b" -> 3, "d" -> 4)
```

```tut:book
map1 |+| map2
```

```tut:book:silent
import cats.instances.tuple._  // для Monoid


val tuple1 = ("hello", 123)
val tuple2 = ("world", 321)
```

```tut:book
tuple1 |+| tuple2
```

Мы также можем написать обобщённый код, который работает с любым типом,
для которого у нас есть экземпляр `Monoid`:

```tut:book:silent
def addAll[A](values: List[A])
      (implicit monoid: Monoid[A]): A =
  values.foldRight(monoid.empty)(_ |+| _)
```

```tut:book
addAll(List(1, 2, 3))
addAll(List(None, Some(1), Some(2)))
```

Моноиды — великолепная тема для знакомства с Cats.
Их легко понять и просто использовать.
Тем не менее, это только верхушка айсберга
с точки зрения абстракций, которые Cats позволяет нам использовать.
В следующей главе мы рассмотрим функторы —
излюбленный метод `map` в обличии тайпкласса.
Вот тут-то и начинается самое интересное!
