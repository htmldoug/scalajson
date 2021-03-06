# ScalaJSON

[![Join the chat at https://gitter.im/mdedetrich/scalajson](https://badges.gitter.im/mdedetrich/scalajson.svg)](https://gitter.im/mdedetrich/scalajson?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![codecov.io](http://codecov.io/github/mdedetrich/scalajson/coverage.svg?branch=master)](http://codecov.io/github/mdedetrich/scalajson?branch=master)
[![Build Status](https://travis-ci.org/mdedetrich/scalajson.svg?branch=master)](https://travis-ci.org/mdedetrich/scalajson)
[![Scala.js](https://www.scala-js.org/assets/badges/scalajs-0.6.17.svg)](https://www.scala-js.org)

ScalaJSON library, currently provides two minimal implementations of [JSON](https://en.wikipedia.org/wiki/JSON) AST, one that is designed for
typical use and another that is designed for performance/corner cases.

## Usage

Built for Scala 2.10.x, 2.11.x and 2.12.x

```sbt
"org.scala-lang.platform" %% "scalajson" % "1.0.0-M2"
```

If you are using Scala.js, you need to do

```sbt
"org.scala-lang.platform" %%% "scalajson" % "1.0.0-M2"
```

## Standard AST
Implementation is in `scalajson.ast.JValue`

### Goals
- Fully immutable (all underlying collections/types used are immutable)
- `constant`/`effective constant` lookup time for `scalajson.ast.JArray`/`scalajson.ast.JObject`
- Adherence to the typical use for the [JSON](https://en.wikipedia.org/wiki/JSON) standard.
    - Number representation for `scalajson.ast.JNumber` is a `String` which checks if its a valid JSON representation
      of a number (http://stackoverflow.com/a/13502497/1519631)
      - Equals will properly detect if two numbers are equal, i.e. `scalajson.ast.JNumber("34.00") == scalajson.ast.JNumber("34")`
      - Hashcode has been designed to provide consistent hash for numbers of unlimited precision.
    - `scalajson.ast.JObject` is an actual `Map[String,JValue]`. This means that it doesn't handle duplicate keys for a `scalajson.ast.JObject`,
    nor does it handle key ordering.
    - `scalajson.ast.JArray` is an `Vector`.
- Library does not allow invalid JSON in the representation and hence we can guarantee that a `scalajson.ast.JValue` will 
always contain a valid structure that can be serialized/rendered into [JSON](https://en.wikipedia.org/wiki/JSON). 
  - Note that you can lose precision when using `scalajson.ast.JNumber` in `Scala.js` (see `Scala.js` 
section for more info).
- Due to the above, has properly implemented deep equality for all types of `scalajson.ast.JValue`

## Unsafe AST
Implementation is in `scalajson.unsafe.JValue`

### Goals
- Uses the best performing datastructure's for high performance/low memory usage in construction of a `unsafe.JValue`
    - `scalajson.ast.unsafe.JArray` stored as an `Array` for JVM and `js.Array` for Scala.js
    - `scalajson.ast.unsafe.JObject` stored as an `Array` for JVM and `js.Array` for Scala.js
    - `scalajson.ast.unsafe.JNumber` stored as a `String`
- Doesn't use `Scala`'s `stdlib` collection's library
- Defer all runtime errors. We don't throw errors if you happen to provide Infinity/NaN or other invalid data.
- Allow duplicate and ordered keys for a `scalajson.ast.unsafe.JObject`.
- Allow any length/precision of numbers for `scalajson.ast.unsafe.JNumber` since its represented as a `String`
  - Equals/hashcode only checks for `String` equality, not number equality.
    - Means that `scalajson.ast.unsafe.JNumber("34.00")` is not equal to `scalajson.ast.unsafe.JNumber("34")`
- This means that `scalajson.ast.unsafe.JValue` can represent everything that can
can be considered valid under the official [JSON spec](https://www.ietf.org/rfc/rfc4627.txt), even if its not considered sane (i.e.
duplicate keys for a `scalajson.ast.unsafe.JObject`).
  - Also means it can hold invalid data, due to not doing runtime checks
- Is referentially transparent in regards to `String` -> `scalajson.ast.unsafe.JValue` -> `String` since `scalajson.ast.unsafe.JObject` 
  preserves ordering/duplicate keys
- Implements structural equality for both `hashCode` and `equals`. If you need reference equality
  you can use `eq` and if you need reference `hashCode` you can use `.value.hashCode`. Also note that for
  deep comparison is used both `hashCode` and `equals`.

## Conversion between scalajson.JValue and scalajson.ast.unsafe.JValue

Any `scalajson.ast.JValue` implements a conversion to `scalajson.ast.unsafe.JValue` with a `toUnsafe` method and vice versa with a
`toStandard` method. These conversion methods have been written to be as fast as possible.

There are some peculiarities when converting between the two AST's. When converting a `scalajson.ast.unsafe.JNumber` to a 
`scalajson.ast.JNumber`, it is possible for this to fail at runtime (since the internal representation of 
`scalajson.ast.unsafe.JNumber` is a `String` and it doesn't have a runtime check). It is up to the caller on how to handle this error (and when), 
a runtime check is deliberately avoided on our end for performance reasons.

Converting from a `scalajson.ast.JObject` to a `scalajson.ast.unsafe.JObject` will produce 
an `scalajson.ast.unsafe.JObject` with an undefined ordering for its internal `Array`/`js.Array` representation.
This is because a `Map` has no predefined ordering. If you wish to provide ordering, you will either need
to write your own custom conversion to handle this case. Duplicate keys will also be removed for the same reason
in an undefined manner.

Do note that according to the JSON spec, whether to order keys for a `JObject` is not specified. Also note that `Map` 
disregards ordering for equality, however `Array`/`js.Array` equality takes ordering into account.

## Scala.js
ScalaJSON also provides support for [Scala.js](https://github.com/scala-js/scala-js).
The usage of Scala.js mirrors the usage of Scala on the JVM however Scala.js also implements
a `.toJsAny` method which allows you to convert any
`scalajson.ast.JValue`/`scalajson.ast.unsafe.JValue` to a Javascript value in `Scala.js`.

Note that, since a `scalajson.ast.JNumber`/`scalajson.ast.unsafe.JNumber` is unlimited
precision (represented internally as a `String`), calls to `.toJsAny` can lose precision on the
underlying number (numbers in Javascript are represented as double precision floating point number).
You can use the `.value` method on a `scalajson.ast.JNumber`/`scalajson.ast.unsafe.JNumber` to
get the raw string value as a solution to this problem.

## jNumberRegex
`scalajson.JNumber` uses `jNumberRegex` to validate whether a number is a valid
JSON number. One can use `jNumberRegex` explicitly if you want to use the validation that
is used by `scalajson.JNumber` (for example, you may want to validate proper numbers
before creating a `scalajson.unsafe.JNumber`).

```scala
import scalajson.jNumberRegex

"3535353".matches(jNumberRegex) // true
```

Code of Conduct
===============
ScalaJSON uses the [Scala Code of Conduct](https://www.scala-lang.org/conduct.html)
for all communication and discussion. This includes both GitHub, Gitter chat and
other more direct lines of communication such as email.

## Code formatting

The project is formatted using [scalafmt](https://github.com/olafurpg/scalafmt). Please run `scalafmt`
in SBT before committing any changes
