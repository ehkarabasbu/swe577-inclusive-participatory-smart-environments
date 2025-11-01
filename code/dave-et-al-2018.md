# Code Dive: Dave et al. (2018) - How BIM and IoT Actually Talk

**Paper:** A Framework for Integrating BIM and IoT through Open Standards <br>
**Authors:** Dave, B., Kubler, S., Främling, K., & Koskela, L. <br>
**Official Repositories:**
- [Frontend](https://github.com/AaltoAsia/Otaniemi3D)
- [Backend](https://github.com/AaltoAsia/O-MI)
<br>

**Internal Wiki Analysis:** [Paper-Analysis-Dave-2018](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#dave-et-al-2018---a-framework-for-integrating-bim-and-iot-through-open-standards)

---

## 1. The Central Switchboard
A question like "how does the O-MI node not crash when a bunch of requests come in?" might be came in mind. The answer of it can be found here. It's about this `RequestHandler` actor. We can think of it as the system's brain. It's where every single request first lands. Its only job is to figure out what the request wants—a read, a write, whatever—and pass it off to the right specialist.

### `RequestHandler.scala`

This `receive` block is the core of it. Just a clean, simple pattern match. For us, this whole actor model is a significant find. It’s a ready-made recipe for a backend that can scale up and handle lots of people interacting with it at once without falling over.

```scala
// Found in: 08_cloned_repos/week3/dave_etal_2018/O-MI/O-MI-Node/src/main/scala/responses/RequestHandler.scala

def receive: PartialFunction[Any, Unit] = {
  case read: ReadRequest => respond(handleReadRequest(read))
  case write: WriteRequest => respond(handleWriteRequest(write))
  case delete: DeleteRequest => respond(handleDeleteRequest(delete))
  case call: CallRequest => respond(handleCallRequest(call))
  // ...and so on
  case na: NewAgent => addAgent(na)
  case na: AgentStopped => agentStopped(na.agentName)
}
```

What's really good is the `splitAndHandle` function that gets called by these handlers. It's the bit that decides if a request should be handled by the main database or by a custom "agent." This is exactly how we can give parts of a digital twin—like a single room in a BIM model—their own special behaviors.

---

## 2. Giving Building Parts a Mind of Their Own

"Internal Agent" pattern? It's probably the most important thing we'll take from this codebase. An `InternalAgent` is basically a little software robot that takes control of one piece of the digital twin data tree. Why does that matter? Because it's how we can make the system "participatory." It's how users get to actually *interact* with bits of the building.

### `InternalAgent.scala`

The `ScalaInternalAgent` trait is the starter kit for making these agents. It gives them the tools to talk to the database and everything else. When a user clicks a button in the UI and sends a `WriteRequest`, this is where it ends up.

```scala
// Found in: 08_cloned_repos/week3/dave_etal_2018/O-MI/O-MI-Node/src/main/scala/agentSystem/InternalAgent.scala

case class ResponsibleWrite(promise: Promise[ResponseRequest], write: WriteRequest)

trait ScalaInternalAgent extends InternalAgent with ActorLogging {

  import context.dispatcher

  protected def requestHandler: ActorRef
  protected def dbHandler: ActorRef

  // Helper functions to talk to the database
  final def writeToDB(write: WriteRequest): Future[ResponseRequest] = requestFromDB(write)
  final def readFromDB(read: ReadRequest): Future[ResponseRequest] = requestFromDB(read)

  final def requestFromDB(request: OdfRequest): Future[ResponseRequest] = {
    // ...
    (dbHandler ? requestWithSenderInfo).mapTo[ResponseRequest]
  }
  // ...
}
```

This pattern is the missing link for our SWE + Architecture goal. We can literally make an agent for a window from a BIM file and program it to do stuff—like "open when CO2 gets too high." It's brilliant. It completely separates the main server logic from the logic of individual devices.

---

## 3. A Common Language for Everything

The whole reason this system works without creating a chaos-like situation is because it forces everything to use the standardized O-MI and O-DF data structures. In Scala, these are just clean, immutable case classes. No guesswork, no side effects.

### `OmiTypes.scala`

Examine `ReadRequest` and `WriteRequest` here. They're useful little data packets. They've got the data payload (`odf`), a callback address, a time-to-live... everything someone need.

```scala
// Found in: 08_cloned_repos/week3/dave_etal_2018/O-MI/O-MI-Node/src/main/scala/types/Omi/OmiTypes.scala

case class ReadRequest(
  odf: ODF,
  begin: Option[Timestamp] = None,
  end: Option[Timestamp] = None,
  newest: Option[Int] = None,
  oldest: Option[Int] = None,
  // ...
  ttl: Duration = 10.seconds,
  // ...
) extends OmiRequest with OdfRequest

case class WriteRequest(
  odf: ODF,
  callback: Option[Callback] = None,
  ttl: Duration = 10.seconds,
  // ...
) extends OmiRequest with OdfRequest with PermissiveRequest
```

This is the technical solution to the "proprietary silo" problem Wilson's paper warned us about. By making every part of the system speak the same language, we're guaranteeing that our framework can grow and change without getting stuck with one company's tech. It's the foundation for the open, connected system we want to build.