---
layout: post
title: Counter Flow Implementation for Akka Streams Using Materialized Value
date:   2017-01-22 23:14:24 +0300
categories: general akka stream akka-streams scala counter materialized
---

The main purpose of this post is to demonstrate a use-case scenario for `Materialized Value` of `Akka Streams`.
In short, the **materialized value** is the only proper way to communicate with a stage after building the `graph`.
For detail information read the [documentation][1]

For this purpose, I am going to implement a `Counter Flow` which **counts** the number of elements pass through the graph by using `scala` language.

If the terms `Graph`, `Stage`, `Source`, `Flow`, `Sink` does not mean anything to you, first read [akka basics][2].
Otherwise, move on.


Let's start:

- Add `akka streams` library to your `build.sbt`

  ```scala
  libraryDependencies += "com.typesafe.akka" %% "akka-stream" % "2.4.16"
  ```


- Create the `CounterFlow.scala` file in your project and add the following **imports**

  ```scala
  //replace with your package
  package io.fcat.flows

  //do not change this part
  import java.util.concurrent.atomic.AtomicLong

  import akka.stream.scaladsl.Flow
  import akka.stream.stage._
  import akka.stream.{Attributes, FlowShape, Inlet, Outlet}
  ```

- Implement a basic `Counter` trait in `CounterFlow.scala`

  ```scala
  trait Counter {
    def get: Long
    def reset()
  }
  ```

- Implement custom `CounterFlow` stage in `CounterFlow.scala`

  ```scala
  final class CounterFlow[A] private() extends GraphStageWithMaterializedValue[FlowShape[A, A], Counter] {

    val in = Inlet[A]("Map.in")
    val out = Outlet[A]("Map.out")

    override val shape = FlowShape.of(in, out)

    override def createLogicAndMaterializedValue(attr: Attributes): (GraphStageLogic, Counter) = {
      val internalCounter = new AtomicLong(0)

      val logic = new GraphStageLogic(shape) {
        setHandler(in, new InHandler {
          override def onPush(): Unit = {
            internalCounter.incrementAndGet()
            push(out, grab(in))
          }
        })
        setHandler(out, new OutHandler {
          override def onPull(): Unit = {
            pull(in)
          }
        })
      }

      val counter = new Counter {
        override def get: Long = internalCounter.get()
        override def reset() = internalCounter.lazySet(0)
      }

      (logic, counter)
    }
  }
  ```

> **Notes:**
  - Basically, we are creating an `internalCounter` variable inside the stage to keep track the number of elements.
  - This counter will be incremented on every element.
  - This stage only updates its counter, and carries the element to the next stage unchanged.
  - By using `counter's` methods, it is possible to read the current value of `internalCounter` or reset it to 0.
  - The `counter` object will be reachable by the creator of the graph, so that while the graph is in use, it will be possible to interact with this stage.


- Create a **companion object** for the `CounterFlow` class:

  ```scala
  object CounterFlow {
    def apply[T]: Flow[T, T, Counter] = {
      Flow.fromGraph(new CounterFlow[T]())
    }
  }
  ```

> **Note:**
  With this signature, we are indicating that, `apply[T]` method returns a `Flow` which
  - Accepts type `T` as incoming element
  - Emits type `T` to the next stage as outgoing element
  - Returns a `Counter` object as **materialized value**


- Now create a `Main.scala` file and **test** your new `flow`:

  ```scala
  object Main extends App{

    // These are required to materialize a graph
    implicit val system: ActorSystem = ActorSystem("test-system")
    implicit val materializer: ActorMaterializer = ActorMaterializer()

    // Source emitting messages with 1 second interval
    val src: Source[String, Cancellable] = Source.tick(0.second, 1.second, "tick")

    // Counter Flow that accepts String
    val counterFlow: Flow[String, String, Counter] = CounterFlow[String]

    // Sink that accepts String and prints them to the stdout
    val sink: Sink[String, Future[Done]] = Sink.foreach[String](x => println(x))

    // Full graph constructed by linking source, flow and sink together
    // This graph returns a Counter when it is materialized (by run() method)
    val graph: RunnableGraph[Counter] = src
      .viaMat(counterFlow)(Keep.right)
      .toMat(sink)(Keep.left)

    //Materialize the graph by calling run() method
    //counter object can be used to get information about number of elements passed
    // By calling run method, source will start emitting the messages
    // And sink will start writing the messages to the console
    val counter: Counter = graph.run()

    // Now we can ask periodically the count of elements
    for (i <- 0 until 100) {
      Thread.sleep(5000)
      println("counter = " + counter.get)
    }
  }
  ```

- There you go. From now on, you have a `CounterFlow` which can be placed into any graph
and measure the number of elements passed on it.


[1]: http://doc.akka.io/docs/akka/current/scala/stream/stream-composition.html#Materialized_values
[2]: http://doc.akka.io/docs/akka/current/scala/stream/stream-flows-and-basics.html