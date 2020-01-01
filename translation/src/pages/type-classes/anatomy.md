## Анатомия Тайпклассов

Паттерн «тайпкласс» состоит из трёх важных компонентов: сам *тайпкласс*, *экземпляры (instances)* и методы интерфейса, доступного пользователю.

### Тайпкласс

*Тайпкласс* — это интерфейс или API, предоставляющий некоторую функциональность, которую мы хотим реализовать. В Cats тайпкласс представляет собой trait как минимум с одним типовым параметром. Например, мы можем представить обобщённую возможность «сериализации в JSON» следующим образом:

```tut:book:silent
// Упрощённое определение AST JSON
sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json

// Поведение, связанное с сериализацией в JSON, описывается этим trait
trait JsonWriter[A] {
  def write(value: A): Json
}
```

`JsonWriter` в этом примере является тайпклассом,
а `Json` и его подтипы играют вспомогательную роль.

### Экземпляры тайпклассов

*Экземпляры* тайпкласса 
предоставляют реализацию для нужных нам типов,
включая типы из стандартной библиотеки Scala
и типы, описывающие нашу предметную область.

В Scala мы определяем экземпляры, создавая
конкретные реализации тайпкласса
и помечая их ключевым словом `implicit`:

```tut:book:silent
final case class Person(name: String, email: String)

object JsonWriterInstances {
  implicit val stringWriter: JsonWriter[String] =
    new JsonWriter[String] {
      def write(value: String): Json =
        JsString(value)
    }

  implicit val personWriter: JsonWriter[Person] =
    new JsonWriter[Person] {
      def write(value: Person): Json =
        JsObject(Map(
          "name" -> JsString(value.name),
          "email" -> JsString(value.email)
        ))
    }

  // и т.д.
}
```

### Интерфейсы тайпклассов

*Интерфейс* тайпкласса — это любая функциональность, которую мы предоставляем пользователям.
Интерфейсы представляют собой обобщённые методы, которые принимают
экземпляры тайпкласса в качестве `implicit` параметров.

Два распространенных способа определения интерфейса:
*Интерфейсные объекты* (interface Objects) и *Интерфейсный синтаксис* (interface Syntax).

**Интерфейсные объекты**

Самый простой способ определить интерфейс —
это разместить методы в объекте-синглтоне:

```tut:book:silent
object Json {
  def toJson[A](value: A)(implicit w: JsonWriter[A]): Json =
    w.write(value)
}
```

Чтобы использовать этот объект, мы импортируем нужные нам экземпляры тайпклассов
и вызываем соответствующий метод:

```tut:book:silent
import JsonWriterInstances._
```

```tut:book
Json.toJson(Person("Dave", "dave@example.com"))
```

Компилятор обнаруживает, что мы вызываем метод `toJson`
без предоставления `implicit`-параметров.
Он пытается исправить это, выполняя поиск экземпляров тайпкласса
для соответствующих типов и подставляя их в вызов:

```tut:book:silent
Json.toJson(Person("Dave", "dave@example.com"))(personWriter)
```
**Интерфейсный синтаксис**

В качестве альтернативы мы можем использовать *методы расширения*,
чтобы дополнить существующие типы интерфейсными методами [^pimping].
В Cats это называется *«синтаксисом»* тайпкласса:

[^ pimping]: иногда методы расширения
упоминаются как «обогащение типа» (type enrichment) или «pimping».
Это устаревшие термины, мы их больше не используем.

```tut:book:silent
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit w: JsonWriter[A]): Json =
      w.write(value)
  }
}
```

Чтобы использовать интерфейсный синтаксис, мы явно импортируем его, 
помимо экземпляров для нужных нам типов:

```tut:book:silent
import JsonWriterInstances._
import JsonSyntax._
```

```tut:book
Person("Dave", "dave@example.com").toJson
```

Подобно предыдущему примеру, компилятор ищет кандидатов
на роль неявных параметров и подставляет их за нас:

```tut:book:silent
Person("Dave", "dave@example.com").toJson(personWriter)
```

**Метод _implicitly_**

Стандартная библиотека Scala предоставляет
обобщённый интерфейс тайпкласса — `implicitly`.
Он определён очень просто:

```scala
def implicitly[A](implicit value: A): A =
  value
```

Мы можем использовать `implicitly`, чтобы «притянуть» любое значение из неявного окружения (implicit scope).
От нас требуется называть желаемый тип, а `implicitly` сделает все остальное:

```tut:book
import JsonWriterInstances._

implicitly[JsonWriter[String]]
```

Большинство тайпклассов в Cats предоставляют другие средства взаимодействия с экземплярами.
Однако `implicitly` хорош как запасной, «отладочный» приём.
Мы можем вставить вызов `implicitly` в код, над которым работаем,
чтобы убедиться, что компилятор сможет найти экземпляр тайпкласса
и не произойдёт ошибок, связанных с неоднозначно определёнными `implicit`-значениями.
