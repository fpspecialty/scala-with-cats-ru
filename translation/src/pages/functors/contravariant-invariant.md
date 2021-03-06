## *Контравариантные* и инвариантные функторы {#contravariant-invariant}

Как мы уже видели, мы можем думать о методе `map` в `Functor` 
как о «добавлении» преобразования в некоторую последовательность.
Рассмотрим два других тайпкласса,
один из которых представляет собой применение *предварительных* операций к последовательности,
другой — создание *двунаправленной*
последовательности операций. Они называются *контравариантным*
и *инвариантным функторами* соответственно.

<div class="callout callout-info">
*Раздел является опциональным!*

Вам не требуется знать о контравариантных и инвариантных функторах для понимания монад,
которые являются наиболее важным паттерном в этой книге и основной темой следующей главы.
Тем не менее, контравариантные и инвариантные функторы пригодятся 
нам в обсуждении `Semigroupal` и `Applicative` в главе [@sec:applicatives].

Если хотите перейти к монадам прямо сейчас —
не стесняйтесь, переходите сразу к главе [@sec:monads].
Но вернитесь сюда, прежде чем приступить к главе [@sec:applicatives].
</div>

### Контравариантные функторы и метод *contramap* {#contravariant}

Первый тайпкласс, *контравариантный функтор*,
предоставляет операцию `contramap`,
которая представляет собой применение «предварительных» операций к последовательности.
Общее представление операции изображено на рисунке [@fig:functors:contramap-type-chart].

![Типовая диаграмма: метод contramap](src/pages/functors/generic-contramap.pdf+svg){#fig:functors:contramap-type-chart}

Метод `contramap` имеет смысл только для тех типов, которые представляют собой *преобразования*.
Например, мы не можем объявить `contramap` для `Option`,
потому что нет способа предварительно обработать значение в
`Option[B]` с помощью функции `A => B`.
Однако, мы можем объявить `contramap` для тайпкласса `Printable`,
который мы обсуждали в главе [@sec:type-classes]:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String
}
```

`Printable[A]` представляет преобразование из `A` в `String`.
Его метод `contramap` принимает функцию `func` типа `B => A`
и создаёт новый `Printable[B]`:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String

  def contramap[B](func: B => A): Printable[B] =
    ???
}

def format[A](value: A)(implicit p: Printable[A]): String =
  p.format(value)
```

#### Упражнение: Хвастаемся с contramap

Реализуйте метод `contramap` для `Printable`, упомянутого выше.
Начните с предложенной заготовки
и замените `???` на рабочий код метода:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String

  def contramap[B](func: B => A): Printable[B] =
    new Printable[B] {
      def format(value: B): String =
        ???
    }
}
```

Если вы застряли, подумайте о типах.
Вам нужно преобразовать `value` типа `B`, в `String`.
Какие функции и методы вам доступны
и в каком порядке их нужно скомбинировать?

<div class="solution">
Ниже представлена рабочая реализация.
Мы вызываем `func` для преобразования `B` в `A`,
и затем используем `Printable`
для преобразования `A` в `String`.
Лёгким движением руки
мы используем алиас `self`, чтобы различать
внешний и внутренний `Printable`:

```tut:book:silent
trait Printable[A] {
  self =>

  def format(value: A): String

  def contramap[B](func: B => A): Printable[B] =
    new Printable[B] {
      def format(value: B): String =
        self.format(func(value))
    }
}

def format[A](value: A)(implicit p: Printable[A]): String =
  p.format(value)
```
</div>

Для тестирования
давайте определим некоторые экземпляры `Printable`
для `String` и `Boolean`:

```tut:book:silent
implicit val stringPrintable: Printable[String] =
  new Printable[String] {
    def format(value: String): String =
      "\"" + value + "\""
  }

implicit val booleanPrintable: Printable[Boolean] =
  new Printable[Boolean] {
    def format(value: Boolean): String =
      if(value) "yes" else "no"
  }
```

```tut:book
format("hello")
format(true)
```

Теперь определите экземпляр `Printable` для
заданного case-класса `Box`.
Вам потребуется объявить его как `implicit def`,
как описано в разделе [@sec:type-classes:recursive-implicits]:

```tut:book:silent
final case class Box[A](value: A)
```

Вместо того, чтобы писать
полную реализацию с нуля
(`new Printable[Box]` и т.д.),
создайте собственный экземпляр из
существующего с помощью `contramap`.

<div class="solution">
Чтобы сделать экземпляр универсальным для всех типов `Box`,
нам нужно брать за основу экземпляр `Printable` для типа внутри `Box`.
Мы можем либо написать полную реализацию вручную:

```tut:book:silent
implicit def boxPrintable[A](implicit p: Printable[A]) =
  new Printable[Box[A]] {
    def format(box: Box[A]): String =
      p.format(box.value)
  }
```

либо использовать `contramap`, чтобы создать новый экземпляр
с помощью неявного параметра:

```tut:book:silent
implicit def boxPrintable[A](implicit p: Printable[A]) =
  p.contramap[Box[A]](_.value)
```

Использовать `contramap` намного проще
и отражает функциональный подход
к решению задач путём комбинирования простых блоков,
используя чистые функциональные комбинаторы.
</div>

Ваш пример должен работать следующим образом:

```tut:book
format(Box("hello world"))
format(Box(true))
```

Если у вас нет `Printable` для типа внутри `Box`,
вызов `format` должен привести к ошибке компиляции:

```tut:book:fail
format(Box(123))
```

### Инвариантные функторы и метод *imap* {#sec:functors:invariant}

*Инвариантные функторы* реализуют метод `imap`,
который неформально эквивалентен
комбинации `map` и `contramap`.
Если `map` позволяет определить новый экземпляр тайпкласса путём
добавления функции к последовательности,
и `contramap` позволяет определить их с помощью
применения предварительных операций к последовательности,
то с imap новый экземпляр определяется при помощи
пары двунаправленных преобразований.

Самыми интуитивно понятными примерами являются такие тайпклассы,
которые представляют кодирование и декодирование некоторых типов данных,
такие как JSON [`Format`][link-play-json-format] в Play
и [`Codec`][link-scodec-codec] в scodec.
Мы можем написать свой `Codec`, расширив `Printable`,
для поддержки как кодирования в `String`, так и декодирования обратно:

```tut:book:silent
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): A
  def imap[B](dec: A => B, enc: B => A): Codec[B] = ???
}
```

```tut:book:invisible
trait Codec[A] {
  self =>

  def encode(value: A): String
  def decode(value: String): A

  def imap[B](dec: A => B, enc: B => A): Codec[B] =
    new Codec[B] {
      def encode(value: B): String =
        self.encode(enc(value))

      def decode(value: String): B =
        dec(self.decode(value))
    }
}
```

```tut:book:silent
def encode[A](value: A)(implicit c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(implicit c: Codec[A]): A =
  c.decode(value)
```

Типовая диаграмма для `imap` показана на
рисунке [@fig:functors:imap-type-chart].
Если у нас есть `Codec[A]`
и пара функиций `A => B` и `B => A`,
метод `imap` создаёт `Codec[B]`:

![Типовая диаграмма: метод imap](src/pages/functors/generic-imap.pdf+svg){#fig:functors:imap-type-chart}

В качестве примера, представьте, что у нас есть базовый `Codec[String]`,
методы `encode` и `decode` возвращают точно то значение, которое принимают:

```tut:book:silent
implicit val stringCodec: Codec[String] =
  new Codec[String] {
    def encode(value: String): String = value
    def decode(value: String): String = value
  }
```

Мы можем создать много полезных `Codec` для других типов,
преобразуя `stringCodec` с помощью `imap`:

```tut:book:silent
implicit val intCodec: Codec[Int] =
  stringCodec.imap(_.toInt, _.toString)

implicit val booleanCodec: Codec[Boolean] =
  stringCodec.imap(_.toBoolean, _.toString)
```

<div class="callout callout-info">
*Обработка ошибок*

Обратите внимание, что метод `decode` нашего тайпкласса `Codec`
не обрабатывает ошибки.
Если мы хотим смоделировать более сложное поведение,
то можем выйти за пределы функторов
и посмотреть на *линзы* и *оптики*.

Оптики выходят за рамки этой книги.
Однако, библиотека [Monocle][link-monocle],
разработанная Julien Truffaut, является отличной
отправной точкой для дальнейшего изучения.
</div>

#### Мышление преобразованиями с *imap*

Реализуйте метод `imap` для `Codec` выше.

<div class="solution">
Ниже представлена рабочая реализация:

```tut:book:silent:reset
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): A

  def imap[B](dec: A => B, enc: B => A): Codec[B] = {
    val self = this
    new Codec[B] {
      def encode(value: B): String =
        self.encode(enc(value))

      def decode(value: String): B =
        dec(self.decode(value))
    }
  }
}
```

```tut:book:invisible
implicit val stringCodec: Codec[String] =
  new Codec[String] {
    def encode(value: String): String = value
    def decode(value: String): String = value
  }

implicit val intCodec: Codec[Int] =
  stringCodec.imap[Int](_.toInt, _.toString)

implicit val booleanCodec: Codec[Boolean] =
  stringCodec.imap[Boolean](_.toBoolean, _.toString)

def encode[A](value: A)(implicit c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(implicit c: Codec[A]): A =
  c.decode(value)
```
</div>

Покажите, что ваш метод `imap` позволяет
создать `Codec` для `Double`.

<div class="solution">
Мы можем реализовать это, используя
метод `imap` у `stringCodec`:

```tut:book:silent
implicit val doubleCodec: Codec[Double] =
  stringCodec.imap[Double](_.toDouble, _.toString)
```
</div>

Наконец, реализуйте `Codec` для следующего типа `Box`:

```tut:book:silent
case class Box[A](value: A)
```

<div class="solution">
Нам нужен общий `Codec` для `Box[A]` для любого `A`.
Мы создаём его, вызывая метод `imap` у `Codec[A]`,
который оказывается в области видимости через неявный параметр:

```tut:book:silent
implicit def boxCodec[A](implicit c: Codec[A]): Codec[Box[A]] =
  c.imap[Box[A]](Box(_), _.value)
```
</div>

Ваши примеры должны работать следующим образом:

```tut:book
encode(123.4)
decode[Double]("123.4")

encode(Box(123.4))
decode[Box[Double]]("123.4")
```

<div class="callout callout-warning">
*Что с названиями?*

Какова связь между терминами
"контравариатность", "инвариантность", и "ковариантность"
и этими разными типами функторов?

Если вы помните из раздела [@sec:variance],
вариантность влияет на подтипирование,
что, по сути, является нашей способностью использовать значение одного типа 
вместо значения другого,
не ломая код.

Подтипирование можно рассматривать как преобразование.
Если `B` — это подтип `A`,
то мы всегда можем преобразовать `B` в `A`.

Аналогично, можно сказать что `B` — это подтип `A`
если существует функция `A => B`.
Стандартный ковариантный функтор рассматривает именно этот случай.
Если `F` — это ковариантный функтор,
где бы у нас ни было `F[A]` и функции `A => B`,
мы всегда сможем получить `F[B]`.

Контрвариантный функтор рассматривает противоположный случай.
Если `F` — это контрвариантный функтор,
где бы у нас ни было `F[A]` и функции `B => A`
мы всегда сможем получить `F[B]`.

Наконец, инвариантные функторы рассматривают случай, где
мы можем преобразовать `F[A]` в `F[B]`
с помощью функции `A => B`
и обратно с помощью функции `B => A`.
</div>
