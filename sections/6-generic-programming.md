## 6. Generic Programming

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 6.1. MUST NOT use automatic typeclass instance derivation

When using [generic programming](https://github.com/milessabin/shapeless) or working with
libraries based on it (such as [io.circe](https://circe.github.io/circe/codecs/auto-derivation.html)) avoid
using **automatic derivation** features. An anti-pattern that often comes up look like this:

```scala
// Beware such imports!
import io.circe.generic.auto._
```

or

```scala
import io.circe.generic.AutoDerivation

object Model extends AutoDerivation {
    case class GraphQLId(value: String)
    case class User(name: String, login: String)
    case class Comment(createdAt: OffsetDateTime, user: Option[User], commentText: Option[String])

    // ... and many more classes
}
```

In the example above `AutoDerivation` (`import io.circe.generic.auto._`) brings 
into scope macros that automatically generate `Encoder` and `Decoder` 
instances for case classes in scope at compile time. It has the following drawbacks:

- It is hard to control what is being generated
- It is hard to reason about what is being really used
- Might result in **a long** [compilation time](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html) 
  (in order to resolve an implicit a macro might get [expanded, resolved and thrown away](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html#the-cost-of-implicit-macros))

Always use explicitly derived type-class instances. Preferably on a `Support` trait:

```scala
import io.circe.Decoder
import io.circe.generic.semiauto.deriveDecoder

/** Reporting related Json instances.
  *
  * @groupname comment Comment
  * @groupname common Common
  * @groupname user User
  */
trait JsonSupport {

  /** @group comment */
  implicit val commentDecoder: Decoder[Comment] = deriveDecoder

  /** @group common */
  implicit val graphQlIdDecoder: Decoder[GraphQLId] = Decoder[String] map GraphQLId.apply

  /** @group user */
  implicit val userDecoder: Decoder[User] = deriveDecoder
}

object JsonSupport extends JsonSupport
```
