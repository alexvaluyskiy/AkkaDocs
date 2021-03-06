---
layout: docs.hbs
title: Cluster Client
---
# Cluster Client

An actor system that is not part of the cluster can communicate with actors somewhere in the cluster via this `ClusterClient`. The client can of course be part of another cluster. It only needs to know the location of one (or more) nodes to use as initial contact points. It will establish a connection to a ClusterReceptionist somewhere in the cluster. It will monitor the connection to the receptionist and establish a new connection if the link goes down. When looking for a new receptionist it uses fresh contact points retrieved from previous establishment, or periodically refreshed contacts, i.e. not necessarily the initial contact points.

> **Note** <br>
`ClusterClient` should not be used when sending messages to actors that run within the same cluster. Similar functionality as the `ClusterClient` is provided in a more efficient way by Distributed Publish Subscribe in Cluster for actors that belong to the same cluster.

Also, note it's necessary to change akka.actor.provider from `akka.actor.LocalActorRefProvider` to `akka.remote.RemoteActorRefProvider` or `akka.cluster.ClusterActorRefProvider` when using the cluster client.

The receptionist is supposed to be started on all nodes, or all nodes with specified role, in the cluster. The receptionist can be started with the `ClusterClientReceptionist` extension or as an ordinary actor.

You can send messages via the `ClusterClient` to any actor in the cluster that is registered in the `DistributedPubSubMediator` used by the `ClusterReceptionist`. The `ClusterClientReceptionist` provides methods for registration of actors that should be reachable from the client. Messages are wrapped in `ClusterClient.Send`, `ClusterClient.SendToAll` or `ClusterClient.Publish`.

Both the `ClusterClient` and the `ClusterClientReceptionist` emit events that can be subscribed to. The `ClusterClient` sends out notifications in relation to having received a list of contact points from the `ClusterClientReceptionist`. One use of this list might be for the client to record its contact points. A client that is restarted could then use this information to supersede any previously configured contact points.

The `ClusterClientReceptionist` sends out notifications in relation to having received contact from a `ClusterClient`. This notification enables the server containing the receptionist to become aware of what clients are connected.

**1. ClusterClient.Send**

The message will be delivered to one recipient with a matching path, if any such exists. If several entries match the path the message will be delivered to one random destination. The `Sender` of the message can specify that local affinity is preferred, i.e. the message is sent to an actor in the same local actor system as the used receptionist actor, if any such exists, otherwise random to any other matching entry.

**2. ClusterClient.SendToAll**

The message will be delivered to all recipients with a matching path.

**3. ClusterClient.Publish**

The message will be delivered to all recipients Actors that have been registered as subscribers to the named topic.

Response messages from the destination actor are tunneled via the receptionist to avoid inbound connections from other cluster nodes to the client, i.e. the `Sender`, as seen by the destination actor, is not the client itself. The `Sender` of the response messages, as seen by the client, is deadLetters since the client should normally send subsequent messages via the `ClusterClient`. It is possible to pass the original sender inside the reply messages if the client is supposed to communicate directly to the actor in the cluster.

While establishing a connection to a receptionist the `ClusterClient` will buffer messages and send them when the connection is established. If the buffer is full the `ClusterClient` will drop old messages when new messages are sent via the client. The size of the buffer is configurable and it can be disabled by using a buffer size of 0.

It's worth noting that messages can always be lost because of the distributed nature of these actors. As always, additional logic should be implemented in the destination (acknowledgement) and in the client (retry) actors to ensure at-least-once message delivery.

## An Example
On the cluster nodes first start the receptionist. Note, it is recommended to load the extension when the actor system is started by defining it in the akka.extensions configuration property:

```hocon
akka.extensions = ["akka.cluster.client.ClusterClientReceptionist"]
```

Next, register the actors that should be available for the client.

```csharp
runOn(host1) {
  val serviceA = system.actorOf(Props[Service], "serviceA")
  ClusterClientReceptionist(system).registerService(serviceA)
}
 
runOn(host2, host3) {
  val serviceB = system.actorOf(Props[Service], "serviceB")
  ClusterClientReceptionist(system).registerService(serviceB)
}
```
On the client you create the `ClusterClient` actor and use it as a gateway for sending messages to the actors identified by their path (without address information) somewhere in the cluster.

```csharp
runOn(client) {
  val c = system.actorOf(ClusterClient.props(
    ClusterClientSettings(system).withInitialContacts(initialContacts)), "client")
  c ! ClusterClient.Send("/user/serviceA", "hello", localAffinity = true)
  c ! ClusterClient.SendToAll("/user/serviceB", "hi")
}
The initialContacts parameter is a Set[ActorPath], which can be created like this:

val initialContacts = Set(
  ActorPath.fromString("akka.tcp://OtherSys@host1:2552/system/receptionist"),
  ActorPath.fromString("akka.tcp://OtherSys@host2:2552/system/receptionist"))
val settings = ClusterClientSettings(system)
  .withInitialContacts(initialContacts)
```

You will probably define the address information of the initial contact points in configuration or system property. See also Configuration.

## ClusterClientReceptionist Extension
In the example above the receptionist is started and accessed with the `akka.cluster.client.ClusterClientReceptionist` extension. That is convenient and perfectly fine in most cases, but it can be good to know that it is possible to start the `akka.cluster.client.ClusterReceptionist` actor as an ordinary actor and you can have several different receptionists at the same time, serving different types of clients.

Note that the `ClusterClientReceptionist` uses the `DistributedPubSub` extension, which is described in Distributed Publish Subscribe in Cluster.

It is recommended to load the extension when the actor system is started by defining it in the akka.extensions configuration property:

```hocon
akka.extensions = ["akka.cluster.client.ClusterClientReceptionist"]
```

## Events
As mentioned earlier, both the ClusterClient and ClusterClientReceptionist emit events that can be subscribed to. The following code snippet declares an actor that will receive notifications on contact points (addresses to the available receptionists), as they become available. The code illustrates subscribing to the events and receiving the ClusterClient initial state.

```csharp
class ClientListener(targetClient: ActorRef) extends Actor {
  override def preStart(): Unit =
    targetClient ! SubscribeContactPoints
 
  def receive: Receive =
    receiveWithContactPoints(Set.empty)
 
  def receiveWithContactPoints(contactPoints: Set[ActorPath]): Receive = {
    case ContactPoints(cps) ⇒
      context.become(receiveWithContactPoints(cps))
    // Now do something with the up-to-date "cps"
    case ContactPointAdded(cp) ⇒
      context.become(receiveWithContactPoints(contactPoints + cp))
    // Now do something with an up-to-date "contactPoints + cp"
    case ContactPointRemoved(cp) ⇒
      context.become(receiveWithContactPoints(contactPoints - cp))
    // Now do something with an up-to-date "contactPoints - cp"
  }
}
```

Similarly we can have an actor that behaves in a similar fashion for learning what cluster clients contact a `ClusterClientReceptionist`:

```csharp
class ReceptionistListener(targetReceptionist: ActorRef) extends Actor {
  override def preStart(): Unit =
    targetReceptionist ! SubscribeClusterClients
 
  def receive: Receive =
    receiveWithClusterClients(Set.empty)
 
  def receiveWithClusterClients(clusterClients: Set[ActorRef]): Receive = {
    case ClusterClients(cs) ⇒
      context.become(receiveWithClusterClients(cs))
    // Now do something with the up-to-date "c"
    case ClusterClientUp(c) ⇒
      context.become(receiveWithClusterClients(clusterClients + c))
    // Now do something with an up-to-date "clusterClients + c"
    case ClusterClientUnreachable(c) ⇒
      context.become(receiveWithClusterClients(clusterClients - c))
    // Now do something with an up-to-date "clusterClients - c"
  }
}
```

## Configuration
The `ClusterClientReceptionist` extension (or `ClusterReceptionistSettings`) can be configured with the following properties:

```hocon
# Settings for the ClusterClientReceptionist extension
akka.cluster.client.receptionist {
  # Actor name of the ClusterReceptionist actor, /system/receptionist
  name = receptionist
 
  # Start the receptionist on members tagged with this role.
  # All members are used if undefined or empty.
  role = ""
 
  # The receptionist will send this number of contact points to the client
  number-of-contacts = 3
 
  # The actor that tunnel response messages to the client will be stopped
  # after this time of inactivity.
  response-tunnel-receive-timeout = 30s
  
  # The id of the dispatcher to use for ClusterReceptionist actors. 
  # If not specified default dispatcher is used.
  # If specified you need to define the settings of the actual dispatcher.
  use-dispatcher = ""
 
  # How often failure detection heartbeat messages should be received for
  # each ClusterClient
  heartbeat-interval = 2s
 
  # Number of potentially lost/delayed heartbeats that will be
  # accepted before considering it to be an anomaly.
  # The ClusterReceptionist is using the akka.remote.DeadlineFailureDetector, which
  # will trigger if there are no heartbeats within the duration
  # heartbeat-interval + acceptable-heartbeat-pause, i.e. 15 seconds with
  # the default settings.
  acceptable-heartbeat-pause = 13s
 
  # Failure detection checking interval for checking all ClusterClients
  failure-detection-interval = 2s
}
```

The following configuration properties are read by the `ClusterClientSettings` when created with a ActorSystem parameter. It is also possible to amend the `ClusterClientSettings` or create it from another config section with the same layout as below. `ClusterClientSettings` is a parameter to the `ClusterClient.props` factory method, i.e. each client can be configured with different settings if needed.

```hocon
# Settings for the ClusterClient
akka.cluster.client {
  # Actor paths of the ClusterReceptionist actors on the servers (cluster nodes)
  # that the client will try to contact initially. It is mandatory to specify
  # at least one initial contact. 
  # Comma separated full actor paths defined by a string on the form of
  # "akka.tcp://system@hostname:port/system/receptionist"
  initial-contacts = []
  
  # Interval at which the client retries to establish contact with one of 
  # ClusterReceptionist on the servers (cluster nodes)
  establishing-get-contacts-interval = 3s
  
  # Interval at which the client will ask the ClusterReceptionist for
  # new contact points to be used for next reconnect.
  refresh-contacts-interval = 60s
  
  # How often failure detection heartbeat messages should be sent
  heartbeat-interval = 2s
  
  # Number of potentially lost/delayed heartbeats that will be
  # accepted before considering it to be an anomaly.
  # The ClusterClient is using the akka.remote.DeadlineFailureDetector, which
  # will trigger if there are no heartbeats within the duration 
  # heartbeat-interval + acceptable-heartbeat-pause, i.e. 15 seconds with
  # the default settings.
  acceptable-heartbeat-pause = 13s
  
  # If connection to the receptionist is not established the client will buffer
  # this number of messages and deliver them the connection is established.
  # When the buffer is full old messages will be dropped when new messages are sent
  # via the client. Use 0 to disable buffering, i.e. messages will be dropped
  # immediately if the location of the singleton is unknown.
  # Maximum allowed buffer size is 10000.
  buffer-size = 1000
 
  # If connection to the receiptionist is lost and the client has not been
  # able to acquire a new connection for this long the client will stop itself.
  # This duration makes it possible to watch the cluster client and react on a more permanent
  # loss of connection with the cluster, for example by accessing some kind of
  # service registry for an updated set of initial contacts to start a new cluster client with.
  # If this is not wanted it can be set to "off" to disable the timeout and retry
  # forever.
  reconnect-timeout = off
}
```

## Failure handling
When the cluster client is started it must be provided with a list of initial contacts which are cluster nodes where receptionists are running. It will then repeatedly (with an interval configurable by establishing-get-contacts-interval) try to contact those until it gets in contact with one of them. While running, the list of contacts are continuously updated with data from the receptionists (again, with an interval configurable with refresh-contacts-interval), so that if there are more receptionists in the cluster than the initial contacts provided to the client the client will learn about them.

While the client is running it will detect failures in its connection to the receptionist by heartbeats if more than a configurable amount of heartbeats are missed the client will try to reconnect to its known set of contacts to find a receptionist it can access.

## When the cluster cannot be reached at all
It is possible to make the cluster client stop entirely if it cannot find a receptionist it can talk to within a configurable interval. This is configured with the `reconnect-timeout`, which defaults to off. This can be useful when initial contacts are provided from some kind of service registry, cluster node addresses are entirely dynamic and the entire cluster might shut down or crash, be restarted on new addresses. Since the client will be stopped in that case a monitoring actor can watch it and upon `Terminate` a new set of initial contacts can be fetched and a new cluster client started.