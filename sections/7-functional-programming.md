## 7. Functional Programming

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

These functional rules generally transcend programming language and/or platform
barrier. Functional abstractions can make your code simpler and easier to read.
A deep knowledge of FP (but oh boy, it is enlightening) is not required and
anyone should be able to utilize these to their benefit. On top of that, in comparison
with actor frameworks, it provides unmatched levels of type-safety and encourages
writing safer code often checked at compile-time.

Scala has several ([cats](https://typelevel.org/cats/), [scalaz](https://scalaz.github.io/7/))
great functional programming libraries which greatly enhance functional programming
experience.

### 7.1 SHOULD embrace *Functor* capabilities

*Functor* is a simple abstraction which enhances your context-bounded computation
programming experience defined by `map` function. By *context* are meant data-types
such as `Future[_]`, `Option[_]`, `Either[E, _]`, `IO[_]`, `List[_]` (and similar
collections), and many more.

Generally speaking, *Functor* can be any data type which:

1) takes yet another data type as its parameter to form a type, hence the `[_]`,
2) satisfies these [laws](https://typelevel.org/cats/typeclasses/functor.html).

Speaking of laws, these do not exist to obfuscate anyone but maintain a
well-behaving `map` implementation.

- For example #1: Can `map` function change the order of elements in a `List[_]`?
  No it can't. It would make it useless.
- For example #2: Is `Set[_]` a *Functor*? No because its `map` might erase
  duplicated values, hence making it useless.

Note that most of of the data types mentioned above already provide `map` function
but *Functor* provides helpful utility functions that can make your code more readable.
Consider the following example:

```scala
val cleanF = Future(cleanOldReports(reportConfig)).map(_ => Done)
cleanF.onComplete {
  case Success(_) => logger.debug("Old report files successfully deleted")
  case Failure(cause) => logger.error("An error occurred while removing old report files", cause)
}
cleanF
```

What is happening here:

1) `cleanOldReports` which is not pure in this case is being lifted into the
   `Future` context.
2) The result of this computation is being thrown away and substituted with
   `Done` instead. This is such a common pattern which has been abstracted
   away into the `as` function:

```scala
import cats.instances.future._
import cats.syntax.functor._

val cleanF = Future(cleanOldReports(reportConfig)).as(Done)
cleanF.onComplete {
  case Success(_) => logger.debug("Old report files successfully deleted")
  case Failure(cause) => logger.error("An error occurred while removing old report files", cause)
}
cleanF
```

This example shows how you can utilize the true power of *Functor*. The
same syntax is available to you for rest of the data types mentioned above
as well.

Although a very trivial example, it truly starts to shine its power when composed
together with other abstractions which FP is all about: *composition-over-inheritance*.
See other practices in this section.
