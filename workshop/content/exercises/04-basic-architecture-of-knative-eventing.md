In *Knative Serving*, there are four kinds of records: *Revision*, *Configuration*, *Route*, and *Service*.
In *Knative Eventing*, there are more kinds of records, subdivided into four major groups: **Messaging**, **Eventing**, **Sources**, and **Flows**. Added to these is a small but growing collection of *duck types*, which you can think of as shared interfaces that show up in multiple categories.

## Messaging
*Messaging* is the business of moving *CloudEvents* from one place to another. The primary record kinds are *Channels* and *Subscriptions*.

*Channels* are used to describe and configure systems like RabbitMQ and Kafka. You provide a *Channel* record to tell Knative about the availability of such systems.
For development convenience, Eventing bundles an “In-Memory Channel” (IMC) implementation. 

*Subscriptions* melds together *Channels* and *Subscribers* into a single unit. You might think of subscriber as being "a process or address that can receive a CloudEvent" and subscription as "a bundle of channels within a subscriber."

## Eventing
*Brokers* and *Triggers* belong in the "Eventing" subcategory of *Knative Eventing*.
The name here emphasizes that Brokers and Triggers are most of what a developer will interact with and think about.

A third kind of record lurking under this category is the *Event Type*. These are the literal representation of *CloudEvent* attributes - the type, source, and so on.

## Sources
I described Sources as places where events flow from. In some sense, these belong in the "Eventing" category above. But in practice, these are broken out for two reasons.

The first is *SinkBinding*, which is also a mostly internal type. Its existence is entirely related to how *Sources* get wired into the wider world of *Brokers*, *Triggers*, *Sinks*, and so forth. It therefore lives in *Sources* with code that rely on it.

While a *Source* is a general interface that can be widely implemented, it’s difficult to do anything or get a sense of the possible unless you have concrete examples out of the box. *Knative Eventing* provides three reference *Sources*: the **PingSource**, the **ApiServerSource**, and the **ContainerSource**.

**PingSource** has one purpose: it produces *CloudEvents* on a schedule you provide. It used to be called *CronJobSource*, a name which tended to cause confusion with the *CronJob* records available in vanilla Kubernetes. Unlike a *CronJob*, *PingSource* doesn’t actually run any jobs. It just produces a *CloudEvent*

The **ApiServerSource** is a much more sophisticated example. It can observe changes made to raw Kubernetes records and convert these into *CloudEvents*. In a philosophical sense, this isn’t really necessary. You can already use audit events or various tools and APIs to directly watch Kubernetes records. The point here is that *ApiServerSource* provides a fairly complete example of how you can wrap an existing event-ish system into a *CloudEvents* form.

**ContainerSource** takes a bit more explanation. It’s essentially a specialized adaptor. You provide a *PodSpec* to *ContainerSource*, and in exchange, it injects information about a *Sink* to which anything running in the Pod should send its information. Put another way, it’s a simple wrapper for existing systems that can run on raw Kubernetes and learn how to send events to a URL provided externally.
While ContainerSource is useful as a reference implementation and for quickly adapting existing software, it should never be seen as the only option or the best option. Where you have a more specific source, you should use it.

While *Knative Eventing* bundles these three *Sources*, there’s no limitation on using third-party sources. There are several of these in various states of maturity, activity, and supportedness.

## Flows
You can build arbitrary graphs of computation using *Brokers* and *Triggers*. But it can be tedious. More to the point, such a hand-assembled graph encodes the structure of computation, but not its meaning.

*Flows* comes with two types, *Sequence* and *Parallel*, to make this problem a little easier. The names are fairly descriptive of the types. *Sequence* provides a way to bundle sequential steps connected via *CloudEvents* sent over *Channels*. And *Parallel* provides a way to encapsulate basic fan-out and fan-in scenarios.

## Duck types
Much of the magic in *Knative Eventing* is due to the "duck typing" capability supported by Knative’s implementation.
[Duck typing](https://en.wikipedia.org/wiki/Duck_typing) means that the compatibility of a resource for use in a Knative system is determined by certain properties being present that can be used to identify the resource control plane shape and behaviors. These properties are based on a set of common definitions for different types of resources, called *duck types*. If a resource has the same fields in the same schema locations as the common definition specifies, and the same control or data plane behaviors as the common definition specifies, Knative can use that resource as if it is the generic duck type, without specific knowledge about the resource type. Some resources may choose to opt-in to multiple duck types.

A fundamental use of duck typing in Knative is the use of object references in resource specs to point to another resource. The definition of the object containing the reference prescribes the expected duck type of the resource being referenced.