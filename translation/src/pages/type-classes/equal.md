## Пример: Eq

В завершение этой главы изучим еще один полезный тайпкласс: 
[`cats.Eq`][cats.kernel.Eq].
`Eq` разработан для поддержки *типобезопасного равенства* 
и устранения неприятностей, которые доставляет встроенный в Scala оператор `==`.

Почти каждый Scala-разработчик в своё время писал код вроде такого:

```tut:book
List(1, 2, 3).map(Option(_)).filter(item => item == 1)
```

Допустим, многие из вас не допустили бы такой простой ошибки, как эта, 
но проблема остаётся актуальной.
Предикат в `filter` всегда возвращает `false`, 
потому что он сравнивает `Int` с `Option[Int]`.

Это является ошибкой программиста — 
мы должны были сравнить `item` с `Some(1)`, а не с `1`.
Технически это не должно считаться ошибкой типа, 
потому что `==` предназначен для сравнения пары любых объектов, независимо от их типа.
`Eq` разработан для того, чтобы на уровне типов 
добавить безопасность в проверки на равенство, 
и тем самым обойти эту проблему.

### Равенство, Свобода, Братство

Мы можем использовать `Eq`, чтобы определить типобезопасное равенство 
между любыми значениями некоторого типа:

```scala
package cats

trait Eq[A] {
  def eqv(a: A, b: A): Boolean
  // и другие методы, основанные на eqv ...
}
```

Интерфейсный синтаксис, определенный в [`cats.syntax.eq`][cats.syntax.eq], 
предоставляет два метода для проверки на равенство 
(при условии, что в области видимости есть соответствующий экземпляр `Eq[A]`):

 - `===` сравнивает два объекта на равенство;
 - `=!=` сравнивает два объекта на неравенство.

### Сравниваем целые числа

Давайте рассмотрим несколько примеров. Сначала импортируем тайпкласс:

```tut:book:silent
import cats.Eq
```

Теперь возьмем экземпляр для `Int`:

```tut:book:silent
import cats.instances.int._ // для Eq

val eqInt = Eq[Int]
```

Для проверки на равенство мы можем непосредственно обратиться к `eqInt`:

```tut:book
eqInt.eqv(123, 123)
eqInt.eqv(123, 234)
```

В отличие от встроенного в Scala метода `==`, 
попытка сравнения объектов разного типа при помощи `eqv` 
приведёт к ошибке компиляции:

```tut:book:fail
eqInt.eqv(123, "234")
```

Мы также можем импортировать интерфейсный синтаксис из [`cats.syntax.eq`][cats.syntax.eq], 
чтобы использовать методы `===` и `=!=`:

```tut:book:silent
import cats.syntax.eq._ // для === and =!=
```

```tut:book
123 === 123
123 =!= 234
```

Опять же, сравнение значений разных типов приведёт к ошибке компиляции:

```tut:book:fail
123 === "123"
```

### Сравниваем Option {#sec:type-classes:comparing-options}

Теперь рассмотрим более интересный пример — `Option[Int]`.
Для сравнения значений типа `Option[Int]` 
нам необходимо импортировать экземпляры `Eq` как для `Option`, так и для `Int`:

```tut:book:silent
import cats.instances.int._    // для Eq
import cats.instances.option._ // для Eq
```

Теперь можем попробовать сравнить:

```tut:fail:book
Some(1) === None
```

Мы получим ошибку, потому что типы не совсем совпадают.
В области видимости нам доступны экземпляры `Eq` для `Int` и для `Option[Int]`, 
но значения, которые мы сравниваем, имеют типы `Some[Int]` и `None`.
Чтобы решить эту проблему, придётся явно указать тип аргументов — `Option[Int]`:

```tut:book
(Some(1) : Option[Int]) === (None : Option[Int])
```

Мы можем сделать это и более лаконичным способом, 
используя методы `Option.apply` и` Option.empty` из стандартной библиотеки:

```tut:book
Option(1) === Option.empty[Int]
```

или используя специальный синтаксис из [`cats.syntax.option`][cats.syntax.option]:

```tut:book:silent
import cats.syntax.option._ // для some и none
```

```tut:book
1.some === none[Int]
1.some =!= none[Int]
```

### Сравнение пользовательских типов

Мы можем определить наши собственные экземпляры `Eq`, 
используя метод `Eq.instance`, 
который принимает функцию типа `(A, A) => Boolean` и возвращает `Eq[A]`:

```tut:book:silent
import java.util.Date
import cats.instances.long._ // для Eq
```

```tut:book:silent
implicit val dateEq: Eq[Date] =
  Eq.instance[Date] { (date1, date2) =>
    date1.getTime === date2.getTime
  }
```

```tut:book:silent
val x = new Date() // сейчас
val y = new Date() // мгновение спустя
```

```tut:book
x === x
x === y
```

### Упражнение: Равенство, Свобода, Котики

Реализуйте экземпляр `Eq` для нашего типа `Cat`:

```tut:book:silent
final case class Cat(name: String, age: Int, color: String)
```

Используйте его, чтобы сравнить данные пары объектов на равенство и неравенство:

```tut:book:silent
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]
```

<div class="solution">
Сначала нам нужно импортировать Cats.
В этом упражнении мы будем использовать тайпкласс `Eq` 
и интерфейсный синтаксис `Eq`:

```tut:book:silent
import cats.Eq
import cats.syntax.eq._ // для ===
```

Наш класс `Cat` тот же, что и прежде:

```scala
final case class Cat(name: String, age: Int, color: String)
```

Теперь добавим в область видимости экземпляры `Eq` для `Int` и `String`, 
чтобы на их основе реализовать `Eq[Cat]`:

```tut:book:silent
import cats.instances.int._    // для Eq
import cats.instances.string._ // для Eq

implicit val catEqual: Eq[Cat] =
  Eq.instance[Cat] { (cat1, cat2) =>
    (cat1.name  === cat2.name ) &&
    (cat1.age   === cat2.age  ) &&
    (cat1.color === cat2.color)
  }
```

Наконец, проверим всё это в небольшом приложении:

```tut:book
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 32, "orange and black")

cat1 === cat2
cat1 =!= cat2
```

```tut:book:silent
import cats.instances.option._ // для Eq
```

```tut:book
val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]

optionCat1 === optionCat2
optionCat1 =!= optionCat2
```
</div>
