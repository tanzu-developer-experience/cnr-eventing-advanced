The underlying model of Eventing is this: things happen in the world. Some of these happenings are observed. Some of these observations are transmitted from place to place. Receivers of the transmission read the observation, deduce some state of the world, then perhaps take actions. These actions perhaps cause still other observations to arise.

*Knative Eventing* is for the bit in the middle. It doesn’t do the happenings. It doesn’t observe the happenings. But it does take observations and transmit those. The format used for this purpose is **CloudEvents**, the mechanisms are **Channels** and **Filters**, **Brokers** and **Triggers**, **Sources** and **Sinks**, **Sequences** and **Parallels**. 

## *CloudEvents*

Events are everywhere. However, event producers tend to describe events differently.

The lack of a common way of describing events means developers are constantly re-learning how to consume events. This also limits the potential for libraries, tooling and infrastructure to aid the delivery of event data across environments, like SDKs, event routers or tracing systems. The portability and productivity that can be achieved from event data is hindered overall.

*CloudEvents* is a specification for describing event data in common formats to provide interoperability across services, platforms and systems.

*CloudEvents* have a two-part structure with data and attributes. Data is what you probably guessed: the part that systems squirt their payloads into. What is of more interest to us now is the attributes. These are roughly analogous to HTTP headers. Like HTTP headers, there is potentially an unlimited number, because anyone can add their own. But only a handful are standardized, which makes my job a bit easier.

### Required attributes
There are four required attributes. These are found on every *CloudEvent*, without exception. If any of these attributes is missing, you don’t have a *CloudEvent*.

- **specversion** This signals the version of the *CloudEvents* spec that should be referred to for a given *CloudEvent*. At the time of writing, there’s only one allowed value: 1.0
- **source** This is where the event occurs. The "where" here is a logical concept, not a physical one. For example, you might provide a *Source* that is something like abc-123.xht2kld.cdn.example.com. But most of the time, you’ll want something more abstracted from the actual layout of machines and networks. It makes more sense for cdn.example.com to be the source, with particular machines identified in data as necessary
- **type** This is the kind of thing. For example, you might have com.example.cdn/flush as a type of *CloudEvent* for a caching service. It’s common practice to use reverse domain notation to scope this to your particular service
- **id** This is meant to be unique per source. It’s not enough to pick something that is unique per machine, per network, and so forth. This is a surprisingly tricky requirement but necessary to enable reliable downstream management of *CloudEvents*. Use "version 4" UUIDs for your id field

### Optional attributes
Besides the mandatory, there are also optional attributes in the core standard.

- **datacontenttype** This is the MIME([RFC 2046](https://datatracker.ietf.org/doc/html/rfc2046)) content type of the data value. The simplest, most likely case is `application/json`. In fact, it’s considered so likely that a *CloudEvent* without a datacontenttype is assumed to be `application/json`
- **dataschema** This lets you point to a schema against which the *CloudEvent’s* data can be validated. At a higher level, this field is where folks building standards and APIs on top of *CloudEvents* will need to reach an agreement. 
- **subject** Subject is meant to be the “thing” that the event is about. You might wonder how this differs from source. After all, you can make the source value as specific as you like. The difference is that subject can identify particular instances or individuals in the population of the source. If my source is hitgub.example.com, my subject might be /repos/123
- **time** When the observed event occurred, according to whatever software created the *CloudEvent*, encoded as a Timestamp in RFC 3339 format. That sounds great, but bear in mind: There are no timecops who go around ensuring that timestamps are in any way consistent. 

### Extension attributes

The attributes you have seen so far are part of the core specification. But there’s also a variety of "Documented Extensions" that are available in secondary specifications. 

- **dataref** It won’t be rare that you’d like to send big chunky things over a *CloudEvent*. dataref lets you point to someplace else for the data field. For example, You might emit a NewCustomer event, which includes various personally identifiable information. So instead of sending a data section, you push them into a trusted service and add a URL to dataref.
Note that data and dataref are not mutually exclusive. You can have both, if you want. 
- **traceparent** and **tracestate** These two keys are intended to carry distributed tracing information according to the W3C Trace Context standard.
- **partitionkey** The typical purpose of a partition key for the event is to define a causal relationship/grouping between multiple events. In cases where the *CloudEvent* is delivered to an event consumer via multiple hops, it is possible that the value of this attribute might change, or even be removed, due to protocol semantics or business processing logic within each hop.
- **rate** A typical feature of metrics systems is the ability to sample. This is a fancy way of saying drop randomly chosen data points when there are too many to forward in a timely fashion. The rate attribute allows a *CloudEvent* to carry a signal of what the sampling rate was when it was created.
- **sequencetype** and **sequence** Sometimes, you just need a sequential numbering scheme to make sense of things. These two fields let you do that, but the definition is a bit scanty.
The only defined sequencetype value is Integer. Once set, this means that sequence should be a signed, 32-bit integer, starting at 1, incrementing by 1 for each *CloudEvent*.

### Content Modes
The *CloudEvents* specification defines two content modes for transferring events from a source to a destination: **structured** and **binary**.

With the structured content mode the event is fully encoded using a stand-alone event format and stored in the message body.
```
{
    "specversion" : "1.0",
    "type" : "com.example.type",
    "source" : "/example/source",
    "id" : "82C32673-0C78",
    "time" : "2020-04-10T01:00:05+00:00",
    "datacontenttype" : "application/json",
    "data" : {
        "foo": "and likewise bar"
    }
}
```

For an event in binary content mode the event data is stored in the message body, and event attributes are stored as part of message meta-data.
```
POST /example/event HTTP/1.1
Host: example.com
Content-Type: application/cloudevents+json
ce-specversion: 1.0
ce-source: /example/source
ce-id: 82C32673-0C78
ce-time: 2020-04-10T01:00:05+00:00
 
{
    "foo": "and likewise bar"
}
```

## Your first experience with *Knative Eventing*

Let's first check the current state using the kn CLI.
```terminal:execute
command: kn trigger list && kn source list && kn broker list
```
Under the hood, these commands are groping around for *Trigger*, *Source*, and *Broker* records in Kubernetes. You haven’t done anything yet, so there are none.

A *Trigger* combines information about an event filter and an event subscriber together, a *Source* is a description of something that can create events. 
In reality, Triggers don’t really do anything in themselves. They’re records that get acted on by a *Broker*, with potentially many *Triggers* per *Broker*.
A *Subscriber* here is anything that *Knative Eventing* knows how to send stuff to.

Let's start with the creation of a *Broker* ...
```terminal:execute
command: kn broker create default
```
... and continue with a *Subscriber*. The simplest thing to put here is a basic web app that can receive *CloudEvents* and perhaps help you to inspect those. 
```terminal:execute
command: kn service create cloudevents-player --image ruromero/cloudevents-player:latest --env BROKER_URL=http://default
```
After the deployment of the *Service* there should be a URL to access the application in the logs. If you open the URL, you should see a form with fields for all the required attributes we talked about in the previous section to send an event to the *Broker*.
To also consume the events from the broker with our application and view them in the blank area on the right, we have to create a *Trigger*.
```terminal:execute
command: kn trigger create cloudevents-player --sink cloudevents-player
```
If you now head back to your browser and try sending an event. You’ll see that the event is both sent and received.
The simple fact here is that I’ve cheated by making the application both the *Source* and *Sink* for events.
The nomenclature of "sinks" and "sources" is already widespread outside of Knative in lots of contexts. *Sources* are where events come from, *Sinks* are where events go.
You didn’t explicitly define a *Source* and only defined a *Sink* in the *Trigger*. This already hints at how flexible *Knative Eventing* actually is.

Let's have a closer look at our created *Trigger*.
```terminal:execute
command: kn trigger describe cloudevents-player
```
As you can see it sets `cloudevents-player` as its *Sink*. Many things can act as a *Sink* without knowing it and they don't have to be Knative Services as you will see at the end of the chapter.
Like with the *Knative Serving* Kubernetes records, there are also **Conditions*

Right now the conditions are all `++`, indicating a state of swellness and general good humor. As with other Knative conditions, `Ready` is the logical `AND` of all the others. If any other condition isn’t `OK`, then `Ready` will not be `++`.
Let's look a little closer at the other conditions:
- **BrokerReady** This signals that the *Broker* is ready to act on the *Trigger*. When this is false, it means that the *Broker* can’t receive, filter, or send events
- **DependencyReady** Because you defined the *CloudEvents* player service as both *Source* and *Sink*, this doesn’t have a deep meaning. This is really meant to tell you how a standalone source, such as PingSource, is doing. It gives you a fighting chance of guessing whether stuff broke because you broke it or whether some external service broke it instead
- **SubscriberReady** This is where the naming of things in *Knative Eventing* starts to get squiggly. The practical point here is that *SubscriberReady* is about whether the *Sink* is `OK`. It’s not a reference to the *Subscriber* field of a *Trigger*.
- **SubscriberResolved** *BrokerReady*, *DependencyReady*, and *SubscriberReady* are all about the status of software that’s running someplace else. That means it can change from time to time.
But *SubscriberResolved* is a once off. It refers to the business of resolving the subscriber (turning the Sink into an actual URL that can actually be reached). That happens right after you submit a *Trigger*. *Knative Eventing* picks up the *Sink* and shakes it to see what’s inside. It might be a directly hard-coded URL that you gave. It might be a Knative service. But it can also be a number of different things, such as a Kubernetes *Deployment*. All of these have some differences in how you get a fully resolved URL, but a fully resolved URL is what the Broker needs to do its job of squirting *CloudEvents* over HTTP.
When it’s `OK`, *SubscriberResolved* can be ignored. But if it’s false, then your *Trigger* is wedged. You won’t fix it directly by tinkering with the *Broker*, *Source* (aka dependency), or *Sink* (aka subscriber). To be sure, if those are misbehaving, fix these first. But you will still need to update the *Trigger* in order to get another go at URL resolving.

The conditions you saw do go beyond a game of true-or-false, so each of these conditions has relatives for things that are broken.
- **BrokerDoesNotExist** This appears when there is no *Broker*. It’s possible for platform operators to configure *Brokers* to be injected automatically, but it’s not the default behavior. In any case, if you see this, it means you need to use `kn broker create´
- **BrokerNotConfigured**, **DependencyNotConfigured**, and **SubscriberNotConfigured** These appear when you first create a *Trigger*. These represent that control loops take time to swirl: upon submitting records, it takes time to reconcile the desired world with the actual world. A lot of the time these will be the kind of thing you can miss if you blink. But if these hang around, something is wrong
- **BrokerUnknown**, **DependencyUnknown**, and **SubscriberUnknown** By convention, an *-Unknown* condition literally means that Knative has no idea what’s happening. The sister condition (e.g., *BrokerReady*) isn’t true, it isn’t false, it isn’t in the middle of being configured. It’s just ... unknown.
On the downside, if you see this, something weird is going on. On the upside, Knative at least has the manners to capture some details and log those.

To clean up the environment for the next section run:
```terminal:execute
command: kn broker delete default && kn service delete cloudevents-player && kn trigger delete cloudevents-player && clear
```



