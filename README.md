1. `echo -e 'middleware\n\n' | sbt new http4s/http4s-io.g8 --branch 0.23-scala3  #Output error is ok`

   `SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
   SLF4J: Defaulting to no-operation (NOP) logger implementation
   SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
   name [quickstart]: organization [com.example]: package [com.example.middleware_example]:
   Template applied in middleware`

2. `cd middleware`
3. `sbt run`

   `build.sbt update for java.net.BindException: Address already in use
   Be sure to exit sbt inbetween runs to prevent java.net.BindException: Address already in use.
   This is not an issue in Production environments and can be corrected in build.sbt with Compile / run / fork := true`

4. `curl http://localhost:8080/hello/world  # initial code working!`

5. Add imports to MiddlewareServer.scala
   `import scala.concurrent.duration.DurationInt
   import org.http4s.server.middleware._`

6. Update file src/main/scala/com/example/middleware/MiddlewareServer.scala with http4s provided middleware:
   `// With Middlewares in place
   loggerHttpApp = Logger.httpApp(true, true)(httpApp)
   responseTimingHttpApp = ResponseTiming(loggerHttpApp)
   timeoutHttpApp = Timeout.httpApp[IO](10.seconds)(responseTimingHttpApp)
   finalHttpApp = GZip(timeoutHttpApp)`

7. Retest with curl -v and observe the X-Response-Time header curl -v http://localhost:8080/hello/world

8. Copy in LanguageHeaderValidation.scala to the source directory
`
package com.example.middleware

import cats.data.Kleisli
import cats.syntax.all.*
import cats.Applicative
import org.http4s.*
import org.typelevel.ci.CIString

import scala.util.matching.Regex

object LanguageHeaderValidator:
  private val pattern: Regex = """en-US|fr-US|en-CA|fr-CA""".r

  def apply[F[_]: Applicative, G[_]](http: Http[F, G]): Http[F, G] =
    def badRequest[A](H: org.http4s.Headers, response: A)(using e: EntityEncoder[G, A]): Response[G] =
      Response(status = Status.InternalServerError).withEntity(response)

    Kleisli { (req: Request[G]) =>
      val ciHeader = CIString("language")

      req.headers.get(ciHeader) match
        case None => http(req) // no header
        case Some(values) =>
          val headerValue = values.head.value // first value

          headerValue match
            case pattern() => http(req)
            case _ =>
              val message = s"$ciHeader header invalid"
              badRequest(req.headers, message).pure[F]
    }

  def httpApp[F[_]: Applicative](httpApp: HttpApp[F]): HttpApp[F] =
    apply(httpApp)`

9. Update MiddlewareServer.scala to add a custom LanguageHeaderValidator

   `// With Middlewares in place
   loggerHttpApp = Logger.httpApp(true, true)(httpApp)
   responseTimingHttpApp = ResponseTiming(loggerHttpApp)
   timeoutHttpApp = Timeout.httpApp[IO](10.seconds)(responseTimingHttpApp)
   gzipHttpApp = GZip(timeoutHttpApp)
   finalHttpApp = LanguageHeaderValidator(gzipHttpApp)`

10. curl -v http://localhost:8080/hello/world -H 'language: xyzzy'  # errors with language header invalid.  Header 'language: en-US' processes normally
