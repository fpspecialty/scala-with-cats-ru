## Упражнение: библиотека Printable

Scala предоставляет метод `toString`, 
который позволяет нам преобразовывать любое значение в `String`.
Однако у этого метода есть недостатки: 
он реализован для *каждого* типа в языке, 
и многие его реализации имеют очень ограниченное применение, 
а мы не можем просто так заменить его стандартную реализацию в нужных нам типах
на какую-либо другую.

Давайте определим тайпкласс `Printable`, чтобы обойти эти проблемы:

 1. Определите класс типов `Printable[A]`, содержащий единственный метод — `format`.
    `format` должен принимать значение типа `A` и возвращать `String`.

 2. Создайте объект `PrintableInstances`, 
    содержащий экземпляры `Printable` для `String` и `Int`.

 3. Определите объект `Printable` с двумя обобщёнными интерфейсными методами:

    `format` — принимает значение типа `A` 
    и экземпляр `Printable` для соответствующего типа.
    Он использует `Printable` для преобразования `A` в `String`.

    `print` — принимает те же параметры, что и `format`, но возвращает `Unit`.
    Он выводит значение `A` в консоль, используя `println`.

<div class="solution">
Следующие шаги определяют три основных компонента нашего тайпкласса:
Сначала определяем `Printable` — сам *тайпкласс*:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String
}
```

Затем добавляем некоторые предопределённые *экземпляры* `Printable` 
и складываем их в `PrintableInstances`:

```tut:book:silent
object PrintableInstances {
  implicit val stringPrintable = new Printable[String] {
    def format(input: String) = input
  }

  implicit val intPrintable = new Printable[Int] {
    def format(input: Int) = input.toString
  }
}
```

И, наконец, определяем *интерфейсный объект* `Printable`:

```tut:book:silent
object Printable {
  def format[A](input: A)(implicit p: Printable[A]): String =
    p.format(input)

  def print[A](input: A)(implicit p: Printable[A]): Unit =
    println(format(input))
}
```
</div>

**Использование библиотеки**

В результате выполнения предыдущей задачи мы получили универсальную библиотеку 
для вывода на консоль, которую мы можем использовать в различных приложениях.
Теперь давайте реализуем приложение, которое использует эту библиотеку.

Для начала определим тип данных, представляющий всем известное пушистое животное:

```scala
final case class Cat(name: String, age: Int, color: String)
```

Затем создадим реализацию `Printable` для `Cat`, 
которая возвращает строку следующего формата:

```ruby
NAME is a AGE year-old COLOR cat.
```

Наконец, воспользуемся нашим тайпклассом в консоли или в небольшом демо-приложении: 
создадим `Cat` и выведем его на консоль:

```scala
// Определение кота:
val cat = Cat(/* ... */)

// Печать кота!
```

<div class="solution">
Это совершенно стандартное применение паттерна «тайпкласс».
Сначала определяем пользовательские типы данных для нашего приложения:

```tut:book:silent
final case class Cat(name: String, age: Int, color: String)
```

Затем определяем экземпляры тайпклассов для нужных нам типов.
Мы разместим их либо в объекте-компаньоне `Cat`, 
либо в отдельном объекте, выступающем в качестве пространства имен:

```tut:book:silent
import PrintableInstances._

implicit val catPrintable = new Printable[Cat] {
  def format(cat: Cat) = {
    val name  = Printable.format(cat.name)
    val age   = Printable.format(cat.age)
    val color = Printable.format(cat.color)
    s"$name is a $age year-old $color cat."
  }
}
```

Наконец, воспользуемся тайпклассом, 
вводя соответствующие экземпляры в область видимости 
и используя интерфейсный объект или интерфейсный синтаксис.
Если мы определили экземпляры в объекте-компаньоне, 
Scala автоматически добавит их в область видимости.
В противном случае используем `import`, чтобы ими воспользоваться:

```tut:book
val cat = Cat("Garfield", 38, "ginger and black")

Printable.print(cat)
```
</div>

**Более подходящий синтаксис**

Давайте сделаем нашу библиотеку ещё более простой в использовании, 
определив некоторые методы расширения для более удобного синтаксиса:

 1. Создайте объект с именем `PrintableSyntax`.

 2. Внутри `PrintableSyntax` определите `implicit class PrintableOps[A]`, 
    который будет оборачивать значение типа `A`.

 3. В `PrintableOps` определите следующие методы:

     - `format` — принимает неявный параметр `Printable[A]` 
       и возвращает строковое представление обернутого `A`;

     - `print` — принимает неявный параметр `Printable[A]` и возвращает `Unit`.
       Он выводит `A` на консоль.

 4. Используйте методы расширения, чтобы вывести на консоль значение типа `Cat`, 
    которое вы создали в предыдущем упражнении.

<div class="solution">
Сначала мы определяем `implicit`-класс, содержащий нужные нам методы расширения:

```tut:book:silent
object PrintableSyntax {
  implicit class PrintableOps[A](value: A) {
    def format(implicit p: Printable[A]): String =
      p.format(value)

    def print(implicit p: Printable[A]): Unit =
      println(format(p))
  }
}
```

И теперь, при условии, что `PrintableOps` будет доступен в области видимости, 
мы сможем вызывать выдуманные нами методы `print` и `format` для любого значения, 
для которого Scala может найти экземпляр `Printable`:

```tut:book:silent
import PrintableSyntax._
```

```tut:book
Cat("Garfield", 38, "ginger and black").print
```

А если для некоторого типа экземпляр `Printable` не будет определён, 
то мы получим ошибку компиляции:

```tut:book:silent
import java.util.Date
```

```tut:book:fail
new Date().print
```
</div>
