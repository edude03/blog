# Thoughts on Kubecon EU 2024

`#kubeconeu2024` `#kubernetes` `#travel` `#ai`

If you didn't know by now, I'm a huge fan of Kubernetes so understandably, when given the opportunity to meet other people who share my level of enthusiasm for it, I take it.

KubeCon EU 2024 is my 4th KubeCon, but first KubeCon in Europe. Why Europe? Well, after the passing of Kris Nova I couldn't bring myself to to attend KubeCon NA 2023, and since many of the people I know in the Kubernetes community were going to Paris anyway, I figured why not break new ground and put a face to many of the people I know by twitter handle.

I did feel like KubeCon was somewhat lacklustre this time around as most of the talks were about repurposing existing solutions for ML/Generative AI on Kubernetes, and the ones that weren't were about technologies that I was already rather familiar with.

With that out of the way, here are some quick notes on projects I saw.

## Istio and Cilium on DPU

Data Processing Units, AKA network cards with a little server on them, captured my imagination in 2022 and have continued to interest me since. This year at kubecon both Isovalent and Istio had talks about offloading their Software Defined Networking solutions to a DPU freeing up resources on the host machine (and potentially simplifying deploying these services).

If you're not familiar with DPUs or want to see more use cases, keep an eye on this space as I intend to write about my own experiences with DPUs in a series of future posts.

## KCP

[KCP](https://github.com/kcp-dev/kcp) is a project that allows you to (at a high level) make virtual kubernetes clusters on top of a normal kubernetes cluster. This allows an organization to provide kubernetes clusters to users (for example, each team or department could get their own cluster) while having a separate team maintain the underlying infrastructure for the cluster.

Most interestingly, Marvin Beckers (kcp maintainer) introduced us to the idea of using kubernetes as a "Platform Engineering API" - essentially, using kubernetes to provide kuberentes clusters that provider higher level abstractions, for example allowing users to create a "Postgres instance" or a "Private Link connection".

Personally, my ambitions aren't so grand, but would love to see if using KCP would make hosting my kubernetes trainings a little easier.

## Kubernetes on Edge

I stopped in at a talk about running kubernetes on android for edge deployments. Although the technology is neat, it's not what I expected as it requires you to be able to run a custom kernel with patches for docker, which [most consumer devices are unable to do][0]

## dapr

Not much to say about [dapr](https://dapr.io). I think it's neat, I'd love to see some real world deployments of it. The talks I saw on it were essentially marketing for dapr which is fine, but I would have loved to have heard about the experiences of using dapr in production and areas where it falls short. It's another technology I'd love to kick to tires on, but since it's really meant for teams I doubt my experiences building a toy app will be representative of real world use.

## Cilium

Not much to say on cilium. I've been using it for years and it's still my favourite CNI. The talks I saw on it were again pretty introductory, but were good if you hadn't used cilium before and needed some convincing to try it.

## Sidero

I've been a happy user of Sidero for the past few years, it was good to see the team again in person. Apparently I'm known in the Sidero community as "that guy with the huge rack in his basement" 😅

## Jetbrains

Saw the new teamcity demo which was pretty cool. I'm more interesting in Kuberentes on Darwin which apparently they have running and plan to opensource?

## Things I want to know more about

- [Kubernetes hosted control planes?](https://kccnceu2024.sched.com/event/1YeQ1/revolutionizing-the-control-plane-management-introducing-kubernetes-hosted-control-planes-katie-gamanji-apple-jussi-nummelin-mirantis-cesar-wong-red-hat-tyler-lisowski-ibm-adriano-pezzuto-clastix)
  - What does it do? What's it for? Is it basically CAPI?
- https://fosdem.org/2024/schedule/event/fosdem-2024-3060-what-s-new-in-containerd-2-0-/
- [Stateful's Runme](https://github.com/stateful/runme)
- [Axiom](https://axiom.co)

---

[0]: although, you _can_ run docker and kubernetes on [PostmarketOS on the Oneplus 6(t)](<https://wiki.postmarketos.org/wiki/OnePlus_6_(oneplus-enchilada)>) which is another thing I've been meaning to write about)
