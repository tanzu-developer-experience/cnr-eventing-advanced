In the last chapter, we focused on the hard-wiring approach to *Knative Eventing*. 
Now I want to look at some of the luxury features. These basically fall into two basic categories.
- **Brokering and filtering** make the shipment of *CloudEvents* from one place to another simpler and more reliable
- **Flows** are higher-level abstractions over the wiring of *Sources*, *Sinks*, and so forth. 
Together, these luxury features can save you a fair amount of boilerplate. The goal is to express your intent at a higher level, after all, and to do so somewhat efficiently. 
**Brokers**, **Triggers**, **Sequences**, and **Parallels** provide higher abstractions that you can use to move *CloudEvents* to where these are needed with a minimum of hassle.

## The *Broker*
*Sources* and *Sinks* are all very well, but have an essential difficulty: brittleness. If a *Source* or *Sink* disappear or isn’t available, your lovely event-driven architecture becomes uneventful rubble.
Knative solves this problem by introducing indirection via the *Broker*.

A *Broker* in Knative land serves two major purposes:
- It’s a *Sink*, a place where *CloudEvents* can be reliably sent by *Sources*
- It gives life to *Triggers*. It applies their filters to incoming *CloudEvents* and forwards these to subscribers when filters match

## Filters
*Triggers* include filters. Let's create a *Trigger* together with a *Broker*.
```execute
kn broker create default && kn trigger create example-trigger --filter type=dev.knative.example.com --sink http://example.com/
kn trigger describe example-trigger
```
As you can see we added the following filter to *Trigger*: `type=dev.knative.example.com`. This says "let through any *CloudEvent* with type of dev.knative.example.com"

*Knative Eventing’s* filtering rules are strict: exact matches only. There are no partial matches, no `startsWith` or `endsWith`, no regular expressions. You can filter on multiple *CloudEvent* attributes, but this too is quite strict: all the fields must match. These are `AND`ed, not `OR`ed.
Suppose you decided to do all your triggering based on the type and source attributes of *CloudEvents*. The following listing shows how you can set up a series of triggers and their filters with kn.
```
kn trigger create trigger-1 --filter type=com.example.type --sink example-sink
kn trigger create trigger-2 --filter type=com.example.type --filter source=/example/source/123 --sink example-sink
kn trigger create trigger-3 --filter type=com.example.type --filter source=/example/source/456 --sink example-sink
kn trigger create trigger-4 --filter type=net.example.another --sink example-sink
kn trigger create trigger-5 --filter type=net.example.another --filter source=a-different-source --sink example-sink
```
Now, suppose you have a *CloudEvent* with type: `com.example.type` and source: `/example/source/123`. 
Because only exact matches will pass through the filter defined by a *Trigger* only the first two will forward the message to the sink.

The upside of this strictness with highly specific filters is that downstream systems are less likely to be accidentally overloaded by traffic with unexpected new fields or changes in demand.
The downside is that it’s inexpressive.

You have three choices. One is to wait for *Knative Eventing* to acquire a more expressive filtering system. 
Another is to perform some amount of filtering at the receiving end, meaning that some fraction of incoming *CloudEvents* is basically wasted. 
The third option is to inject additional information at the origin, against which simple filters can be applied.

You can filter on broadly any attribute in a *CloudEvent*. You can add filters for `source`, `type` and other required attributes (`specversion` and `id`). 
You can also add filters for optional attributes (`datacontenttype`, `dataschema`, `subject`, and `time`). 
And you can add these for extension attributes (like `dataref`, `partitionkey`, and so on).

Note what’s missing from this list: filtering on the body of the *CloudEvent*. Only attributes are watched by a filter!

## *Sequences*
You can use *Sources* (and *Sinks*) to wire everything together. But that’s inconvenient, so you can use *Brokers* and *Triggers* to do it more simply. But at some point, that too becomes a hassle: remembering to provide the right collection of *Triggers* and being careful to set these up in the correct order. And further, *Brokers* can become a choke point in your architecture. The answer to this problem is to more directly move traffic from place to place without passing through the central hub. *Sequences* are the annointed way to fulfill this goal.

Why not just skip the *Broker*? Well, for one thing, it is a simple and flexible way to get started.
Another reason to use the *Broker*/*Trigger* approach first is that, as of this writing, the kn CLI doesn’t support *Sequences* (or *Parallels*).

Let's now build a simple *Sequence* to demonstrate three main points: how *CloudEvents* get into a *Sequence*, how these move through a *Sequence*, and how these leave the *Sequence*.

Instead of the CloudEvents player application, we will use Sockeye which reveals a slightly lower-level view.
```terminal:execute
command: kn service create sockeye --image docker.io/n3wscott/sockeye:v0.5.0
```
The next step is the creation of the Sequence.
```execute
kubectl apply -f - << EOF
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-sequence
spec:
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: first-sequence-service
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: second-sequence-service
  reply:
    ref:
      kind: Service
      apiVersion: serving.knative.dev/v1
      name: sockeye  
EOF
``` 
The `spec.steps` block is the only compulsory part of a *Sequence* definition. It’s the truly sequential bit of *Sequences*, representing a list of destinations to which *Knative Eventing* will send *CloudEvents*, using YAML’s array syntax. Order is meaningful: *Knative Eventing* will read it from top to bottom.
`ref` is the same type of record used for sinks. You can either put a URI here or manually fill out the identifying Kubernetes fields (apiVersion, kind, and name).
The `spec.reply` section is also a Ref, but only one Ref is allowed here. Unlike `spec.steps`, this is not an array. You can again choose between a URI or Ref.

If you run ...
```terminal:execute
command: kubectl get sequence example-sequence
``` 
... you can see that the *Sequence* is not ready, because `SubscriptionsNotReady`. The *Subscriptions* in this case are the two *Services*: first-sequence-service and second-sequence-service.
You will create these now, using a simple example system provided by *Knative Eventing*.
```execute
kn service create first-sequence-service --image gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/appender --env MESSAGE=" - Handled by 0"
kn service create second-sequence-service --image gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/appender --env MESSAGE=" - Handled by 1"
kubectl get sequence example-sequence
```
What’s left for this example is to add a *PingSource*.
```terminal:execute
command: kn source ping create ping-sequence --data '{"message": "Hello world!"}' --sink $(kubectl get sequence example-sequence -o json  | jq --raw-output '.status.address.url')
```
Now, if you go to the Sockeye application, you can see the *CloudEvents* as those arrive after passing through the *Sequence*.
```copy
http://sockeye.{{ session_namespace }}.{{ ingress_domain }}
```

Note the appending of `Hello world! - Handled by 0 - Handled by 1`, which is the evidence that *Knative Eventing* shipped the *CloudEvent* via the two steps defined in the *Sequence*.

One note, you don’t need *Sources* to drive a *Sequence*. The *Sequence* satisfies the *Addressable* duck type in *Knative Eventing*. In short, anything that can send a *CloudEvent* over HTTP to a URI can be used to kick off *Sequences*. Such as, for example, a *Broker* with the right *Trigger* configuration.

To clean up the environment run:
```terminal:execute
command: kn service delete first-sequence-service && kn service delete second-sequence-service && kn service delete sockeye && kn source ping delete ping-sequence && kubectl delete sequence example-sequence && clear
```

### The anatomy of *Sequences* 
*Sequences* have three main, top-level components. These include the `steps` and `reply`, which you’ve already seen, plus `channelTemplate`.

#### Step
Steps contain destinations. You can either provide a URI, or you can provide some lower-level Kubernetes fields (the Ref) to identify what it is you’re addressing.
Suppose you have a *Knative Service* called `example-svc-1`, which answers to the URL `https:/ /svc-1.example.com`, plus another `example-svc-2`. Then I can define steps for each using either a URI or a Ref, as this listing shows.
```
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-with-uri-and-ref
spec:
  steps:
    - uri: https://svc-1.example.com
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: example-svc-2
```
Note that you can mix and match the URI and Ref formats in the same *Sequence*.

#### Reply
Reply is an optional reference to where the results of the final step in the sequence are sent to.
```
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-with-uri-and-ref
spec:
  steps:
    - uri: https://svc-1.example.com
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: example-svc-2
  reply:
    ref:
      kind: Service
      apiVersion: serving.knative.dev/v1
      name: event-display
```

#### ChannelTemplate and Channels
Channels are Kubernetes custom resources that define a single event forwarding and persistence layer.
A channel provides an event delivery mechanism that can fan-out received events, through subscriptions, to multiple destinations, or sinks.
*Channels* is intended for pipeline-type 1:1 async relationships (events are sent from A to B asynchronously, but A and B know about each other), whereas *Brokers* are intended to provide an event distribution mechanism(select events based on content you’re interested in).

ChannelTemplate defines the template which will be used to create Channels between the steps. 
A ChannelTemplate only requires that two subfields be set: `apiVersion` and `kind`. There is an optional `spec` subfields that can be anything and therefore is not validated by *Knative Eventing*.
```
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-sequence-in-memory
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1beta1
    kind: InMemoryChannel
    spec:
        # ... anything goes!
  steps:
    # ... steps, etc
``` 
In the example, the `kind: InMemoryChannel` means that *Knative Eventing* delegates the Channel here to the in-memory Channel.
Another option is e.g. `KafkaChannel`.
Unlike `InMemoryChannel`, the `KafkaChannel` does need a spec. 
```
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-sequence-with-kafka
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: KafkaChannel
    spec:
        numPartitions: 1
        replicationFactor: 1
  steps:
  # ... steps, etc.
```
Here it carries configuration information about the connection to a Kafka broker, which you’d not see on an InMemoryChannel. And the same is true for other kinds of Channel implementations - their specs are going to be specialized for that particular Channel implementation.

Setting a `channelTemplate` field tells *Knative Eventing* to perform the binding dance for you, but it won’t necessarily provision a Kafka broker. Someone needs to have installed Kafka and installed some kind of software that knows how to read and act on `KafkaChannel` records.
*Knative Eventing* provides the `InMemoryChannel` Channel implementation by default, but platform engineers can override that default for either a namespace or for an entire cluster.

A developer won’t be setting `channelTemplate` often. It might be fine to use `InMemoryChannel` for a development environment, but less acceptable in production. If you manually set a `channelTemplate`, you’d need to either maintain two versions of the record or add some kind of ChannelTemplate ... template ... to your CI/CD infrastructure. Leaving out `channelTemplate` entirely rescues you from this fate.

## Mixing *Sequences* and filters
You can mix and match *Sequences* with *Broker*/*Trigger* setups in basically any combination you please. This is again due to the magic of *duck typing*: a *Broker* can be a destination for a *Sequence* step or reply, and a *Sequence* can be a destination for a *Trigger*.

## Parallels

*Parallels* resemble *Sequences*, but there are some ergonomic differences. 
```
---
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-sequence
spec:
  steps:
    - uri: https://step.example.com
---
apiVersion: flows.knative.dev/v1beta1
kind: Parallel
metadata:
  name: example-parallel
spec:
  branches:
    - subscriber:
        uri: https://subscriber.example.com
```
In *Sequence*, each entry in the `spec.steps` array is a destination—a URI or Ref, as desired.
In *Parallel*, the top-level array is `spec.branches`. It’s not an array of destinations. It’s an array of branches.
Each branch has one required field: a subscriber, which is a destination. Again, you can use a URI or Ref here.
So why the extra level of indirection via subscriber, between `spec.branches` and uri or ref? It exists because a branch can actually carry quite a bit of optional configuration:
- `filter` A specialized destination that can pass or reject a *CloudEvent*
- `reply` It’s our old friend, Reply, but you can set one for each and every branch if you like.

The filter in *Parallel* is not the same as a filter on a *Trigger*. It’s just a completely different, completely unrelated thing. 
Instead of being a rule that’s applied by a *Broker* or *Broker*-like system, a Branch’s filter is a destination. It’s a URI or Ref to which a *CloudEvent* is sent by *Knative Eventing* and then whatever lives at that destination has to give a thumbs up or thumbs down.
If you squint a bit, the combination of filter and subscriber is a lot like a two-step *Sequence*. The *CloudEvent* flows to the filter, then from the filter onto the subscriber.
But realistically, the filter and the subscriber are both fully-fledged processes; anything the filter can do, the subscriber can, and vice versa. In terms of expressing developer intention, it’s a nice separation and resembles guarded clauses. But the overhead of routing through a process to get a pass/fail decision can prove to be fairly hefty.
When should you use a filter on *Parallel* branches? The view of the author of "Knative in Action" is that you shouldn’t, with one exception. If your subscriber is an expensive or limited resource, you will want to shed as much unwanted demand before you reach it. For example, I might be running a system where I want to send some small fraction of *CloudEvents* to an in-memory analytics store for further analysis. Rather than inserting everything coming off the wire, I would prefer to shed load before reaching the database. In this scenario, the filter is a useful ally.

The simplest thing you can do with a Parallel is to pretend it’s a Sequence.
```execute
kubectl apply -f - << EOF
apiVersion: flows.knative.dev/v1beta1
kind: Parallel
metadata:
  name: example-parallel
spec:
  branches:
  - subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: first-branch-service
    reply:
      ref:
        kind: Service
        apiVersion: serving.knative.dev/v1
        name: sockeye
EOF
```
```execute
kn service create first-branch-service --image gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/appender --env MESSAGE=" - Handled by branch 0"
kn service create sockeye --image docker.io/n3wscott/sockeye:v0.5.0
kn trigger create parallel-example --filter type=com.example.parallel --sink $(kubectl get parallel example-parallel -o json  | jq --raw-output '.status.address.url')
```
You now have a *Trigger* to send matching *CloudEvents* to the *Parallel*’s URI. The *Parallel* sends that *CloudEvent* on to the *Service* I created, which appends `FIRST BRANCH` to whatever *CloudEvent* message passes it by. Then the *CloudEvent* should pop up in in the Sockeye application.
To try this out, execute the following command.
```terminal:execute
command: curl -XPOST -H 'Ce-Id: $(uuidgen)' -H 'Ce-Specversion: 1.0' -H 'Ce-Type: com.example.parallel' -H 'Ce-Source: example/parallel' -H "Content-type: application/json" -d '{"message": "Hello world!"}' '$(kubectl get broker default -o json  | jq --raw-output ".status.address.url")'
```
Let's now add a second subscriber that also replies to the Sockeye application.
```execute
kubectl apply -f - << EOF
apiVersion: flows.knative.dev/v1beta1
kind: Parallel
metadata:
  name: example-parallel
spec:
  branches:
  - subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: first-branch-service
    reply:
      ref:
        kind: Service
        apiVersion: serving.knative.dev/v1
        name: sockeye
  - subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: second-branch-service
    reply:
      ref:
        kind: Service
        apiVersion: serving.knative.dev/v1
        name: sockeye
EOF
```
```terminal:execute
command: kn service create second-branch-service --image gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/appender --env MESSAGE=" - Handled by branch 1"
```
To try this out, re-execute the following command.
```terminal:execute
command: curl -XPOST -H 'Ce-Id: $(uuidgen)' -H 'Ce-Specversion: 1.0' -H 'Ce-Type: com.example.parallel' -H 'Ce-Source: example/parallel' -H "Content-type: application/json" -d '{"message": "Hello world!"}' $(kubectl get broker default -o json  | jq --raw-output ".status.address.url")
```
The *Parallel* made two copies of the *CloudEvent* and sent those to each of the branches (fan-out). Then those branches sent their reply to the same instance of the Sockeye application (fan-in).

In this example we now have two *CloudEvents* with identical `id`, `source`, and `type` fields. Any conforming implementation is within its rights to treat these as the same logical *CloudEvent*, even though these are physically distinct. When working with *Parallels*, you need to take this into consideration when you have a fan-in. For example, if one branch does some kind of conversion to a different *CloudEvent*, you are largely in the clear. But if you are merely adding to a *CloudEvent*, as we did you need something stronger. Either you should be filtering so that logical duplicates don’t arise, or you should be changing one of `id`, `type`, or `source` according to what makes the most sense.

You might now be wondering whether you will need to laboriously provide a reply for every branch. The answer is "no", for two reasons. 
1. You just might not care about the fate of a *CloudEvent* sent to a branch’s subscriber. You can leave reply out entirely. *CloudEvents* are delivered as HTTP requests, but no HTTP reply is expected or dealt with. 
2. You can provide a single top-level reply for the whole *Parallel*. This acts as a default for every branch. You can override on any branch by providing a reply specific to that branch, but otherwise, anything coming out is sent on to the top-level reply. 

That means we can rewrite the YAML to be slightly shorter.
```execute
kubectl apply -f - << EOF
apiVersion: flows.knative.dev/v1beta1
kind: Parallel
metadata:
  name: example-parallel
spec:
  reply:
      ref:
        kind: Service
        apiVersion: serving.knative.dev/v1
        name: sockeye
  branches:
  - subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: first-branch-service
  - subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: second-branch-service
EOF
```

As with *Sequence*, it’s possible to set the `channelTemplate` field on a *Parallel* at the top level. 

To clean up the environment run:
```terminal:execute
command: kn service delete first-branch-service && kn service delete second-branch-service && kubectl delete parallel example-parallel && kn trigger delete parallel-example && kn broker delete default && clear
```

## Dealing with failures
*Knative Eventing* provides allowances for failure by implementing some common patterns: retries, retry backoffs, and dead-letter destinations.

You can configure those for each *Subscription* or *Parallel* by adding a `delivery` spec to either a *Sequence* step or in a *Parallel* branch.
So what’s in a `delivery` field? Basically, it covers two things: retries plus backoffs, and dead-letters. 
All the delivery configurations are optional. 

**Retries** are a simple coping tactic for failed operations. You typically don’t want to do it forever, so a first stop in retry logic is to cap the number of times an operation is retried as this listing indicates.
```
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: example-sequence-delivery
spec:
  steps:
    - uri: http://foo.example.com
      delivery:
        retry: 10
```
A `delivery.retry` field is a simple integer. It’s defined as the minimum number of retries attempted, in addition to a first failed attempt to deliver a *CloudEvent*. So if you configre the number of retries to 10, if everything goes wrong, there will be at least 11 requests made, not 10.

It can be more than 11, because *Channel* implementations are allowed to deliver a *CloudEvent* more than once. This may come as a surprise. It’s common for various queue or message systems to have "at-least-once" guarantees. It’s much less common for these to provide "at-most-once" guarantees. "Once-and-only-once" guarantees, which is the intersection of the two, is arguably impossible, depending on how one defines the problem. This is partly why *CloudEvents* encourage you to provide unique `id`, `source`, and `type` fields to help downstream systems to sanely ignore re-deliveries. Especially since retry might be causing deliveries which are successful, but where the delivery result is incorrectly considered to be failed. In that scenario, retries cause the same *CloudEvent* to be sent multiple times.

Speaking of problems with retries, you probably don’t want to try again instantly. 
Hence the need for **backoffs**, configured with `backoffDelay` and `backoffPolicy`. The `backoffDelay` is a duration expressed in a simple format (e.g., “10s” for 10 seconds). The `backoffPolicy` describes how that duration will be used.

If you set `backoffPolicy: linear`, retries are made after fixed delays. If you have `backoffDelay: 10s`, retries are attempted at 10 seconds, 20 seconds, 30 seconds, and so on.
If you set `backoffPolicy: exponential`, retries take twice as long between each attempt. With the same `backoffDelay: 10s`, attempts are made at 10 seconds, 20 seconds, 40 seconds, and so on. The backoffDelay provides a base value that is raised by a power of 2 on each attempt (1x, 2x, 4x, 8x, ...).

All the retries may be for naught, however. One option might be to just give up entirely and let the *CloudEvent* evaporate into thin air. But if you’re losing a lot of *CloudEvents*, or if you’re using *CloudEvents* to encode information where the individual event has a high independent value, then accepting silent lossiness is not ideal.
The **deadLetterSink** is an additional guardrail against unforeseen problems like these. You nominate a place where, if all regular delivery attempts fail, a *CloudEvent* will wind up.

The dead letter pattern isn’t perfect. The sink can be down, or it can be the victim of sudden hammering when some high-demand service vanishes and it begins to receive everything. But it’s invaluable as a safety net. When it’s there and when it’s working, you get the real *CloudEvent* that failed to get somewhere, which gives you more clues as to why.

### The bad news
It's optional for Channel implementations to interpret and act on the `delivery` configuration. `InMemoryChannel` actually does so, but maybe the Channel implementation you are using, not.