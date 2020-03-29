## Отступление: Обработка ошибок и MonadError

Cats предоставляет дополнительный тайпкласс `MonadError`,
который абстрагирует `Either`-подобные типы данных,
использующиеся для обработки ошибок.
`MonadError` предоставляет дополнительные операции
для создания и обработки ошибок.

<div class="callout callout-info">
*Эта глава необязательна!*

Вам нужно использовать `MonadError`
только если вы хотите абстрагироваться от монад, обрабатывающих ошибки.
Например, мы можете использовать `MonadError`
для абстракции над `Future` и `Try`,
или над `Either` и `EitherT`
(с которым мы познакомимся в разделе [@sec:monad-transformers]).

Если вы не нуждаетесь в подобной абстрации прямо сейчас,
можете переходить к главе [@sec:monads:eval].
</div>

### Тайпкласс «MonadError»

Ниже представлена упрощённая версия
определения `MonadError`:

```scala
package cats

trait MonadError[F[_], E] extends Monad[F] {
  // «Поднятие» ошибки в `F` контекст:
  def raiseError[A](e: E): F[A]

  // Обработка ошибки с возможным восстановлением:
  def handleError[A](fa: F[A])(f: E => A): F[A]

  // Проверка экземпляра `F`,
  // которая не пройдёт, если предикат не будет удовлетворён:
  def ensure[A](fa: F[A])(e: E)(f: A => Boolean): F[A]
}
```

`MonadError` определен с помощью двух типовых параметров:

- тип монады `F`;
- тип ошибки `E`, содержащейся внутри `F`.

Ниже представлен пример
создания экземпляра тайпкласса для `Either`,
демонстрирующий совместную работу этих параметров:

```tut:book:silent
import cats.MonadError
import cats.instances.either._ // для MonadError

type ErrorOr[A] = Either[String, A]

val monadError = MonadError[ErrorOr, String]
```

<div class="callout callout-warning">
*ApplicativeError*

На самом деле, `MonadError` расширяет другой тайпкласс 
под названием `ApplicativeError`.
Однако, мы познакомимся с `Applicative`
только в разделе [@sec:applicatives].
Семантика будет одинакова для каждого тайпкласса,
поэтому пока можно игнорировать такие подробности.
</div>

### Создание и обработка ошибок

`raiseError` и `handleError` 
являются двумя наиболее важными методами `MonadError`.
`raiseError` подобен методу `pure` тайпкласса `Monad`,
за исключением того, что он создаёт экземпляр, представляющий собой ошибку:

```tut:book
val success = monadError.pure(42)
val failure = monadError.raiseError("Badness")
```

`handleError` дополняет метод `raiseError`.
Он позволяет обработать ошибку и (возможно)
преобразовать её в успешное состояние,
подобно методу `recover` для `Future`:

```tut:book
monadError.handleError(failure) {
  case "Badness" =>
    monadError.pure("It's ok")

  case other =>
    monadError.raiseError("It's not ok")
}
```

Полезный третий метод `ensure` 
реализует `filter`-подобное поведение.
Он проверяет успешное значение монады на соответствие предикату
и создаёт ошибку, если предикат вернул `false`:

```tut:book:silent
import cats.syntax.either._ // для asRight
```

```tut:book
monadError.ensure(success)("Number too low!")(_ > 1000)
```

Cats предоставляет методы `raiseError` и `handleError`
с помощью пакета [`cats.syntax.applicativeError`][cats.syntax.applicativeError]
и метод `ensure` с помощью пакета [`cats.syntax.monadError`][cats.syntax.monadError]:

```tut:book:silent
import cats.syntax.applicative._      // для pure
import cats.syntax.applicativeError._ // для raiseError etc
import cats.syntax.monadError._       // для ensure
```

```tut:book
val success = 42.pure[ErrorOr]
val failure = "Badness".raiseError[ErrorOr, Int]
success.ensure("Number to low!")(_ > 1000)
```

Также существуют другие полезные варианты этих методов.
В исходном коде [`cats.MonadError`][cats.MonadError]
и [`cats.ApplicativeError`][cats.ApplicativeError]
вы найдёте более подробную информацию.

### Экземпляры MonadError

Cats предоставляет экземпляры `MonadError`
для различных типов данных, включая
`Either`, `Future`, и `Try`.
Экземпляры для `Either` являются настриваемыми под любой тип ошибки,
в то время как экземпляры для `Future` и `Try`
всегда представляют ошибки как `Throwable`:

```tut:book:silent
import scala.util.Try
import cats.instances.try_._ // для MonadError

val exn: Throwable =
  new RuntimeException("It's all gone wrong")
```

```tut:book
exn.raiseError[Try, Int]
```

### Упражнение: Абстрагирование