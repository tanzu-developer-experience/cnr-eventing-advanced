*Sources* and their slightly less visible companion, *Sinks*, are the bread-and-butter concepts you use most often in *Knative Eventing*. Quickly revisiting: a *Source* describes a thing that can emit *CloudEvents* and a place (the *Sink*) where those events should be sent. *Sources* are the canonical way of describing how *CloudEvents* will move between points in a *Knative Eventing* design.
Knative’s structure is that *Sources* are the top-level concept and that *Sinks* are a component of *Sources*.

## Sources
Knative’s design is that any Kubernetes record having the two fields `Sink` and `CloudEventOverrides` can be treated as a *Source* by Knative. 

Let’s first have a look at the Sources that are installed by default.
```terminal:execute
command: kn source list-types
```
You should see the *ApiServerSource*, *ContainerSource*, and *PingSource*. Additionally there is a *SinkBinding*, which we’ll discuss later.

`kn source list-types` shows what *Source* definitions have been installed, while the `kn source list` shows *Source* records that have been created and submitted. The kubectl counterparts are `kubectl get sources` and with `kubectl get crds -l duck.knative.dev/source=true` you are able to get the CRDs for the installed *Source* definitions.

For working with *ApiServerSource* and *PingSource*, kn provides some convenient subcommands. First you have to deploy a broker and a sink.
```terminal:execute
command: kn broker create default
```
Instead of the CloudEvents player application, we will use Sockeye which reveals a slightly lower-level view.
```terminal:execute
command: kn service create sockeye --image docker.io/n3wscott/sockeye:v0.5.0
```

Now you are able to create the *PingSource* and specify the sink via the kn CLI.
```terminal:execute
command: kn source ping create ping-player --sink $(kn service list sockeye -o json  | jq --raw-output '.items[].status.url')
```
If you have a look at the details of the created *Source* ...
```terminal:execute
command: kn source ping describe ping-player
```
... you can see the `Schedule` with the rule `* * * * *` that is satisfied every minute.
As you would expect, there is also the HTTP(S) URI to which *PingSource* is meant to send the *CloudEvent* and as always, some `Conditions`.
*Data* is the actual JSON sent onwards to the *Sink*. Because we didn't specify any data at the creation of the *PingSource*, let's do this now.
```terminal:execute
command: kn source ping update ping-player --data '{"message": "Hello world!"}'
```
The kubectl/YAML counter-part for our source looks like this:
```
kubectl apply -f - << EOF
apiVersion: sources.knative.dev/v1beta2
kind: PingSource
metadata:
  name: ping-player
spec:
  data: '{"message": "Hello world!"}'
  schedule: '* * * * *'
  sink:
    uri: http://sockeye.{{ session_namespace }}.{{ ingress_domain }} 
EOF
```

If you open the application URL you see the incoming events and if you click the Message icon (✉) you can look at the *CloudEvent* more closely. 
The `root` here is not part of *CloudEvents*: it’s how our application presents the *CloudEvent* JSON object. However `attributes` and `data` are definitely part of a *CloudEvent*, as are the attribute fields for `datacontenttype`, `id`, `source`, `specversion`, `time`, and `type`.
So what about `extensions`? This is the serialized name for *CloudEventOverrides*, which you’ll remember are part of any *Source*. These are more or less what they sound like. If you set an override, then *Triggers* will treat the overridden value as the actual value. This is most useful for editing values to add context (for example, in a tracing framework). 

Truthfully, the author of the book "Knative in Action" doesn’t think you should use extensions. It’s the vestiges of a previous scheme for adding flexibility in design, which has been largely superseded by *duck types*.

## Sinks

In previous examples so far we’ve positioned the `Sink` as being a URI. It turns out that this is only one way to express “send my *CloudEvents* here.” The other is to use a "Ref" — a reference to another Kubernetes record.
```terminal:execute
command: kn source ping update ping-player --sink ksvc:sockeye
```
The kubectl/YAML counter-part for our source looks like this:
```
kubectl apply -f - << EOF
apiVersion: sources.knative.dev/v1beta2
kind: PingSource
metadata:
  name: ping-player
spec:
  data: '{"foo":"and likewise bar"}'
  schedule: '* * * * *'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: sockeye
      namespace: {{ session_namespace }}
EOF
```
There’s no `--sink-ref` or `--ref` argument to signal to kn that you’re using a Ref instead of a URI. Your intention is derived from the syntax of what you pass in. If your argument starts with http:// or https://, the assumption is that you want a URI. If instead it starts with `ksvc:`, the assumption is that you want a Ref. You could also have used `service:sockeye` as a more explicit alternative. Whichever you pick, just remember that a Knative Service is not the same as a Kubernetes Service.

Which is better? URIs are simpler to start with and allow you to target endpoints that live outside the cluster Knative is running on. But consider using Refs instead. A URI is just an address, what lies on the other side is (for good or ill) a blackbox. But a Ref is a Knative construct referring to a Knative Service. Knative knows how to open the box to peek at what’s inside.

This is most useful when things go awry. Let’s demonstrate by creating some unexpected havoc as uncovered in this listing.
```terminal:execute
command: kn service delete sockeye && kn source ping describe ping-player
```
The `Ready` Condition is a top-level rollup of other Conditions. The more diagnostic Condition is that `SinkProvided` is `!!` (not OK). The `NotFound` reason explains why. This information is surfaced because *Knative Eventing* can go and see if the nominated Ref actually exists. That’s not something it can do with URI. 

## The mysterious SinkBinding

*PingSource* is helpful for introducing the basics of working with *Sources*, but otherwise, there’s not much to talk about. *ApiServerSource* does a lot more, but you might recall that we are trying to keep Kubernetes itself at arm’s length from this workshop.

It’s now time to introduce *SinkBinding*, which is a kind of adapter plug for more or less "anything" that uses vanilla Kubernetes to run. And time also to briefly point out why *ContainerSource* is sorta-kinda a special case of *SinkBinding*.

The basic problem is this: other software already exists unaware that *Knative Eventing* exists, providing no proper *Sources* that can be easily put to good use. 
The *SinkBinding* provides a non-specialized recipe for configuring *Knative Eventing* to interact with existing systems. It has two parts. First, it has a *Sink*. Second, it has a *Subject*. The *Sink* we already know reasonably well: it can be a URI or a reference (Ref) to a Knative Service, or even to a vanilla Kubernetes record such as a Pod or Deployment. The *Subject* is also a reference to a record, such as a Knative Service, Pod, or Deployment, using the Namespace, Name, etc., fields.

Note that *Sinks* and *Subjects* are distinct types. In particular, a *Sink* points to a single thing identified either by URI or by Ref. A *Subject* can only be identified by reference and can, in theory, refer to any number of matching records. 

What does setting a *Subject* do that a *Sink* does not? After all, *Knative Eventing* can’t read your code and do anything fancy to it. But what it can do is provide enough information for your code to do the right thing. In the case of *SinkBinding*, nominating something as a *Subject* causes *Knative Eventing* to inject a `K_SINK` environment variable into the software’s container environment. The `K_SINK` variable is set to the URI of the *Subject*. By providing the `K_SINK` variable, Eventing enables the software inside the blackbox to target the Sink and send information directly to it.

For example, you might have a catalog microservice that typically POSTs updates to a shopping cart microservice. If you wanted to have Knative Serving take over this interaction, the first step would be to set your Subject to the existing shopping cart Service URL. Then you’d have the catalog POST to the address found in `K_SINK`. Later, you could change `K_SINK` to anywhere that makes sense—a Broker, for example.

*ContainerSource* closely resembles a *SinkBinding*, but it is technically a distinct kind of thing. As its name suggests, *ContainerSource* is limited to dealing with one kind of thing: containers. More accurately, it lets you set the container for a given Pod.

I think that while *ContainerSource* is easy to first grasp, you should prefer *SinkBinding* when you need to perform a quick adaptation of existing software without needing to write a full-blown *Source*. A *SinkBinding* is more general, for one thing. And it is more likely to stick around if *ContainerSource* is ever retired.

## Other Sources
In addition to the first-party *Sources* that enable you to kick the Eventing tires immediately after installation, there are a few places to look for more Sources.
- [*Knative Eventing* documentation “Knative Eventing Sources” page](https://knative.dev/docs/eventing/sources/)
- [Knative’s own eventing-contrib repository](https://github.com/knative/eventing-contrib)
- [TriggerMesh provides a growing collection of *Sources* for AWS](https://github.com/triggermesh/aws-event-sources).
- [Google first-party Sources](https://github.com/google/knative-gcp)
- [VMware has an experimental Source for vSphere and is working on others like for RabbitMQ](https://github.com/vmware-tanzu/sources-for-knative)

To clean up the environment for the next section run:
```terminal:execute
command: kn broker delete default && kn service delete sockeye && kn source ping delete ping-player && clear
```





