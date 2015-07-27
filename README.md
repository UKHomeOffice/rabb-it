Scala publish/subscribe JSON API for RabbitMQ
=============================================
Scala general functionality to interface with RabbitMQ via JSON protocol.

Project built with the following (main) technologies:

- Scala

- SBT

- Akka

- RabbitMQ

- Specs2

Introduction
------------
TODO

Include: installing RabbitMQ

Build and Deploy
----------------
The project is built with SBT (using Activator on top).

To compile:
> activator compile

To run the specs:
> activator test

To run integration specs:
> activator it:test

The project can be "assembled" into a "one JAR":
> activator assembly

Note that "assembly" will first compile and test.

Publishing
----------
To publish the jar to artifactory you will need to 

1. Copy the .credentials file into your <home directory>/.ivy2/
2. Edit this .credentials file to fill in the artifactory user and password

> activator publish

Note that initially this project refers to some libraries held within a private Artifactory. However, those libraries have been open sourced under https://github.com/UKHomeOffice.

Example Usage
-------------
```scala
  object ExampleBoot extends App with HasConfig {
    implicit val json4sFormats = DefaultFormats
  
    val system = ActorSystem("example-actor-system", config)
  
    // Consume
    system.actorOf(Props(new ConsumerActor with Consumer[String] with ExampleQueue with Rabbit {
      def consume(json: JValue) = Future {
        val message = (json \ "message").extract[String]
        println(s"Congratulations, consumed message '$message'")
        Good(message)
      }
    }))
  
    // Publish
    val publisher = new Publisher with ExampleQueue with Rabbit
    publisher.publish("message" -> "hello world!")
  }
  
  trait ExampleQueue extends Queue {
    def queueName = "rabbit-example"
  }
```

Noting that a "configuration" such as application.conf must be provided e.g.
```scala
  amqp {
    addresses = [{
      host = "127.0.0.1"
      port = 5672
    }]
  
    automatic-recovery = on
  }
```

Writing specs (tests) against Rabbit is very easy (integration tests are so easy, they can be regarded as unit tests). Upon running the specs, the SBT build will attempt to start Rabbit (but it is easier to start Rabbit yourself and keep it running, as all specs will create unique, temporary queues, which are removed when examples have finished, and all connections are automatically closed, closing all Rabbit channels).

A spec that consumes valid and error messages, upon publication of said messages. All the plumbing is handled automatically, allowing you to concentrate on writing specs to build your API and subsequently your code.

```scala
class WithConsumerSpec(implicit ev: ExecutionEnv) extends Specification with RabbitSpec {
  "Consumer" should {
    "consume valid message" in {
      val validMessageConsumed = Promise[Boolean]()

      val publisher = new Publisher with WithQueue.Consumer with WithRabbit {
        def json(json: JValue) = validMessageConsumed success true
      }

      publisher.publish(JObject())

      validMessageConsumed.future must beTrue.awaitFor(10 seconds)
    }

    "consume error message" in {
      val errorMessageConsumed = Promise[Boolean]()

      val publisher = new Publisher with WithQueue.ErrorConsumer with WithRabbit {
        def jsonError(jsonError: JsonError) = errorMessageConsumed success true
      }

      publisher.publish(JsonError(JObject(), "Error"))

      errorMessageConsumed.future must beTrue.awaitFor(10 seconds)
    }
  }
}
```