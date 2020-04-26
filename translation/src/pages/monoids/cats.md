## Моноиды в Cats

Сейчас мы увидели что такое моноиды,
давайте посмотрим на их реализацию в Cats.
Мы уже рассмотрели три главных аспекта реализации:
*тайпкласс*, *экземпляры*, и *интерфейс*.

### Тайпкласс моноида

Тайпкласс моноида представлен `cats.kernel.Monoid`,
и доступен по псевдониму [`cats.Monoid`][cats.kernel.Monoid].
`Monoid` расширяет `cats.kernel.Semigroup`,
которая доступна по псевдониму [`cats.Semigroup`][cats.kernel.Semigroup].
При использовании Cats, мы обычно импортируем тайпкласс
из модуля [`cats`][cats.package]:

```tut:book:silent
import cats.Monoid
import cats.Semigroup
```

<div class="callout callout-info">
*Cats Kernel?*

Cats Kernel это подпроект Cats,
предоставляющий небольшой набор тайпклассов
для библиотек, которые не требуют полного набора инструментов Cats.
Пока это ядро тайпклассов технически
определено в модуле [`cats.kernel`][cats.kernel.package],
они все доступны по псевдониму модуля [`cats`][cats.package],
поэтому нам не нужно беспокоиться о различии.

Тайпклассы Cats Kernel, расммотренные в этой книге, это
[`Eq`][cats.kernel.Eq],
[`Semigroup`][cats.kernel.Semigroup],
и [`Monoid`][cats.kernel.Monoid].
Все остальные тайпклассы, которые мы рассмотрели,
это часть основного проекта Cats, и они
определены в модуле [`cats`][cats.package].
</div>

### Экземпляры моноидов {#sec:monoid-instances}

`Monoid` следует стандартному паттерну Cats пользовательского интерфейса:
объект-компаньон имеет метод `apply`,
который возвращает экземпляр тайпкласса для конкретного типа.
Например, если мы хотим экземпляр моноида для `String`,
и мы имеем подходящие значения неявных параметров в области видимости,
мы можем написать следующее:

```tut:book:silent
import cats.Monoid
import cats.instances.string._ // для Monoid
```

```tut:book
Monoid[String].combine("Hi ", "there")
Monoid[String].empty
```

что эквивалентно следующему:

```tut:book
Monoid.apply[String].combine("Hi ", "there")
Monoid.apply[String].empty
```

Как мы знаем, `Monoid` расширяет `Semigroup`.
Если нам не нужен метод `empty`, мы можем также написать:

```tut:book:silent
import cats.Semigroup
```

```tut:book
Semigroup[String].combine("Hi ", "there")
```

Экземпляр тайпкласса для `Monoid`
организован под `cats.instances`
стандартным образом, описанным
в [Главе 1](#importing-default-instances).
Например, если мы хотим получить экземпляры для `Int`
мы импортируем их из [`cats.instances.int`][cats.instances.int]:

```tut:book:silent
import cats.Monoid
import cats.instances.int._ // для Monoid
```

```tut:book
Monoid[Int].combine(32, 10)
```

Похожим образом, мы можем собрать `Monoid[Option[Int]]`, используя
экземпляры из [`cats.instances.int`][cats.instances.int]
и [`cats.instances.option`][cats.instances.option]:

```tut:book:silent
import cats.Monoid
import cats.instances.int._    // для Monoid
import cats.instances.option._ // для Monoid
```

```tut:book
val a = Option(22)
val b = Option(20)

Monoid[Option[Int]].combine(a, b)
```

Вернитесь к [Главе 1](#importing-default-instances)
для более полного списка импортов.

### Синтаксис моноидов {#sec:monoid-syntax}

Cats предоставляет синтаксис для метода `combine`
в виде оператора `|+|`.
Так как `combine` технически получен от `Semigroup`,
мы получаем этот синтаксис через импорт из [`cats.syntax.semigroup`][cats.syntax.semigroup]:

```tut:book:silent
import cats.instances.string._ // для Monoid
import cats.syntax.semigroup._ // для |+|
```

```tut:book
val stringResult = "Hi " |+| "there" |+| Monoid[String].empty
```

```tut:book:silent
import cats.instances.int._ // для Monoid
```

```tut:book
val intResult = 1 |+| 2 |+| Monoid[Int].empty
```

### Упражнение: добавляем всё

Передовой *SuperAdder v3.5a-32* лучший выбор в мире для сложения чисел.
Главная функция программы имеет сигнатуру `def add(items: List[Int]): Int`.
По трагической случайности этот код удалён! Перепиши метод и спаси день!

<div class="solution">
Мы можем записать сложение просто как `foldLeft`, используя `0` и оператор `+`:

```tut:book:silent
def add(items: List[Int]): Int =
  items.foldLeft(0)(_ + _)
```

Также мы можем записать это с использованием `Monoid`,
хотя здесь ещё нет в этом необходимости:

```tut:book:silent
import cats.Monoid
import cats.instances.int._    // для Monoid
import cats.syntax.semigroup._ // для |+|

def add(items: List[Int]): Int =
  items.foldLeft(Monoid[Int].empty)(_ |+| _)
```
</div>

Хорошая работа! Доля SuperAdder на рынке продолжает расти,
и появился спрос на дополнительную функциональность.
Сейчас люди хотят складывать `List[Option[Int]]`.
Измените `add` так, чтобы это было возможным.
Кодовая база SupperAdder высочайшего качества,
так что убедитесь, что в решении не будет дублирования кода!

<div class="solution">
Теперь появилось применение для `Monoid`.
Нам нужен один метод, который складывает `Int` и экземпляры `Option[Int]`.
Мы можем записать это как шаблонный метод, который получает `Monoid` как неявный параметр:

```tut:book:silent
import cats.Monoid
import cats.instances.int._    // для Monoid
import cats.syntax.semigroup._ // для |+|

def add[A](items: List[A])(implicit monoid: Monoid[A]): A =
  items.foldLeft(monoid.empty)(_ |+| _)
```

По желанию, мы можем использовать *context bound* синтаксис Scala, чтобы записать то же самое понятнее: 

```tut:book:silent
def add[A: Monoid](items: List[A]): A =
  items.foldLeft(Monoid[A].empty)(_ |+| _)
```

Мы можем использовать этот код для добавления значений типа `Int` и `Option[Int]`по запросу:

```tut:book:silent
import cats.instances.int._ // для Monoid
```

```tut:book
add(List(1, 2, 3))
```

```tut:book:silent
import cats.instances.option._ // для Monoid
```

```tut:book
add(List(Some(1), None, Some(2), None, Some(3)))
```

Обратите внимание, что если мы попытаемся добавить список, состоящий исключительно из значений `Some`,
мы получим ошибку компиляции:

```tut:book:fail
add(List(Some(1), Some(2), Some(3)))
```

Это происходит потому, что предполагаемый тип списка - `List[Some[Int]]`,
в то время как Cats будет генерировать `Monoid` только для `Option[Int]`.
Мы скоро увидим, как обойти это.
</div>

SuperAdder выходит на рынок POS (точек продаж, а не других POS).
Теперь мы хотим добавить `Order`:

```tut:book:silent
case class Order(totalCost: Double, quantity: Double)
```

Нам нужно очень скоро выпустить этот код в релиз, поэтому мы не можем внести никаких изменений в `add`.
Сделай это так!

<div class="solution">
Легко — мы просто определяем экземпляр моноида для `Order`!

```tut:book:silent
implicit val monoid: Monoid[Order] = new Monoid[Order] {
  def combine(o1: Order, o2: Order) =
    Order(
      o1.totalCost + o2.totalCost,
      o1.quantity + o2.quantity
    )

  def empty = Order(0, 0)
}
```
</div>
