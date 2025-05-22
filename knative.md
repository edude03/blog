# Knative for the Kurious Kubernetes Admin

One challenge I see with Kubernetes adoption at most companies is onboarding the "typical" engineer who isn't familiar with deploying their services using an orchestrator. I'm always quick to mention that Knative would likely help decrease that learning curve, but am usually rebuffed with "oh, we don't need functions as a service" - assuming they've even heard of knative. 

Unfortunately, Knative has really gained a reputation as "lambda for kubernetes" but that's completely the wrong way to think of it - even [the early contributors agree](https://ahmet.im/blog/knative-positioning/) that the marketing could have been better.

So - here's my attempt to dispel some misconceptions, and convince you that knative might be a good fit for you even if FaaS isn't ideal for your workload. 

## Misconceptions 

Probably the biggest misconception is Knative is about functions as a service, but in fact it's main aim is to make building **K**ubernetes **Native** faster, simplier and easier to maintain and as a side effect - allowing your applications to scale to 0 when not being used. 

You see, Kubernetes is a framework for orchestrating containerized workloads, and while it makes it easy/easier to follow best practices, it doesn't force you to. Don't have a health check? Fine. Don't set resource requests/limits? Fine. Running your container as root with host paths mounted and `CAP_SYS_ADMIN`? Sure, seems legit. 

Knative on the other hand, forces you to adhere to best practices essentially by assuming you're already following them.

Take for example the simplest Knative example deployment:

```yaml
apiVersion: serving.knative.dev/v1 
kind: Service 
metadata: 
	name: helloworld-go 
	namespace: default 
spec: 
	template: 
		spec: 
			containers: 
				- image: ghcr.io/knative/helloworld-go:latest 
				  env:
					- name: TARGET 
					  value: "Go Sample v1"
```

Looks kind of like a simplified Pod Spec yeah? Underneath the hood however this will generate & deploy a `Deployment` (with `SecurityContext` set and a readiness/liveness check), `HorizontalPodAutoscaler`, `Service` and `PodDisruptionBudget` as well as injecting the environment variables you need to send your telemetry signals to the global collector. 

How does it know what port your app listens on? It doesn't, your app needs to listen on `$PORT` (the environment variable it injects) or your application won't start. How does it know what command to run in my container? It doesn't, it expects the `ENTRYPOINT` to just work. How does it know when my app is ready? it scrapes `/healthz` and your app won't start if it's not defined. Etc, etc. 

It does more than this of course, for example, automatically numbering your deployment versions (service-v1, service-v2 etc) and does a rolling deploy between the versions but even with just the things I listed above, you can see how a good 90% of footguns/best practices are addressed out of the box.

So what does this have to do with FaaS? Well turns out if your app fits into the opinions of Knative then it also likely can stop running when there are no requests to it. Since Knative knows if your application has traffic, and already defines a `HPA` it's easy to "just" set the `spec.replicas` to 0, then back to `1` when it observes a request come in. 

## Serving? Eventing? `fn`? 

Confusingly to many, knative has three mostly unrelated (sub)components `Serving`, `Eventing` and `fn`. What I've been talking about above is the `Serving` component, but while I have your attention let me explain the other two parts. 

Back to the opinions Knative has about your application, it assumes that most applications will want to deal with events somehow - typically webhooks, but essentially Eventing turns different types of events into a standardized interface. At Sproutfi for example, we used this functionality to trigger alerts from kafka messages but point being any of the [supported sources](https://knative.dev/docs/eventing/sources/) will trigger essentially a webhook with a standardized schema to your application. Combined with scale to zero it becomes a convenient way to build data processing pipelines that only run when needed (like we did with benthos in our case)

`fn` on the other hand is something like `helm` for Knative. It allows a developer to manage the lifecycle of their application via the cli, so you can `create` a `fn`, which generates code using templates (see [the python one for example](https://github.com/knative/func/blob/main/docs/function-templates/python.md)), you can `build` a `fn`  which uses [Cloud Native Build Packs](http://buildpacks.io) under the hood, which allows you to build your application without dockerfiles, and of course, you can `deploy` a `fn`, and `list` `fn`s that have been deployed. 

So - although all these pieces work together, you can pick and choose which pieces make sense for your application. 

## Overhead (technical & emotional)

The TL;DR to overhead is there essentially is none. Other than needing to run the Knative operator and the Knative gateway (which, IIRC anything that supports the IngressGateway spec works) there is no performance hit to your application - even in the scale to 0 case, it "just" uses labels to decide if traffic is going to go to the gateway, or directly to your app. 

As for having to learn something new - not really? Yes, Knative is an abstraction over Kubernetes, but if you think about it as sane defaults for your Deployment, the mental load becomes a lot lighter. Developers who already know Kubernetes don't have to learn much, since it "just" creates the objects you'd expect to see anyway and standard tooling just works, and for developers who don't know much about kubernetes, it gives them an easy onramp as it points out things they need to do to run an application in a production ready way.

## Outro

So in summary, 

- Knative just generates normal Kubernetes object, no magic
- Knative has a few addons, but they're optional
- Scale to Zero is a side effect of the opinions it has, not a requirement
- Easy to adopt, since all the normal tooling and knowledge transfer over

I hope this makes sense, and I hope this encourages you to give Knative a second look if you've dismissed it in the past. If you want to kick the tires, with `k3d` and `fn` you can have a demo function working in under 30 minutes. If you do let me know how you like it and if you ran into any rough edges.

