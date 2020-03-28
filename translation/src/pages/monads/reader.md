## Монада Reader ("Читатель") {#sec:monads:reader}

Монада [`cats.data.Reader`][cats.data.Reader]
позволяет создавать последовательность операций, зависящих от некоторых общих входных данных.
Экземпляры `Reader` оборачивают функции одного аргумента,
предоставляя методы для их композиции.

Монада `Reader` часто используется для внедрения зависимостей (dependency injection).
В случае, когда у нас есть набор операций,
зависящих от общей внешней конфигурации,
мы можем скомпозировать их в последовательность с помощью `Reader`.
В результате мы получим одну большую операцию,
принимающую конфигурацию в качестве параметра
и запускающую нашу программу в указанном порядке.

### Создание и распаковка Reader

Экземпляр `Reader[A, B]` можно создать из функции `A => B`,
используя конструктор `Reader.apply`:

```tut:book:silent
import cats.data.Reader
```

```tut:book
case class Cat(name: String, favoriteFood: String)

val catName: Reader[Cat, String] =
  Reader(cat => cat.name)
```

Мы можем извлечь функцию обратно,
используя метод `run`,
и вызвать её через `apply`, как обычно:

```tut:book
catName.run(Cat("Garfield", "lasagne"))
```

Пока что всё просто,
но какие преимущества даёт `Reader` перед обычными функциями?

### Композиция Reader

Мощь `Reader` - в методах `map` и `flatMap`,
которые предоставляют другие виды композиции функций.
Обычно создается набор экземпляров `Reader`,
принимающие одинаковый тип конфигурации,
далее они композируются с помощью `map` и `flatMap`,
и, наконец, вызывается `run` для внедрения конфигурации.

Метод `map` просто расширяет вычисление в `Reader`,
передавая результат через функцию:

```tut:book:silent
val greetKitty: Reader[Cat, String] =
  catName.map(name => s"Hello ${name}")
```

```tut:book
greetKitty.run(Cat("Heathcliff", "junk food"))
```

Метод `flatMap` интереснее.
Он позволяет комбинировать читателей, которые зависят от одного входного типа.
Чтобы это проиллюстрировать, расширим наш пример (с приветствием)
кормлением:

```tut:book:silent
val feedKitty: Reader[Cat, String] =
  Reader(cat => s"Have a nice bowl of ${cat.favoriteFood}")

val greetAndFeed: Reader[Cat, String] =
  for {
    greet <- greetKitty
    feed  <- feedKitty
  } yield s"$greet. $feed."
```

```tut:book
greetAndFeed(Cat("Garfield", "lasagne"))
greetAndFeed(Cat("Heathcliff", "junk food"))
```

### Упражнение: Хакинг читателей

Классическое использование `Reader` - программы,
принимающие конфигурацию в качестве параметра.
Давайте закрепим это исчерпывающим примером
простой системы авторизации.
Наша конфигурация будет состоять из двух баз данных:
списка пользователей и списка их паролей:

```tut:book:silent
case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
```

Начнем с создания псевдонима типа `DbReader` для
`Reader`, принимающего `Db` в качестве типового параметра.
Это сократит дальнейший код.

<div class="solution">
Наш псевдоним типа фиксирует тип `Db`,
но оставляет тип результата в качестве типового параметра:

```tut:book:silent
type DbReader[A] = Reader[Db, A]
```
</div>

Теперь напишем методы для создания экземпляров `DbReader`,
которые будут искать имя пользователя по ID пользователя, и
искать пароль по имени пользователя.
Получим следующие сигнатуры типов:

```tut:book:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  ???

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
Запомните: главная идея в том, чтобы оставить внедрение конфигурации на последний шаг.
Это значит, что функции должны принимать конфигурацию в качестве параметра,
и использовать её на конкретной полученной информации о пользователе:

```tut:book:silent
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))

def checkPassword(
      username: String,
      password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))
```

</div>

Наконец, создадим метод `checkLogin`
для проверки пароля по данному userId.
Получим следующую сигнатуру типов:

```tut:book:silent
def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  ???
```

<div class="solution">
Как вы могли ожидать,
здесь мы используем `flatMap` для композиции `findUsername` и `checkPassword`.
Мы используем `pure` для поднятия (lift) `Boolean` до `DbReader[Boolean]` в случае,
когда имя пользователя не было найдено:

```tut:book:silent
import cats.syntax.applicative._ // for pure

def checkLogin(
      userId: Int,
      password: String): DbReader[Boolean] =
  for {
    username   <- findUsername(userId)
    passwordOk <- username.map { username =>
                    checkPassword(username, password)
                  }.getOrElse {
                    false.pure[DbReader]
                  }
  } yield passwordOk
```
</div>

Вы можете использовать `checkLogin` следующим образом:

```tut:book:silent
val users = Map(
  1 -> "dade",
  2 -> "kate",
  3 -> "margo"
)

val passwords = Map(
  "dade"  -> "zerocool",
  "kate"  -> "acidburn",
  "margo" -> "secret"
)

val db = Db(users, passwords)
```

```tut:book
checkLogin(1, "zerocool").run(db)
checkLogin(4, "davinci").run(db)
```

### Когда использовать Reader?

`Reader` предоставляют инструмент для внедрения зависимостей.
Мы описали шаги в программе как экземпляры `Reader`,
упорядочили их с помощью `map` и `flatMap`,
и построили функцию, принимающую зависимость в качестве аргумента.

Есть множество способов реализации внедрения зависимостей в Скала:
от простых техник вроде методов с набором списков параметров,
техник с использованием подразумеваемых параметров и типовых классов,
до таких сложных техник, как cake pattern и фреймворков внедрения зависимостей (DI frameworks).

`Reader` наиболее полезны в следующих случаях:

- нужно построить программу для исполнения в пакетном режиме (batch program),
  которая легко может быть представлена функцией;

- нужно отложить (defer) внедрение (injection) известного параметра
  или набора параметров;

- нужна возможность тестирования
  частей программы в режиме изоляции (?песочнице?).

Реализуя шаги программы в виде экземпляров `Reader`,
мы можем протестировать их так же легко, как чистые функции,
и бонусом получаем доступ к комбинаторам `map` и `flatMap`.

Для более сложных задач, где мы имеем дело с множеством зависимостей,
или где программу нелегко представить в виде чистой функции,
другие способы внедрения зависимостей, скорее всего, подойдут лучше.

<div class="callout callout-warning">
  *Стрелки Клейсли*

  Возможно, вы заметили в консольном выводе,
  что `Reader` реализован посредством типа `Kleisli`.
  *Стрелки Клейсли* предоставляют более общую форму `Reader`,
  которая обобщает конструктор типа для типа результата.
  Мы столкнемся с `Kleisli` сновa в главе [@sec:monad-transformers].
</div>
