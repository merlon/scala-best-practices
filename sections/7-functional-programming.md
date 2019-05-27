## 7. Functional Programming

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

These functional rules generally transcend programming language and/or platform
barrier. Functional abstractions can make your code simpler and easier to read.
A deep knowledge of FP is not required (but oh-boy, it is enlightening) and
anyone should be able to utilize these to their benefit. On top of that, in comparison
with actor frameworks, it provides unmatched levels of type-safety and encourages
writing safer code often checked at compile-time.

Scala has several ([cats](https://typelevel.org/cats/), [scalaz](https://scalaz.github.io/7/))
great functional programming libraries which greatly enhance functional programming
experience.

### 7.1 SHOULD embrace *Functor* capabilities

[*Functor*](https://typelevel.org/cats/typeclasses/functor.html) is a simple abstraction
which enhances your context-bounded computation
programming experience defined by `map` function. By *context* are meant data-types
such as `Future[_]`, `Option[_]`, `Either[E, _]`, `IO[_]`, `List[_]` (and similar
collections), and many more. As most other functional abstractions, it is expressed
in form of a [*type-class*](https://scalac.io/typeclasses-in-scala/).

Generally speaking, *Functor* can be any data type which:

1) Takes yet another data type as its parameter to form a type, hence the `[_]`,
2) satisfies these [laws](https://typelevel.org/cats/typeclasses/functor.html).

Speaking of laws and a common sense, these do not exist to obfuscate anyone but
maintain a well-behaving `map` implementation. For example:

1) Can `map` function change the order of elements in a `List[_]`? No it can't.
   It would make it useless.
2) Is `Set[_]` a *Functor*? No because its `map` might erase duplicated values,
   hence making it useless.

Note that most of the data types mentioned above already provide `map` function
but *Functor*, on top of that, provides helpful utility functions that can make
your code more readable and easier to write. Consider the following example:

```scala
val cleanF = Future(cleanOldReports(reportConfig)).map(_ => Done)
cleanF.onComplete {
  case Success(_) => logger.debug("Old report files successfully deleted")
  case Failure(cause) => logger.error("An error occurred while removing old report files", cause)
}
cleanF
```

What is happening here:

1) `cleanOldReports` which is not pure in this case, is being lifted into the
   `Future` context.
2) The result of this computation is being thrown away by `map` and substituted with
   `Done` instead. This is such a common pattern which has been abstracted
   away into the [`as`](https://typelevel.org/cats/api/cats/Functor.html#as[A,B](fa:F[A],b:B):F[B]) function:

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

This (though very trivial) example shows how you can utilize the true power of
*Functor*. The same functorial capabilities are available to you for the rest
of the data types mentioned above as well. To see full list of what it can do
see [this](https://typelevel.org/cats/api/cats/Functor$.html).

Next, consider the following example demonstrating functorial laws in action.
It is a part of an `akka.Actor`:

```scala
protected def getActiveReports = context.children.toSet - reportRouter - statisticsCollector

override def receive = LoggingReceive {
  case GetActiveReports => {
    val apiSender = sender()
    val statuses = Future
      .sequence(
        getActiveReports
          .map(_ ? MonitorReportStatus)
          .map(_.mapTo[ReportStatus].map(rs => (rs.reportId, rs.status)))
      )
      .map(rep => ActiveReports(rep.toMap))

    statuses pipeTo apiSender
    ()
  }
```

When reading through this code, which tries to collect `ReportStatus` from
workers currently generating customer reports, your senses may start *tingling*.
And they probably should since `map` (from standard library) over a `Set[_]` is
performed:

```scala
getActiveReports
  .map(_ ? MonitorReportStatus)
  .map(_.mapTo[ReportStatus])
```

What is the outcome of this? Do you get:

- All the requested `ReportStatus`es?
- Or maybe one with `Future.Successful` and one with `Future.failed` outcome?
- Or maybe just one `ReportStatus` (possibly the last one)?

The answer is non-trivial. It depends on how equality is defined for the data type
being carried inside the `Set[_]` (a `Future[_]` in this case). Constructions like this
should be avoided using a different collection instead. To read more on this
topic see [Mapping sets](https://typelevel.org/blog/2014/06/22/mapping-sets.html) and/or
[Fake theorems for free](https://failex.blogspot.com/2013/06/fake-theorems-for-free.html).
