## Версии {-}

Эта книга написана для Scala 2.12.3 и Cats 1.0.0.
Вот минимальное содержимое файла `build.sbt`, 
включающее актуальные для данной книги зависимости и настройки[^sbt-version]:

```scala
scalaVersion := "2.12.3"

libraryDependencies +=
  "org.typelevel" %% "cats-core" % "1.0.0"

scalacOptions ++= Seq(
  "-Xfatal-warnings",
  "-Ypartial-unification"
)
```

[^sbt-version]: Мы предполагаем, что вы используете SBT версии 0.13.13 или новее.

### Шаблоны проектов {-}

Чтобы помочь вам поскорее приступить к работе, 
мы создали шаблон Giter8.
Вы можете склонировать его, выполнив команду:

```bash
$ sbt new underscoreio/cats-seed.g8
```

Эта команда создаст пустой проект, 
в зависимостях которого уже установлена библиотека Cats.
Ознакомьтесь с файлом `README.md`,
чтобы узнать, как запустить пример кода
или открыть интерактивную консоль Scala.

Шаблон `cats-seed` минимален.
Если же вы предпочитаете конфигурации, в которых «всё включено»,
попробуйте шаблон `sbt-catalysts` от Typelevel:

```bash
$ sbt new typelevel/sbt-catalysts.g8
```

Такая команда создаст проект 
с набором зависимостей и плагинов компилятора, 
а также шаблонами для модульных тестов 
и документацией [tut][link-tut].

За подробностями обращайтесь 
к страницам проектов [catalysts][link-catalysts] 
и [sbt-catalysts][link-sbt-catalysts].
