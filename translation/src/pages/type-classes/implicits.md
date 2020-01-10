## Работа с неявными параметрами

```tut:book:invisible
// Тайпкласс и экземпляры, определённые ранее

sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json

trait JsonWriter[A] {
  def write(value: A): Json
}

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

import JsonWriterInstances._

object Json {
  def toJson[A](value: A)(implicit w: JsonWriter[A]): Json =
    w.write(value)
}
```

Работа с тайпклассами в Scala 
подразумевает работу с неявными параметрами и их значениями.
Чтобы быть эффективными в этом, следует знать несколько правил.

### Упаковка implicit-значений

Любопытной особенностью Scala является то, 
что любые определения, помеченные как `implicit`, должны быть расположены 
внутри объекта или трейта, а не на верхнем уровне модуля.
В приведенном выше примере мы упаковали наши экземпляры тайпкласса
в объект с именем `JsonWriterInstances`.
Мы могли бы также поместить их 
в объект-компаньон `JsonWriter`.
Размещение экземпляров в объекте-компаньоне тайпкласса
имеет особое значение в Scala, 
потому что такой объект является частью неявного контекста (implicit scope).

### Неявный контекст

Как мы уже убедились, компилятор ищет
кандидатов в экземпляры тайпкласса на основании их типа.
Например, в следующем выражении
он будет искать экземпляр, который имеет тип

`JsonWriter[String]`:

```tut:book:silent
Json.toJson("A string!")
```

Компилятор ищет экземпляры-кандидаты
в *неявном контексте* по месту вызова, 
которое приближённо состоит из:

- локальных или унаследованных определений;

- импортированных определений;

- определения в объекте-компаньоне тайпкласса
  или объекте-компаньоне самого типа 
  (в данном случае — `JsonWriter` или `String`).

Определения включаются в неявный контекст только в том случае,
если они помечены ключевым словом `implicit`.
Кроме того, если компилятор встречает несколько определений кандидатов одного типа,
он прекращает компиляцию с ошибкой о том, 
что *`implicit`-значения определены неоднозначно* (ambiguous implicit values):

```scala
implicit val writer1: JsonWriter[String] =
  JsonWriterInstances.stringWriter

implicit val writer2: JsonWriter[String] =
  JsonWriterInstances.stringWriter

Json.toJson("A string")
// <console>:23: error: ambiguous implicit values:
//  both value stringWriter in object JsonWriterInstances of type => JsonWriter[String]
//  and value writer1 of type => JsonWriter[String]
//  match expected type JsonWriter[String]
//          Json.toJson("A string")
//                     ^
```

Точные правила поиска значений неявных параметров существенно сложнее, 
но эта сложность должна рассматриваться за пределами данной книги [^implicit-search].
Нашим же потребностям отвечают четыре основных способа разместить экземпляры тайпклассов:

1. в объекте, подобно `JsonWriterInstances`;
2. в трейте;
3. в объекте-компаньоне тайпкласса;
4. в объекте-компаньоне нужного нам типа.

В варианте 1 мы добавляем экземпляры в область видимости, явно импортируя их.
В варианте 2 мы вводим их в область видимости в результате наследования.
В вариантах 3 и 4 экземпляры *всегда* находятся в неявном контексте,
независимо от того, где мы пытаемся их использовать.

[^implicit-search]: Если вас заинтересуют более тонкие правила 
поиска значений неявных параметров в Scala,
обратите внимание на [этот ответ о неявном контексте на Stack Overflow][link-so-implicit-scope]
и [этот пост о приоритетах в поиске неявных параметров][link-implicit-priority].

### Рекурсивный поиск неявных параметров {#sec:type-classes:recursive-implicits}

Сила тайпклассов и неявных параметров происходит 
из способности компилятора *комбинировать* определения `implicit`-значений 
при поиске возможных вариантов.

Ранее мы говорили об экземплярах тайпклассов так, 
будто они всегда объявляются как `implicit val`. 
Это было упрощением.
Мы можем определять экземпляры двумя способами:

1. определяя конкретные экземпляры 
   как `implicit val` требуемого типа[^implicit-objects];

2. определяя `implicit`-методы для создания экземпляров 
   нужного типа из экземпляров другого типа.

[^implicit-objects]: К этому способу относятся и объявления `implicit object`.

Зачем нам может понадобиться создавать одни экземпляры из других?
В качестве мотивационного примера 
представим себе определение `JsonWriter` для `Option`.
Очевидно, нам понадобится `JsonWriter[Option[A]]` 
для каждого типа `A`, который будет использоваться в приложении.
Мы могли бы попытаться решить проблему «в лоб», 
создав библиотеку таких `implicit`-значений:

```scala
implicit val optionIntWriter: JsonWriter[Option[Int]] =
  ???

implicit val optionPersonWriter: JsonWriter[Option[Person]] =
  ???

// и т.д.
```

Однако, совершенно очевидно, что такой подход не масштабируется.
Кончится тем, что мы объявим по два `implicit`-значения 
для каждого типа `A` в нашем приложении: 
одно для `A` и одно для `Option[A]`.

К счастью, мы можем абстрагировать код для работы с любыми `Option[A]` 
в общий конструктор, основанный на конкретных экземплярах для типа `A`:

- если `option` представлен значением `Some(aValue)`, 
  то обрабатываем значение `aValue`, используя writer для `A`;

- если `option` представлен значением `None` — возвращаем `JsNull`.

Вот как это реализуется посредством `implicit def`:

```tut:book:silent
implicit def optionWriter[A]
    (implicit writer: JsonWriter[A]): JsonWriter[Option[A]] =
  new JsonWriter[Option[A]] {
    def write(option: Option[A]): Json =
      option match {
        case Some(aValue) => writer.write(aValue)
        case None         => JsNull
      }
  }
```

Этот метод *создает*  экземпляр `JsonWriter` для `Option[A]`, 
полагаясь на неявный параметр в том, чтобы реализовать функциональность, 
специфичную для типа `A`.
Когда компилятор встречает выражение вроде такого:

```tut:book:silent
Json.toJson(Option("A string"))
```

он ищет значение для неявного параметра `JsonWriter[Option[String]]`.
Он находит `implicit`-метод для `JsonWriter[Option[A]]`:

```tut:book:silent
Json.toJson(Option("A string"))(optionWriter[String])
```

и затем рекурсивно ищет значение типа `JsonWriter[String]`, 
которое он мог бы использовать в качестве параметра для `optionWriter`:

```tut:book:silent
Json.toJson(Option("A string"))(optionWriter(stringWriter))
```

Таким образом, поиск одного определения для неявного параметра 
становится поиском в среди возможных комбинаций таких определений 
с целью подобрать одну комбинацию, 
которая даст нужный итоговый тип.

<div class="callout callout-warning">
*Неявные преобразования*

Когда вы создаете конструктор экземпляра тайпкласса, 
используя `implicit def`, 
обязательно пометьте параметры этого метода 
как `implicit` параметры.
Без этого ключевого слова компилятор 
не сможет подобрать для них значения.

`implicit`-методы в сочетании с явными параметрами, 
тем временем, образуют в Scala другой паттерн — *неявное преобразование*. 
Этот паттерн отличается и от описанного в предыдущем разделе *интерфейсного синтаксиса*, 
поскольку в его случае `JsonWriter` являлся бы `implicit`-классом с *методами расширения*.
Неявные преобразования — это устаревший паттерн, 
и он не одобряется в современном коде Scala.
К счастью, компилятор предупредит вас, если вы захотите им воспользоваться.
Для его использования вам придётся вручную включить неявные преобразования, 
импортировав `scala.language.implicitConversions` в ваш файл:

```tut:book:fail
implicit def optionWriter[A]
    (writer: JsonWriter[A]): JsonWriter[Option[A]] =
  ???
```
</div>
