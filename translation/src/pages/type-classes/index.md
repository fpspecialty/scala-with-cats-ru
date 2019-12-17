# Введение {#sec:type-classes}

Cats содержит широкий ассортимент средств функциональных программирования
и позволяет разработчикам выбирать, какие из них использовать.
Большая часть этих инструментов предоставляется в виде *классов типов (тайпклассы, type classes)*,
которые мы можем применять к существующим типам Scala.

Классы типов — это паттерн программирования, пришедший из Haskell[^type-class-defn].
Они позволяют нам расширять существующие библиотеки новой функциональностью,
не используя традиционное наследование
и не изменяя исходный код библиотек.

<!--
Type classes work well with another programming pattern: *algebraic data types*.
These are closed systems of types that we use to represent data or concepts.
Because the systems are closed (and therefore cannot be extended by other users),
we can process them using pattern matching
and the compiler will check the exhaustiveness of our case clauses.

There are two other patterns we need to cover in this chapter.
*Value classes* provide a way to wrap up
generic data types like `Strings` and `Ints`
and give them specific meanings in a given context.
The extra type information is useful when type classes.
*Type aliases* are another pattern that
provide aliases for large, complex types.
-->

В этой главе мы освежим в памяти информацию о классах типов
из книги Underscore [Essential Scala][link-essential-scala]
и взглянем на кодовую базу Cats.
Мы рассмотрим два класса типов — `Show` и `Eq` — 
чтобы на их примере познакомиться с паттернами, лежащими в основе всего, что описано в этой книге.

В конце рассмотрим классы типов в связке с алгебраическими типами данных, 
сопоставлением с образцом, классами значений и псевдонимами типов,
и таким образом представим структурированный подход к функциональному программированию в Scala.

[^type-class-defn]: Слово «класс» не обязательно должно означать `class` в языке Scala или Java.
