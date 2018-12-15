
# Akka Messenger
Akka Messenger is an API which provides asynchronous query, command and event messaging behavior for micro services.

The messenger runs in an Akka Cluster and is implemented using Distributed Pub Sub which provides two key things.
First, the cluster is a central component for client services to connect to for dispatching messages.
Second, clusters with multiple nodes are fault tolerant. 
If a cluster node crashes or just becomes unavailable then messaging will continue to work through the other messenger cluster nodes. 

A messenger client uses a Connector to connect to the messenger cluster and perform messaging.
A Connector can message another service by name to ask a query or tell a command.
A query and a command will yield an event as a result which is asynchronously returned to that Connector through a Future.

Commands are mutating.
In the case of a command an event is produced as the result of the command and returned.
The event produced by the command is also used to notify of an event.
Any Connectors subscribed to that service's events will receive the event as the result of the command.

Queries are non-mutating.
In the case of a query an event is reproduced as the result from a previous command and returned.
The event is not used to notify other Connectors. 

Lastly, a Connector can directly notify of an event. All other Connectors subscribed to that service by name will receive the event.

In production a service will be deployed in multiple nodes such as an AWS Elastic Beanstalk Application, AWS Elastic Container Service Cluster or Kubernetes ReplicaSet.
Each of the nodes of the service should contain one connector but share a service name string. 

For example a user service composed of five cluster nodes each with a Connector configured with the name "user-svc".
A query or command made by a node in the service will have the resulting Event returned to that same node and handled in a Future.
ie it will not be returned to a different node in the service.

Subscriptions are of service names. 
An event generated by any node in a service will be received by all nodes within subscribing services.

## Queries
To run the query multi-jvm integration test use the sbt command:
```
sbt "multi-jvm:run queryTest.QueryTest"
```
Ask a Query with a Connector and use a Partial Function within a Future to map the Event.
```
connector.askQuery(toServiceName = "bar-svc", FooQuery(message).map {
  case BarEvent(message) =>
    println(message)
  case _ =>
    println("foo-svc - Unknown event")
}.recover {
  case e: Exception =>
    println(e.toString)
}
```
Answer the Query with a Connector and use a handler Function to yield a Future Event.
```
connector.installQueryHandlerFunction {
  case FooQuery(message) =>
    Future {
      println(s"bar-svc - Foo Query: $message")
      BarEvent( message)
    }
  case _ =>
    Future {
      println("bar-svc - Unknown query")
      throw UnknownQuery()
  }
}
```
## Commands
To run the command multi-jvm integration test use the sbt command:
```
sbt "multi-jvm:run commandTest.CommandTest"
```
Tell a Command with a Connector and use a Partial Function within a Future to map the Event.
```
connector.tellCommand(toServiceName = "bar-svc", FooCommand(message).map {
  case BarEvent(message) =>
    println(s"foo-svc - Bar event: $message")
  case _ =>
    println(s"foo-svc - Unknown event")
  }.recover {
  case e: Exception =>
    println(e.toString)
  }
```
Answer the Command with a Connector and use a handler Function to yield a Future Event.
```
connector.installCommandHandlerFunction {
  case FooCommand(message) =>
    Future {
      println(s"bar-svc - Foo Command: $message")
      BarEvent(message)
    }
  case _ =>
    Future {
      println("bar-svc - Unknown command")
      throw UnknownCommand()
  }
}
```
Subscribe to a service with a Connector and receive the Event yielded from the Command.
```
connector.installEventHandlerFunction {
  case BarEvent(message) =>
    println(s"another-svc - Notified of event: $message")
  case _ =>
    println(s"another-svc - Unknown event")
}

connector.subscribeToServiceEvents("bar-svc")
```
## Events
To run the event multi-jvm integration test use the sbt command:
```
sbt "multi-jvm:run eventTest.EventTest"
```
Notify Subscribers of a generated Event with a Connector.
```
for (i <- 0 to 1000) {
  connector.notifyEvent(FooEvent(counter = i))
}
```
Subscribe to a service with a Connector and receive the Event from notifying.
```
connector.installEventHandlerFunction {
  case FooEvent(counter) =>
    println(s"bar-svc - Foo Event: $counter")
  case _ =>
    println(s"bar-svc - Unknown event")
}

connector.subscribeToServiceEvents("foo-svc")
```