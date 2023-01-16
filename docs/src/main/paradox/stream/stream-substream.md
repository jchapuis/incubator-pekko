# Substreams

## Dependency

To use Pekko Streams, add the module to your project:

@@dependency[sbt,Maven,Gradle] {
  bomGroup=org.apache.pekko bomArtifact=pekko-bom_$scala.binary.version$ bomVersionSymbols=PekkoVersion
  symbol1=PekkoVersion
  value1="$pekko.version$"
  group="org.apache.pekko"
  artifact="pekko-stream_$scala.binary.version$"
  version=PekkoVersion
}

## Introduction

Substreams are represented as @java[@javadoc[SubSource](pekko.stream.javadsl.SubSource) or] @apidoc[stream.*.SubFlow] instances, on which you can multiplex a single @java[@apidoc[stream.*.Source] or] @apidoc[stream.*.Flow]
into a stream of streams.

SubFlows cannot contribute to the super-flow’s materialized value since they are materialized later,
during the runtime of the stream processing.

operators that create substreams are listed on @ref[Nesting and flattening operators](operators/index.md#nesting-and-flattening-operators)

## Nesting operators

### groupBy

A typical operation that generates substreams is @apidoc[groupBy](stream.*.Source) {scala="#groupBy[K](maxSubstreams:Int,f:Out=%3EK,allowClosedSubstreamRecreation:Boolean):org.apache.pekko.stream.scaladsl.SubFlow[Out,Mat,FlowOps.this.Repr,FlowOps.this.Closed]" java="#groupBy(int,org.apache.pekko.japi.function.Function,boolean)"}.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #groupBy1 }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #groupBy1 }

![stream-substream-groupBy1.png](../../images/stream-substream-groupBy1.png)

This operation splits the incoming stream into separate output
streams, one for each element key. The key is computed for each element
using the given function, which is `f` in the above diagram. When a new key is encountered for the first time
a new substream is opened and subsequently fed with all elements belonging to that key.
If `allowClosedSubstreamRecreation` is set to `true` a substream belonging to a specific key
will be recreated if it was closed before, otherwise elements belonging to that key will be dropped.

If you add a @apidoc[stream.*.Sink] or @apidoc[stream.*.Flow] right after the `groupBy` operator,
all transformations are applied to all encountered substreams in the same fashion.
So, if you add the following `Sink`, that is added to each of the substreams as in the below diagram.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #groupBy2 }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #groupBy2 }

![stream-substream-groupBy2.png](../../images/stream-substream-groupBy2.png)

Also substreams, more precisely, @apidoc[stream.*.SubFlow] @java[and @javadoc[SubSource](pekko.stream.javadsl.SubSource)] have methods that allow you to
merge or concat substreams into the main stream again.

The @apidoc[mergeSubstreams](stream.*.SubFlow) {scala="#mergeSubstreams:F[Out]" java="#mergeSubstreams()"} method merges an unbounded number of substreams back to the main stream.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #groupBy3 }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #groupBy3 }

![stream-substream-groupBy3.png](../../images/stream-substream-groupBy3.png)

You can limit the number of active substreams running and being merged at a time,
with either the @apidoc[mergeSubstreamsWithParallelism](stream.*.SubFlow) {scala="#mergeSubstreamsWithParallelism(parallelism:Int):F[Out]" java="#mergeSubstreamsWithParallelism(int)"} or @apidoc[concatSubstreams](stream.*.SubFlow) {scala="#concatSubstreams:F[Out]" java="#concatSubstreams()"} method.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #groupBy4 }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #groupBy4 }

However, since the number of running (i.e. not yet completed) substreams is capped,
be careful so that these methods do not cause deadlocks with back pressure like in the below diagram.

Element one and two leads to two created substreams, but since the number of substreams are capped to 2 
when element 3 comes in it cannot lead to creation of a new substream until one of the previous two are completed 
and this leads to the stream being deadlocked.

![stream-substream-groupBy4.png](../../images/stream-substream-groupBy4.png)

### splitWhen and splitAfter

@apidoc[splitWhen](stream.*.Source) {scala="#splitWhen(p:Out=%3EBoolean):org.apache.pekko.stream.scaladsl.SubFlow[Out,Mat,FlowOps.this.Repr,FlowOps.this.Closed]" java="#splitWhen(org.apache.pekko.japi.function.Predicate)"} and @apidoc[splitAfter](stream.*.Source) {scala="#splitAfter(p:Out=%3EBoolean):org.apache.pekko.stream.scaladsl.SubFlow[Out,Mat,FlowOps.this.Repr,FlowOps.this.Closed]" java="#splitAfter(org.apache.pekko.japi.function.Predicate)"} are two other operations which generate substreams.

The difference from @apidoc[groupBy](stream.*.Source) {scala="#groupBy[K](maxSubstreams:Int,f:Out=%3EK,allowClosedSubstreamRecreation:Boolean):org.apache.pekko.stream.scaladsl.SubFlow[Out,Mat,FlowOps.this.Repr,FlowOps.this.Closed]" java="#groupBy(int,org.apache.pekko.japi.function.Function,boolean)"} is that, if the predicate for `splitWhen` and `splitAfter` returns true,
a new substream is generated, and the succeeding elements after split will flow into the new substream.

`splitWhen` flows the element on which the predicate returned true to a new substream,
 whereas `splitAfter` flows the next element to the new substream after the element on which predicate returned true.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #splitWhenAfter }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #splitWhenAfter }

These are useful when you scanned over something and you don't need to care about anything behind it.
A typical example is counting the number of characters for each line like below.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #wordCount }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #wordCount }

This prints out the following output.

```
23
16
26
``` 

![stream-substream-splitWhen-splitAfter.png](../../images/stream-substream-splitWhen-splitAfter.png)

## Flattening operators

### flatMapConcat

@apidoc[flatMapConcat](stream.*.Source) {scala="#flatMapConcat[T,M](f:Out=%3Eorg.apache.pekko.stream.Graph[org.apache.pekko.stream.SourceShape[T],M]):FlowOps.this.Repr[T]" java="#flatMapConcat(org.apache.pekko.japi.function.Function)"} and @apidoc[flatMapMerge](stream.*.Source) {scala="#flatMapMerge[T,M](breadth:Int,f:Out=%3Eorg.apache.pekko.stream.Graph[org.apache.pekko.stream.SourceShape[T],M]):FlowOps.this.Repr[T]" java="#flatMapMerge(int,org.apache.pekko.japi.function.Function)"} are substream operations different from @apidoc[groupBy](stream.*.Source) {scala="#groupBy[K](maxSubstreams:Int,f:Out=%3EK,allowClosedSubstreamRecreation:Boolean):org.apache.pekko.stream.scaladsl.SubFlow[Out,Mat,FlowOps.this.Repr,FlowOps.this.Closed]" java="#groupBy(int,org.apache.pekko.japi.function.Function,boolean)"} and `splitWhen/After`.

`flatMapConcat` takes a function, which is `f` in the following diagram.
The function `f` of `flatMapConcat` transforms each input element into a @apidoc[stream.*.Source] that is then flattened
into the output stream by concatenation.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #flatMapConcat }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #flatMapConcat }

![stream-substream-flatMapConcat1.png](../../images/stream-substream-flatMapConcat1.png)

Like the `concat` operation on @apidoc[stream.*.Flow], it fully consumes one @apidoc[stream.*.Source] after the other.
So, there is only one substream actively running at a given time.

Then once the active substream is fully consumed, the next substream can start running.
Elements from all the substreams are concatenated to the sink.

![stream-substream-flatMapConcat2.png](../../images/stream-substream-flatMapConcat2.png)

### flatMapMerge

`flatMapMerge` is similar to `flatMapConcat`, but it doesn't wait for one `Source` to be fully consumed.
 Instead, up to `breadth` number of streams emit elements at any given time.

Scala
:   @@snip [SubstreamDocSpec.scala](/docs/src/test/scala/docs/stream/SubstreamDocSpec.scala) { #flatMapMerge }

Java
:   @@snip [SubstreamDocTest.java](/docs/src/test/java/jdocs/stream/SubstreamDocTest.java) { #flatMapMerge }

![stream-substream-flatMapMerge.png](../../images/stream-substream-flatMapMerge.png)
