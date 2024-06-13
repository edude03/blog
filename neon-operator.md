# My experience building a rust k8s operator for the first time (featuring tilt)

At Materialize we have a handful of rust based kubernetes operators for managing various operators that I've committed code to as part of my job as a Cloud Engineer, however I always felt like I didn't understand the various trade offs being made and why certain decisions were made so I figured I'd try and build an operator from scratch to get a feel for it.

Now, I wouldn't be me if I copied and pasted the toy example and called it a day, so I decided to revive a project I've had on the back burner for months now - deploying [Neon](https://github.com/neondatabase) on Kubernetes - as a practical and motivating example.

First things first though, since I was going to need a kubernetes cluster and a way to iterate rapidly, I decided to give Tilt (+kind) a shot, since it had shown a lot of promise on other projects I had experimented with.

Getting tilt setup was super easy - with a quick `tilt init` and some light editing I had something that would be able to build and deploy my operator. But of course, there was nothing to deploy yet, so let's get to scaffolding.

## Starting with the rust code

I've been using rust in production for the past 4ish years and I'm assuming the reader is also somewhat familiar with the rust ecosystem so I won't bore you - essentially I just needed to `cargo new neon-operator` and `cargo add kube-rs kube-api tokio clap thiserror tracing tracing-subscriber` (my typical litany of rust library for most projects) and I was off to the races.

## Setting up the build

The build was a little more interesting, though I'm bullish on [Cloudnative Buildpacks](https://buildpacks.io) I wasn't sure if there was any native dependencies I was going to need that would necessitate a custom base image - and since [my own custom base image](https://github.com/Melenion/rust-cnb-builder) has definitely bit rotted in 2 years I decided to go with a "simple" dockerfile.

Now, building rust projects in docker is known to be slow, mostly because of how dockers cache invalidation rules work, typically causing the image to built from scratch every time there is a code change. There are a few workarounds for this, such as using [cargo-chef ](https://github.com/LukeMathWalker/cargo-chef) however, since buildx added the `--mount=cache` directive you can build rust projects in docker almost as fast as native with no fancy tricks. The [dockerfile](https://warehouse.tail7ff6c.ts.net/Melenion/neon-operator/src/branch/main/Dockerfile) for this project is just 7 lines! Simply changing

```
RUN cargo build
```

to

```
# Cache the cargo registry, cache the build directory
RUN --mount=type=cache,target=/usr/local/cargo/registry --mount=type=cache,target=/build/target cargo build
```

is all you need [^0].

## Manifests for the operator

Again, for those familiar with kubernetes, the [manifests](https://warehouse.tail7ff6c.ts.net/Melenion/neon-operator/src/branch/main/manifests) needed weren't so surprising. Getting the RBAC rules right took a handful of iterations but nothing too exciting.

## Designing, building and iterating on the operator

The idea of the operator[^1] was basically that it would deploy all of the required infrastructure for your neon database(s) that would then allow you to deploy multiple databases like neon cloud does.

Initially, my approach was based on this [rust-kubernetes-operator-example](https://github.com/Pscheidl/rust-kubernetes-operator-example/blob/master/src/main.rs) I found, to have a reconcile loop that decides what action to take then apply it, but after trying it I realized although this is maybe the "right" way to do things, it'd be too large of an undertaking.

Let's step back and talk about how kubernetes works. At a high level, kubernetes is "just" a database of "(Custom) Resources" and then instances of those resources. A "Resource" has some required data like it's type, and metadata for it's name, but the rest of the object is just data a controller can look at and know what to do with.

In the case of a Custom Resource, you need to define that object, and tell kuberentes about it. kube-rs has a macro for this, and for this project, the object looks like this

```
#[derive(CustomResource, Deserialize, Serialize, Clone, Debug, PartialEq, JsonSchema)]
#[kube(
    group = "melenion.com",
    version = "v1",
    kind = "NeonDatabase",
    namespaced,
    shortname = "nd",
)]

pub struct NeondatabaseSpec {
    pub compute_image_ref: String,
    pub neon_image_ref: String,
    pub postgres_version: String,
}
```

The macro hides some details from you so that you can focus on the information you care about - for example, saying that your type is `namespaced` means that the `namespace` field will be required for your custom resource to be applied. Later in the reconcile loop however, you'll essentially get an instance of this `NeonDatabaseSpec` object to operate on.

Of course, kubernetes would be pretty useless if it "just" stored these objects, so it has another trick up it's sleeve - reconciliation. Kubernetes constantly compares the defined state of resources to the actual state of resources, and if those states are different - it will attempt to make them the same. It turns out however, that this is _not_ kubernetes behaviour but actually the behaviour of the controller for the resource!

So, as a custom resource builder, you need to build a tool that can, at some frequency, look at the state of your object definitions, and the state of the objects themselves and decide to do something. For example, if you increase the number of `replicas` on a deployment from 1 -> 2 the controller should notice there is 1 replica in reality, but the object says there should be two. Therefore if you were implementing a replicaset controller you'd need to ask kubernetes to create a new pod. In the example I linked, they do this with a function called `determine_action(spec, reality) that compares the two and emits an emum with the name of the action to take that then calls a function to do that thing. With this approach though, you need to anticipate all the ways your object could change, and of course how to affect that change.

I didn't have time for that however (it was a 1 day skunkworks project after all) and so after fighting with it for awhile I decided to "cheat" and use `Server Side Apply`. At a high level, to deploy all the infrastructure necessary to run Neon you could "just" apply a folder of manifests, with templating with [kustomize](https://kustomize.io) if you want to get fancy, so it follows that the reconciliation loop in this case could be simplified to `if !object.exist() { kubectl_apply(object) }` and let kubernetes manage the resulting objects. However, without Server Side Apply, the client is responsible for diffing the state, so for example, if the object already exists and you try and apply the object again it'll fail. This is handled for you in the `kubectl` client, but when building your own client, you don't get this for free.

So, I defined each object I needed in rust, configured kube-rs to do SSA, and ended up with something that looks like this:

```
async fn reconcile(context: MyThingSpec) -> Result<Action, _> {
  apply_minio(kube_client, namespace, ...)?
  apply_pageserver(kube_client, namespace, ...)?
  apply_storage_server(kube_client, namespace, ...)?

  # Tell the kube-rs controller calling this function to wait before calling this function again
  return Action::wait(duration(5))
}

```

## Testing

With the basic implementation out of the way, testing was pretty straight forward. Tilt automatically replaces image refs in objects it knows about, so redeploying the controller was literally save, wait 2 seconds, check the logs and kubernetes to see if it did the right thing, do it again until I had all the services up and running. There is a minor race condition though, the Custom Resource Definition needs to exist before tilt can replace image refs in it, but I have kube-rs generate and apply the CRD when the service starts. In this case I cheated again, and did a `kubectl get crd neondatabase -o yaml > neondatabase.yaml` and moved on, but for to scale this, I'd have to have the crd generated as part of a build script most likely.

To run the tests neon runs for local development, I needed to build a custom image, so I [added their docker-compose assets](https://warehouse.tail7ff6c.ts.net/Melenion/neon-operator/src/branch/main/orig-compose) to this repo. Since I registered the compute_image image ref with Tilt, it is able to automatically build and apply this image for testing. I could then run the included `docker_compose_test.sh` and it ran successfully, success![^2]

## Takeaways

So, I succeeded in learning about kube-rs and kubernetes operators, and managed to get a proof of concept neon operator built in ~1 day. I'm happy with the tech choices I made, however there are a bunch of things I'd do better next time namely:

### Properly implementing the reconciliation loop

I explained this above, but one thing I left out is in practice this reconciliation loop doesn't actually work in practice for two reasons:

1. There are statefulsets that can't just be `apply`d when a change is made
2. None of the services actually wait for each other

1 would probably need to be solved with the `diff` approach to decide if the statefulset needs to be deleted or if it can be updated in place
2 is complicated because each service needs a good definition of its state (at least, if it's ready or not) but also

### Better handling of parameters in rust

This is a style choice for sure, but I don't like functions that take a bunch of duplicate parameters. Ideally, I'd really like Scala style [Using Clauses](https://docs.scala-lang.org/scala3/reference/contextual/using-clauses.html) for things like `kube_client` and `owner_ref` but I'm also sure there is a pattern I could use (like passing a closure) that would clean this up.

Futhermore, having a way to apply the functions in order and as dependencies pulumi style ie

```
let minio = Minio::new(some, config).await?
let page_server = PageServer::new(some, config, minio).await?

# Then in page servers `apply` function
let env = [
  EnvVar: {
    name: S3_ENDPOINT,
    # this would implicitly create a relationship between minio and page server in the dag, so I could later
    # walk the dag and decide what needs to be created/deleted/updated
    value: self.minio.endpoint()
  }
]
```

but I couldn't figure out a nice way to do this without trying to implement pulumi in rust. A friend of mine did say I could have `Minio::new()` then in downstream functions have (minio.endpoint, minio.password) etc, but again, long functions and doesn't capture the behaviour I want.

## Fin

So that's it for now, I plan to iterate on this in my free time and address some of these issues, so expect edits to this document and possible a follow up post. Keep an eye on the repo for the tests, and hopefully this will be usable by other people in the not too distant future.

[^0]: You also need to mount the build directory again later if you want to copy files out of it for example, as well dockerfiles don't really have variables so you need to make sure these paths match your project
[^1]: I use operator and controller interchangeably after this point, though they're technically different things. A controller is a binary that runs the reconciliation loop for a resource, whereas operator is a pattern in the kubernetes community for automatically managing a certain type of thing - IE a "postgres operator" might automatically replace a failed instance, or have a way to configure backups for the postgres deployment. Both do essentially have the same reconciliation loop, just operators tend to deal with higher level features than pods and services.
[^2]: Although, I won't consider this project a success truly until I at least get the tests running in CI so that others (my future self included) can iterate from a known working point.
