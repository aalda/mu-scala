---
layout: docs
title: Custom gRPC Serialization
section: guides
permalink: /guides/custom-grpc-serialization
---

# Custom gRPC serialization

Mu will serialize [gRPC] requests and responses using [Avro] or [Protobuf],
depending on the compression format specified in the `@service` annotation on
the service definition trait.

This serialization can be customised in a few different ways.

## Compression

Mu supports compression of RPC requests and responses. We can enable this compression either on the server or the client side.

Mu supports `Gzip` as the compression format.

For server side compression, set the compression type argument to `Gzip` in the
`@service` annotation when defining the service.

For example:

```scala mdoc:silent
import higherkindness.mu.rpc.protocol._

object CompressionExample {

  case class HelloRequest(name: String)
  case class HelloResponse(greeting: String)

  @service(Protobuf, Gzip)
  trait Greeter[F[_]] {

    def emptyCompressed(req: HelloRequest): F[HelloResponse]

  }

}
```

To enable compression on the client side, you need to add an option to the client in the channel builder.

Let's see an example of a client with the compression enabled.

```scala mdoc:silent
import cats.effect.{ConcurrentEffect, ContextShift, Resource}
import higherkindness.mu.rpc._
import io.grpc.CallOptions
import CompressionExample._

object CompressionExampleClient {

  val channelFor: ChannelFor = ChannelForAddress("localhost", 12345)

  def clientResource[F[_]: ConcurrentEffect: ContextShift]: Resource[F, Greeter[F]] =
    Greeter.client[F](channelFor, options = CallOptions.DEFAULT.withCompression("gzip"))

}
```

## Custom codecs

Mu allows you to use custom decoders and encoders for Avro/Protobuf
serialization of gRPC requests and responses.

Let's look at an example in both Avro and Protobuf.

### Custom Protobuf codec

Mu uses a library called [PBDirect] for Protobuf serialization.

To customise the serialization of fields in Protobuf messages, you need to
provide instances of PBDirect's `PBScalarValueReader` and `PBScalarValueWriter`
type classes.

Here is an example of providing a custom reader and writer for
`java.time.LocalDate` that serializes the date as a `String` in ISO 8601 format.
We create the reader and writer by building on PBDirect's built-in reader and
writer for `String`, using `map` and `contramap` respectively.

```scala mdoc:silent
object ProtobufCustomCodecExample {

  import java.time._
  import java.time.format._

  import pbdirect._

  import cats.syntax.contravariant._
  import cats.syntax.functor._

  implicit val localDateReader: PBScalarValueReader[LocalDate] =
    PBScalarValueReader[String].map(string => LocalDate.parse(string, DateTimeFormatter.ISO_LOCAL_DATE))

  implicit val localDateWriter: PBScalarValueWriter[LocalDate] =
    PBScalarValueWriter[String].contramap[LocalDate](_.format(DateTimeFormatter.ISO_LOCAL_DATE))

  case class HelloRequest(name: String, date: LocalDate)

  case class HelloReply(message: String)

  @service(Protobuf)
  trait Greeter[F[_]] {
    def sayHello(request: HelloRequest): F[HelloReply]
  }
}
```

### Custom Avro codec

Mu uses a library called [avro4s] For Avro serialization.

To customise the serialization of fields in Avro records, you need to provide
instances of three avro4s type classes: `SchemaFor`, `Encoder`, and `Decoder`.

Let's look at the same example as above, this time for Avro.

```scala mdoc:silent
object AvroCustomCodecExample {

  import java.time._
  import java.time.format._

  import com.sksamuel.avro4s._
  import org.apache.avro.Schema

  implicit object LocalDateSchemaFor extends SchemaFor[LocalDate] {
    override def schema(fm: com.sksamuel.avro4s.FieldMapper): Schema =
      Schema.create(Schema.Type.STRING)
  }

  implicit object LocalDateEncoder extends Encoder[LocalDate] {
    override def encode(value: LocalDate, schema: Schema, fm: FieldMapper): String =
      value.format(DateTimeFormatter.ISO_LOCAL_DATE)
  }

  implicit object LocalDateDecoder extends Decoder[LocalDate] {
    override def decode(value: Any, schema: Schema, fm: FieldMapper): LocalDate =
      LocalDate.parse(value.toString(), DateTimeFormatter.ISO_LOCAL_DATE)
  }

  case class HelloRequest(name: String, date: LocalDate)

  case class HelloReply(message: String)

  @service(Avro)
  trait Greeter[F[_]] {
    def sayHello(request: HelloRequest): F[HelloReply]
  }
}
```

## Protobuf codecs

Mu provides Protobuf codecs for:

* `BigDecimal`
* `java.time.LocalDate`, `java.time.LocalDateTime` and `java.time.Instant`
* `org.joda.time.LocalDate` and `org.joda.time.LocalDateTime`

Add the following imports to your service code:

| Types | Import |
|---|--|
| `BigDecimal` | `import higherkindness.mu.rpc.internal.encoders.pbd.bigDecimal._` |
| `java.time.{LocalDate, LocalDateTime, Instant}` | `import higherkindness.mu.rpc.internal.encoders.pbd.javatime._` |
| `org.joda.time.{LocalDate, LocalDateTime}` | `import higherkindness.mu.rpc.marshallers.jodaTimeEncoders.pbd._` |

Notes:

* For the jodatime encoders, you will need to add a dependency on the
  `mu-rpc-marshallers-jodatime` module.

## Avro codecs

Mu provides Avro codecs for:

* `BigDecimal`
* `BigDecimal` tagged with 'precision' and 'scale' (e.g. `BigDecimal @@ (Nat._8, Nat._2)`
* `java.time.LocalDate`, `java.time.LocalDateTime` and `java.time.Instant`
* `org.joda.time.LocalDate` and `org.joda.time.LocalDateTime`

Add the following imports to your service code:

| Types | Import |
|---|--|
| `BigDecimal` | `import higherkindness.mu.rpc.internal.encoders.avro.bigDecimal._` |
| Tagged `BigDecimal` | `import higherkindness.mu.rpc.internal.encoders.avro.bigDecimalTagged._` |
| `java.time.*` | `import higherkindness.mu.rpc.internal.encoders.avro.javatime._` |
| `org.joda.time.*` | `import higherkindness.mu.rpc.marshallers.jodaTimeEncoders.avro._` |

Notes:

* The `BigDecimal` codec is not compliant with the Avro spec. We recommend you
  use the tagged `BigDecimal` codec.
* For the jodatime encoders, you will need to add a dependency on the
  `mu-rpc-marshallers-jodatime` module.

If you want to send one of these types directly as an Avro-encoded request or
response (instead of as a field within a request or response), you need to
provide an instance of `io.grpc.MethodDescriptor.Marshaller`.

Mu provides marshallers for these types under separate imports:

| Types | Import |
|---|--|
| `BigDecimal` | `import higherkindness.mu.rpc.internal.encoders.avro.bigDecimal.marshallers._` |
| Tagged `BigDecimal` | `import higherkindness.mu.rpc.internal.encoders.avro.bigDecimalTagged.marshallers._` |
| `java.time.*` | `import higherkindness.mu.rpc.internal.encoders.avro.javatime.marshallers._` |
| `org.joda.time.*` | `import higherkindness.mu.rpc.marshallers.jodaTimeEncoders.avro.marshallers._` |

[Avro]: https://avro.apache.org/
[avro4s]: https://github.com/sksamuel/avro4s
[gRPC]: https://grpc.io/
[Mu]: https://github.com/higherkindness/mu
[PBDirect]: https://github.com/47deg/pbdirect
[Protobuf]: https://developers.google.com/protocol-buffers
