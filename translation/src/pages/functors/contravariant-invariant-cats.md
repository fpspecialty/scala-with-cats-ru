## *Контравариантность* и инвариантность в Cats

Давайте рассмотрим реализацию
контравариантных и инвариантных функторов в Cats,
представляемых тайпклассами [`cats.Contravariant`][cats.Contravariant]
и [`cats.Invariant`][cats.Invariant].
Ниже представлена упрощённая версия кода:

```tut:book:invisible
import scala.language.higherKinds
```

```tut:book:silent
trait Contravariant[F[_]] {
  def contramap[A, B](fa: F[A])(f: B => A): F[B]
}

trait Invariant[F[_]] {
  def imap[A, B](fa: F[A])(f: A => B)(g: B => A): F[B]
}
```

### Контравариантность в Cats

Мы подбираем экземпляры `Contravariant`,
используя метод `Contravariant.apply`.
Cats предоставляет экземпляры для типов данных, которые принимают параметры,
включая `Eq`, `Show`, и `Function1`.
Например:

```tut:book:silent:reset
import cats.Contravariant
import cats.Show
import cats.instances.string._

val showString = Show[String]

val showSymbol = Contravariant[Show].
  contramap(showString)((sym: Symbol) => s"'${sym.name}")
```

```tut:book
showSymbol.show('dave)
```

Для удобства мы можем использовать
[`cats.syntax.contravariant`][cats.syntax.contravariant],
который предоставляет метод расширения `contramap`:

```tut:book:silent
import cats.syntax.contravariant._ // для contramap
```

```tut:book
showString.contramap[Symbol](_.name).show('dave)
```

### Инвариантность в Cats

Среди прочих типов,
Cats предоставляет экземпляр `Invariant` для `Monoid`.
Это немного отличается от примера с `Codec`,
который был представлен в разделе [@sec:functors:invariant].
Если вы помните, это то, как выглядит `Monoid`:

```scala
package cats

trait Monoid[A] {
  def empty: A
  def combine(x: A, y: A): A
}
```

Представьте, что мы хотим создать экземпляр `Monoid`
для типа [`Symbol`][link-symbol] в Scala.
Cats не предоставляет `Monoid` для `Symbol`,
но предоставляет `Monoid` для похожего типа: `String`.
Мы можем написать полугруппу с
методом `empty`, который возвращает пустую `String`,
и методом `combine`, который работает следующим образом:

1. принимает два параметра `Symbol`;
2. преобразует `Symbol` в `String`;
3. соединяет `String`, используя `Monoid[String]`;
4. преобразует результат обратно в `Symbol`.

Мы можем реализовать `combine`, используя `imap`,
передавая функции типа `String => Symbol`
и `Symbol => String` как параметры.
Ниже представлен код, который написан с помощью
метода расширения `imap`,
представленного в `cats.syntax.invariant`:

```tut:book:silent
import cats.Monoid
import cats.instances.string._ // for Monoid
import cats.syntax.invariant._ // for imap
import cats.syntax.semigroup._ // for |+|

implicit val symbolMonoid: Monoid[Symbol] =
  Monoid[String].imap(Symbol.apply)(_.name)
```

```tut:book
Monoid[Symbol].empty

'a |+| 'few |+| 'words
```
