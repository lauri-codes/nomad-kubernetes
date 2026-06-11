---
theme: ./theme
class: text-center
hideInToc: true
---

# Deploying NOMAD Oasis with Kubernetes

## FAIRmat Users Meeting

Lauri Himanen, Technical Coordinator at FAIRmat

16/06/2026

<style>
h1 { font-size: 3rem !important; line-height: 1.12 !important; }
</style>

<!--
Welcome. This is a hands-on workshop: ~45 min of slides, ~45 min of live demoing, and ~30 min for debugging and questions at the end.

Goal: by the end you should understand *why* and *how* you would move a NOMAD Oasis from Docker Compose to Kubernetes, and be able to reproduce a deployment yourself — locally on your laptop and in the cloud.
-->

---
hideInToc: true
---

# Overview
<Toc text-2xl minDepth="1" maxDepth="1" />

---
layout: section
---

# Why Kubernetes?

---
layout: two-cols-header
level: 2
---

# Docker Compose

::left:: 

Most Oasis installations run with a single **`docker compose up`**:

- One host, one command: easy to reason about
- Great for laptops, single servers, small groups
- **nomad-distro-template** ships a ready `docker-compose.yaml`

Where it starts to strain:

- Everything lives on **one machine** = only vertical scaling
- **No self-healing**: a crashed container stays down until you act
- Scaling workers or NORTH means **editing the file and restarting**

::right::

<div class="h-full flex flex-col justify-center">
  <div class="flex justify-center">
    <img src="./assets/docker.png" class="max-h-[700px] object-contain mb-20" />
  </div>
</div>

<style>
.slidev-layout.two-cols-header {
  grid-template-columns: 60% 40%;
}
</style>

<!--
This is the architecture almost every Oasis admin already knows. The point of today is not that Compose is bad — it is excellent for getting started. The point is what happens when a single machine is no longer enough.
-->

---
layout: two-cols-header
level: 2
---

# Kubernetes

::left:: 

Kubernetes runs the **same containers**, but as a self-managing system across a cluster of machines:

- You declare the **desired state**; Kubernetes keeps reality matching it
- One Helm command deploys the **whole NOMAD stack**
- Same workflow on a **laptop** (minikube) or **many cloud nodes**

Where Compose strained, Kubernetes helps:

- **Self-healing**: crashed containers are restarted automatically
- **Scaling**: spread replicas across **many machines**
- **Rolling updates** & one-command rollback — no downtime

::right::

<div class="h-full flex flex-col justify-center">
  <div class="flex justify-center">
    <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/kubernetes/icon/color/kubernetes-icon-color.svg" class="max-h-[250px] object-contain" />
  </div>

</div>

<style>
.slidev-layout.two-cols-header {
  grid-template-columns: 60% 40%;
}
</style>

<!--
The counterpart to the previous slide: the same NOMAD containers, but now a cluster runs them for you. Keep it high-level here — the anatomy (pods, nodes, control plane) gets its own slide in the history section. The one idea to land: you describe the desired state, and Kubernetes makes it real and keeps it that way. Each bullet in the second group answers a pain point from the Compose slide.
-->

---
level: 2
---

# Docker Compose vs. Kubernetes

|  | **Docker Compose** | **Kubernetes** |
| --- | --- | --- |
| **Scope** | Single host | Multi-node cluster |
| **Scaling** | Manual (`--scale`) | Autoscaling (HPA), across nodes |
| **Self-healing** | None | Restarts & reschedules failed pods |
| **Updates** | Manual restart | Rolling updates + one-command rollback |
| **Load balancing** | Single host only | Services + Ingress across the cluster |
| **Configuration** | A compose file | Declarative manifests (desired state) |
| **Learning curve** | Gentle | Steep |
| **Overhead** | Minimal | Control plane + per-node agents |
| **Best for** | Dev, single server | Production, HA, many concurrent users |

➡️ A common path: **start on Docker Compose, grow into Kubernetes** — the same container image works on both.

<style>
table th, table td { padding: 7px 10px !important; line-height: 1.25 !important; }
</style>

<!--
Walk the table row by row. Emphasise the last row: the application doesn't change, the *orchestrator* does. That's why your NOMAD image is identical in both worlds. Land the closing line: most installs start on Compose and only grow into Kubernetes when a single machine is no longer enough.
-->

---
layout: section
---

# A bit of Kubernetes history

---
layout: two-cols-header
level: 2
---

# Where Kubernetes comes from

::left:: 

<v-clicks>

- **Google Borg**: an in-house system that scheduled containers across Google's data centers since the early 2000s. Battle-tested at planet scale long before "containers" went mainstream.
- In **2014**, Kubernetes was born as an open-source rewrite — **announced 6 June 2014**, written in **Go** (Borg was C++).
- **v1.0** shipped in **July 2015**, and Google handed the project to the brand-new **Cloud Native Computing Foundation (CNCF)** as its seed project: so **no single vendor owns it**.
- A huge community has since made it the **de-facto standard for container orchestration**, from start-ups to research data centres.

</v-clicks>

::right::

<div class="h-full flex flex-col justify-center ml-8">
  <div class="text-[0.82rem] px-3 py-2 rounded border border-gray-400/40">

  **What's in a name?** *Kubernetes* is Greek (κυβερνήτης) for **helmsman / pilot** — the same root as *cybernetics* and *governor*. Often shortened to **K8s** (K + 8 letters + s). Its **seven-spoked ship's-wheel** logo nods to the team's codename, *"Project Seven of Nine"* — a Star Trek Borg reference.
  </div>
</div>

<style>
.slidev-layout.two-cols-header {
  grid-template-columns: 55% 45%; /* Adjust to your desired proportions */
}
</style>

<!--
The one-liner: you describe the state you want ("3 replicas of the worker, this much memory"), and Kubernetes continuously works to make reality match that description. Borg ran Google for over a decade before Kubernetes existed — this is battle-tested thinking, not a science project.

Two things worth emphasising: (1) it's vendor-neutral — donated to the CNCF, which is why every cloud offers it; (2) the "helmsman" etymology and the ship's-wheel logo set up the next section nicely — Helm, the package manager, is literally the ship's wheel you steer the cluster with. The "Seven of Nine" trivia is a reliable chuckle.
-->

---
layout: two-cols-header
level: 2
---

# What is a Kubernetes cluster?

::left:: 

A **cluster** = a control plane that gives orders + worker nodes that run your containers.

**Control plane** (the brain):

- **api-server** — the front door, everything talks to it
- **scheduler** — decides which node runs a pod
- **etcd** — the database of cluster state
- **controller-manager** — drives reality toward desired state

**Worker nodes** (the muscle):

- **kubelet** — runs containers on the node
- **kube-proxy** — wires up networking
- a **container runtime** (e.g. containerd)

::right::

<div class="h-full flex flex-col items-center justify-center ml-4">
  <img src="https://kubernetes.io/images/docs/components-of-kubernetes.svg" class="max-w-full rounded-lg" />
  <div class="text-[0.8rem] mt-4 self-start">

  Key objects: **Pod** (one+ containers) · **Node** (a machine) · **Service** (stable address) · **Deployment** (keeps N replicas running)

  </div>
</div>

<!--
On a managed cloud cluster you never see the control plane — the provider runs it for you. On minikube the control plane and the single worker node both live inside one VM/container on your laptop.
-->

---
layout: section
---

# Helm: packaging for Kubernetes

---
layout: two-cols-header
level: 2
---

# Where Helm fits in

::left:: 

A real app like NOMAD is **dozens of Kubernetes objects** (deployments, services, config maps, secrets, volumes…).

**Helm is the package manager for Kubernetes** — think `apt` / `pip` / `conda`, but for clusters.

- **Chart** — a packaged, templated application
- **values.yaml** — the knobs you can turn
- **Release** — one installed instance in your cluster
- **Templating** — values + templates → plain manifests

That's why FAIRmat ships a **Helm chart** instead of a pile of raw YAML: one command installs the whole stack.

::right::

<div class="h-full flex flex-col items-center justify-center ml-8 gap-8">
  <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/helm/icon/color/helm-icon-color.svg" class="max-h-[130px]" />

```bash
# install the whole NOMAD stack
helm install nomad-oasis nomad/default \
  -f my-values.yaml
```

</div>

<!--
Analogy that lands with scientists: a Helm chart is to Kubernetes what a conda package is to Python — someone has done the hard packaging work, you just pick a version and a few settings.
-->

---
layout: section
---

# Getting a cluster

---
layout: two-cols-header
level: 2
---

# A Kubernetes server in the cloud

::left:: 

You rarely build a cluster by hand. Cloud providers run a **managed Kubernetes**: they operate the control plane, you just add worker nodes.

Using **Google Kubernetes Engine (GKE) Autopilot** as the example:

- Google runs *and* bills the control plane for you
- **Autopilot** even manages the nodes — you just deploy
- A cluster is ready in a few minutes

The same idea exists on **AWS (EKS)**, **Azure (AKS)**, DigitalOcean, etc.

::right::

<div class="h-full flex flex-col items-center justify-center ml-8 gap-6">
  <img src="https://www.vectorlogo.zone/logos/google_cloud/google_cloud-icon.svg" class="max-h-[110px]" />

```bash
# one command → a production-grade cluster
gcloud container clusters create-auto \
  nomad-oasis --region=europe-west1

# point kubectl at it
gcloud container clusters \
  get-credentials nomad-oasis \
  --region europe-west1
```

</div>

<!--
The mental shift: in the cloud you don't think about "servers" anymore, you think about a cluster you submit work to. We'll actually do this live in part 2.
-->

---
layout: two-cols-header
level: 2
---

# Local Kubernetes with minikube

::left:: 

You don't need the cloud to learn Kubernetes — **minikube** runs a full single-node cluster on your laptop.

- Same `kubectl` / `helm` workflow as the cloud
- Perfect for development and for this workshop
- Needs a driver (Docker is the easiest)

Install: **https://minikube.sigs.k8s.io/docs/start/**

::right::

<div class="h-full flex flex-col items-center justify-center ml-8 gap-6">
  <img src="https://raw.githubusercontent.com/kubernetes/minikube/master/images/logo/logo.png" class="max-h-[90px] object-contain" />

```bash
# start a local cluster
minikube start --cpus=4 --memory=8192

# enable an ingress controller
minikube addons enable ingress

# a web dashboard for the cluster
minikube dashboard

# clean up when done
minikube stop      # or: minikube delete
```

</div>

<!--
minikube is the "Docker Desktop" of Kubernetes: one binary, one command, a real cluster. Everything we show locally transfers 1:1 to the cloud cluster.
-->

---
layout: section
---

# Deploying NOMAD

---
layout: two-cols-header
level: 2
---

# The NOMAD Helm charts

::left:: 

Repo: **`FAIRmat-NFDI/nomad-helm-charts`**

- One chart: **`default`** — the full Oasis stack
- Bundles the dependencies as **subcharts** (app, worker, NORTH, MongoDB, Elasticsearch, RabbitMQ, …)
- Ready-made **`custom-values/`** for each target: `minikube.yaml`, `kind.yaml`, `aws.yaml`
- **`helpers/`** scripts that bootstrap a local cluster for you

::right::

<div class="h-full flex flex-col justify-center ml-8">

```text
nomad-helm-charts/
├── charts/
│   └── default/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── custom-values/
│           ├── minikube.yaml
│           ├── kind.yaml
│           └── aws.yaml
└── helpers/
    ├── minikube-setup.sh
    ├── kind-setup.sh
    └── check-status.sh
```

```bash
helm repo add nomad \
  https://fairmat-nfdi.github.io/nomad-helm-charts
helm repo update
```

</div>

<!--
Two ways to consume it: add the published Helm repo and `helm install nomad/default`, or clone the repo and use the helper scripts + custom-values for a turnkey local setup. We'll do the latter in the demo because it also wires up ingress.
-->

---
layout: two-cols-header
level: 2
---

# From distribution to deployment

::left:: 

Your **distribution** defines *what* NOMAD you run; the **Helm chart** defines *how* it runs on Kubernetes.

1. Start from **`nomad-distro-template`** → *Use this template*
2. Add your plugins in `pyproject.toml`
3. CI builds custom **app / worker / jupyter** images
4. Point the Helm chart at those images via **values**

The **same custom image** powers both your Compose and your Kubernetes deployments.

::right::

<div class="h-full flex flex-col justify-center ml-8">

```yaml
# my-values.yaml
nomad:
  image:
    repository: ghcr.io/my-lab/my-nomad-oasis
    tag: "1.4.2"
  config:
    services:
      api_host: nomad.my-institute.de
      api_base_path: /nomad-oasis
```

</div>

<!--
This is the key conceptual link between last session's "distribution" idea and today's deployment: the distribution produces an image, the chart consumes it. Plugins, schemas and branding all live in the image; scaling and infrastructure live in the values.
-->

---
level: 2
---

# Install NOMAD locally (minikube)

```bash
# 1. Install minikube  →  https://minikube.sigs.k8s.io/docs/start/

# 2. Get the NOMAD helm charts
git clone https://github.com/FAIRmat-NFDI/nomad-helm-charts
cd nomad-helm-charts

# 3. One-shot bootstrap: starts minikube, enables ingress, installs the chart
./helpers/minikube-setup.sh

# 4. Map the hostname so your browser can reach the cluster
echo "$(minikube ip) nomad-oasis.local" | sudo tee -a /etc/hosts

# 5. Open a tunnel (keep this terminal running)
minikube tunnel
```

Then open **http://nomad-oasis.local/nomad-oasis/gui/** — the script prints the exact URL. 🎉

<!--
The helper script encapsulates all the fiddly bits. If anyone wants the manual path: `helm dependency update ./charts/default` then `helm install nomad-oasis ./charts/default -f ./charts/default/custom-values/minikube.yaml --timeout 15m`. We'll run the script live next.
-->

---
layout: statement
hideInToc: true
---

# 🔴 Live demo 1
## minikube + NOMAD Oasis, from zero to running

<!--
Live: minikube start (or the helper), watch pods come up, open the GUI in the browser. If pods are slow, that's a perfect segue into the next section on kubectl.
-->

---
layout: section
---

# Operating a live cluster

---
layout: two-cols-header
level: 2
---

# Essential kubectl

::left:: 

`kubectl` is your window into the cluster. Add `-n nomad-oasis` to scope to the namespace.

**Inspect & observe**

```bash
kubectl get pods -n nomad-oasis    # is it up?
kubectl get pods -A                # all namespaces
kubectl get nodes -o wide          # the machines
kubectl get svc,deploy             # services / deployments
kubectl describe pod <pod>         # status + events
kubectl get events \
  --sort-by=.lastTimestamp         # what just happened
kubectl top pods                   # CPU / memory usage
```

::right::

<div class="ml-8">

**Debug & access**

```bash
kubectl logs -f <pod>              # stream logs
kubectl logs <pod> --previous      # logs of a crash
kubectl exec -it <pod> -- /bin/bash  # shell in
kubectl port-forward \
  svc/<svc> 8000:80                # reach it locally
kubectl rollout status \
  deploy/<name>                    # update progress
kubectl delete pod <pod>           # force a restart
```

</div>

<!--
These ~12 commands cover 90% of day-to-day operations. The two most-used in an incident: `kubectl get pods` to see what's unhealthy, then `kubectl logs` / `kubectl describe` on the offender.
-->

---
layout: two-cols-header
level: 2
---

# Essential Helm

::left:: 

Helm manages the **lifecycle** of the whole release, not individual objects.

**Deploy & update**

```bash
helm repo add nomad \
  https://fairmat-nfdi.github.io/nomad-helm-charts
helm repo update

# install or upgrade (idempotent)
helm upgrade --install nomad-oasis \
  nomad/default -f my-values.yaml

# something broke? go back one revision
helm rollback nomad-oasis 1
```

::right::

<div class="ml-8">

**Inspect**

```bash
helm list                    # releases here
helm status nomad-oasis      # what's deployed
helm get values nomad-oasis  # effective config
helm history nomad-oasis     # revisions
helm uninstall nomad-oasis   # tear it down
```

</div>

<div v-click class="col-span-2 mt-4 text-[0.95rem]">

To change anything — image tag, replica count, resources — **edit `my-values.yaml` and re-run `helm upgrade`**. That's the whole update story.

</div>

<!--
Contrast with Compose: there's no "edit-and-restart-by-hand". You declare the new desired state in values, Helm computes the diff, Kubernetes rolls it out with no downtime, and you can roll back instantly.
-->

---
layout: statement
hideInToc: true
---

# ☕ Questions & a short break

### Part 2 next: the same chart, in the cloud

<!--
Good moment to take questions on part 1 before we switch context to Google Cloud.
-->

---
layout: section
---

# NOMAD in the cloud

---
layout: two-cols-header
level: 2
---

# A free Kubernetes cluster for testing

::left:: 

You can try all of this on a real cloud cluster for **little or no money**.

**Google Cloud**

- New accounts get **$300 in credits / 90 days**
- GKE's monthly **free tier** covers one Autopilot/zonal cluster's management fee
- ⚠️ Compute, load balancers and storage are **still billed** — delete the cluster when you're done!

::right::

<div class="ml-8">

**What you need first**

<v-clicks>

1. A Google account + a **project** with billing enabled
2. Enable the **Kubernetes Engine API**
3. Install the local tools:
   - `gcloud` (Google Cloud CLI)
   - `kubectl`
   - `helm`

</v-clicks>

<div v-click class="mt-4 text-[0.9rem]">

Other easy options: **DigitalOcean** (free control plane), or **minikube/k3s** for fully local.

</div>

</div>

<!--
Be transparent about cost: the cluster control plane can be free, but the moment you add nodes and a load balancer you spend money. The $300 trial is plenty for a workshop; the discipline is to `gcloud container clusters delete` afterwards.
-->

---
level: 2
---

# Create a GKE Autopilot cluster

```bash
# 1. Authenticate and select your project
gcloud auth login
gcloud config set project <PROJECT_ID>

# 2. Enable the Kubernetes Engine API (once per project)
gcloud services enable container.googleapis.com

# 3. Create an Autopilot cluster — Google runs the control plane & nodes
gcloud container clusters create-auto nomad-oasis --region=europe-west1

# 4. Point kubectl at the new cluster
gcloud container clusters get-credentials nomad-oasis --region europe-west1
kubectl get nodes
```

You can do all of this in the **Cloud Console UI** too — *Kubernetes Engine → Create → Autopilot*.

<!--
We'll run this live. The cluster takes a few minutes; while it provisions I'll show the same thing in the web console so people who prefer clicking can follow along.
-->

---
layout: statement
hideInToc: true
---

# 🔴 Live demo 2a
## Create the cluster in the Google Cloud Console

<!--
Live: create the Autopilot cluster (console + gcloud), then `get-credentials` and `kubectl get nodes` to prove kubectl is wired up.
-->

---
layout: two-cols-header
level: 2
---

# Deploy the Helm chart on GKE

::left:: 

Exactly the same chart as on minikube — only the **values** change.

```bash
helm repo add nomad \
  https://fairmat-nfdi.github.io/nomad-helm-charts
helm repo update

helm install nomad-oasis nomad/default \
  -f gke-values.yaml --timeout 15m
```

⚠️ A GKE values file isn't shipped yet — we start from **`custom-values/aws.yaml`** and adapt it live.

::right::

<div class="ml-8">

```yaml
# gke-values.yaml (adapted from aws.yaml)
nomad:
  config:
    services:
      api_host: <LB_IP_OR_DOMAIN>
      api_base_path: /nomad-oasis

ingress:
  enabled: true

# cloud specifics:
#  • Service type LoadBalancer / managed ingress
#  • a cloud StorageClass for volumes
#    (GKE: standard-rwo)
persistence:
  storageClass: standard-rwo
```

</div>

<!--
The honesty slide: there's no gke.yaml in the repo today, so I adapt the AWS one. The two things that always differ per cloud are (1) how external traffic gets in (LoadBalancer / ingress class) and (2) the StorageClass for persistent volumes.
-->

---
layout: statement
hideInToc: true
---

# 🔴 Live demo 2b
## helm install on GKE

<!--
Live: helm install with the adapted values, then `kubectl get pods -n nomad-oasis -w` to watch the stack come up on the cloud cluster.
-->

---
layout: two-cols-header
level: 2
---

# View & monitor the running server

::left:: 

**Find the entry point, open the GUI**

```bash
kubectl get ingress -n nomad-oasis
kubectl get svc -n nomad-oasis -w   # wait for EXTERNAL-IP
```

Then browse to **http://&lt;EXTERNAL-IP&gt;/nomad-oasis/gui/**

::right::

<div class="ml-8">

**Keep an eye on it**

```bash
kubectl get pods -n nomad-oasis
kubectl logs -f deploy/nomad-oasis-app \
  -n nomad-oasis
kubectl top pods -n nomad-oasis
```

Plus the **GKE Console**:

- *Workloads* — health of every deployment
- *Logs* — searchable, centralised
- *Cloud Monitoring* — CPU/memory dashboards

</div>

<!--
The payoff: the same NOMAD GUI you saw on the laptop, now served from a multi-node cloud cluster. Show the console Workloads view — for many admins the graphical health view is the "aha" moment versus tailing logs by hand.
-->

---
layout: statement
hideInToc: true
---

# 🔴 Live demo 2c
## Open NOMAD in the browser & monitor it

<!--
Live: open the cloud GUI, then flip to the GKE console Workloads/Logs to monitor the live server. Then tear the cluster down (cost!).
-->

---
layout: section
---

# Recap

---
level: 2
---

# Recap

<v-clicks>

- **Docker Compose** = one host, one command — the right place to *start*.
- **Kubernetes** = many nodes, self-healing, scaling — for production *at scale*.
- **Helm** packages all of NOMAD into one installable chart: **`nomad/default`**.
- The **same chart** runs locally (**minikube**) and in the cloud (**GKE**) — only the **values** change.
- Your distribution image carries your plugins; the chart values carry your infrastructure.
- Day-to-day toolbox: **`kubectl`** to observe & debug, **`helm`** to deploy, update & roll back.

</v-clicks>

<div v-click class="mt-8 text-[1.05rem]">

➡️ Now: ~45 min hands-on — reproduce this on your own laptop — then open debugging & Q&A.

</div>

<!--
Land the single most important message: Kubernetes is not a rewrite of your Oasis, it's a different orchestrator for the same images. Adopt it when, and only when, a single machine stops being enough.
-->

---
layout: two-cols-header
hideInToc: true
---

# Resources

::left:: 

<div style="display: flex">
  <div style="min-width: 480px">
    <p><strong>NOMAD</strong></p>
    <ul>
      <li><a href="https://nomad-lab.eu/nomad-lab/">nomad-lab.eu</a></li>
      <li>Docs: <a href="https://nomad-lab.eu/prod/v1/docs/">nomad-lab.eu/prod/v1/docs</a></li>
    </ul>
    <p><strong>Deploy on Kubernetes</strong></p>
    <ul>
      <li>Helm charts: <a href="https://github.com/FAIRmat-NFDI/nomad-helm-charts">FAIRmat-NFDI/nomad-helm-charts</a></li>
      <li>Distribution template: <a href="https://github.com/FAIRmat-NFDI/nomad-distro-template">FAIRmat-NFDI/nomad-distro-template</a></li>
    </ul>
    <p><strong>Kubernetes toolbox</strong></p>
    <ul>
      <li>minikube: <a href="https://minikube.sigs.k8s.io/docs/start/">minikube.sigs.k8s.io</a></li>
      <li>Helm: <a href="https://helm.sh/docs/">helm.sh/docs</a></li>
      <li>GKE: <a href="https://cloud.google.com/kubernetes-engine/docs">cloud.google.com/kubernetes-engine</a></li>
    </ul>
  </div>
  <div class="flex flex-col items-center gap-8 mt-8 ml-8">
    <img src="https://nomad-lab.eu/nomad-lab/assets/nomad.svg" class="max-h-[90px]" />
    <img src="https://raw.githubusercontent.com/cncf/artwork/master/projects/kubernetes/icon/color/kubernetes-icon-color.svg" class="max-h-[110px]" />
    <h2>Questions & hands-on?</h2>
  </div>
</div>

::right::

<!--
Leave this up during the hands-on and Q&A block. All links are clickable in the rendered slides.
-->
