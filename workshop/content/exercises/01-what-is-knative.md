The purpose of Knative(pronounced "KAY-nay-tiv") is to provide a simple, consistent layer over Kubernetes that solves common problems of deploying software, connecting disparate systems together, upgrading software, observing software, routing traffic, and scaling automatically. This layer creates a firmer boundary between the developer and the platform, allowing the developer to concentrate on the software they are directly responsible for.

The major subprojects of Knative are *Serving* and *Eventing*.
- **Serving** is responsible for deploying, upgrading, routing, and scaling. 
- **Eventing** is responsible for connecting disparate systems. Dividing responsibilities this way allows each to be developed more independently and rapidly by the Knative community.

The software artifacts of Knative are a collection of software processes, packaged into containers, that run on a Kubernetes cluster. In addition, Knative installs additional customizations into Kubernetes itself to achieve its ends. This is true of both *Serving* and *Eventing*, each of which installs its own components and customizations. While this might interest a platform engineer or platform operator, it shouldn’t matter to a developer. Developers should only care that it is installed, not where or how.

The API or surface area of Knative is primarily YAML documents. These are CRDs (Custom Resource Definitions), which are, essentially, plugins or extensions for Kubernetes that look and feel like vanilla Kubernetes.

You can also work in a more imperative style using the Knative kn command-line client, which is useful for tinkering and rapid iteration. You work with both of these approaches throughout the workshop. But first, let’s take a quick motivational tour of Knative’s capabilities.