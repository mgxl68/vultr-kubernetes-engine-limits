# Vultr Kubernetes Engine: What the Free Control Plane Doesn't Cover, Cluster Setup Steps, and the Limits Worth Checking First (Node Pools, HA, Storage)

The managed control plane in Vultr Kubernetes Engine costs nothing. You pay for worker nodes, load balancers and block storage, and every worker node has to sit on an instance with at least 2 GB of RAM. Those two facts settle the cost question most people arrive with, so the useful part of this guide is what sits around them: how provisioning works in the console, what node pools and high availability change, which storage class your region will hand you, and the jobs Vultr leaves to you on purpose.

VKE is Vultr's managed Kubernetes service. Vultr runs the control plane, you choose the node instance type, the node plan and the node count, and one cluster can hold several node pools. It's certified under the CNCF's Certified Kubernetes Conformance Program, which matters if you want the same manifests to behave the same way after a move between providers.

> **Quick verdict**
> - Good fit: teams already billing on Vultr, small production clusters, staging environments meant to mirror production, anyone who wants a conformant Kubernetes API without a control plane invoice.
> - Poor fit: workloads that need bare metal workers, teams expecting an ingress controller to be installed for them, anyone who wants shell access to the control plane.
> - 👉 [Check VKE node plan rates on the official pricing page](https://www.vultr.com/pricing/?ref=9738262-9J)

## What Vultr Kubernetes Engine bills you for

Vultr's own FAQ states the model plainly: the managed control plane is included free of charge, and you pay for the worker nodes, load balancers and block storage you deploy. Nothing there is exotic. The part that catches budgets is that a load balancer is a real cloud server instance, not a free attachment to the cluster, and block storage is billed as the volumes you create. A cluster running three persistent volumes carries three storage lines on top of the node lines.

There's no VKE bundle to buy. The "node plan" you select during provisioning is a standard Vultr cloud compute plan, so the names, vCPU and memory combinations and hourly rates come from the general instance catalogue and change over time. That's why there's no fixed price ladder to print here: the rate you pay depends on which compute plan you attach and how many nodes you run.

| What you pay for | Where you set it | Billed to you | Verified limit | Current rate |
| --- | --- | --- | --- | --- |
| Managed control plane | Automatic with the cluster; HA is a creation-time option | No, $0 | HA spreads the control plane across multiple nodes | [Included free](https://www.vultr.com/pricing/?ref=9738262-9J#kubernetes-engine) |
| Worker nodes | Node Pool Type, Node Pool Plan, Number of Nodes | Yes, at standard compute rates | Instances with 2 GB of RAM or more | [Node plan rates](https://www.vultr.com/pricing/?ref=9738262-9J) |
| Vultr Load Balancer | Created for your Kubernetes LoadBalancer/Ingress services via the Cloud Controller Manager | Yes | Also runs on cloud server instances with 2 GB RAM or more | [Load balancer rates](https://www.vultr.com/pricing/?ref=9738262-9J) |
| Block storage volumes | Persistent volumes through the CSI driver | Yes | 10 GB to 100 TB on NVMe, 40 GB to 40 TB on HDD, depending on class | [Block storage rates](https://www.vultr.com/pricing/?ref=9738262-9J) |

The console helps before you commit: the Summary panel in the cluster creation form shows a monthly cost preview, so you can see what a given plan and node count add up to at list rates before clicking Deploy Now.

## Provisioning a cluster from the console

The console path is short enough that most people finish it in one sitting. The order of fields below matches the current documentation.

1. Go to **Products**, click **Kubernetes**, then **Add Cluster**.
2. Name the cluster in **Cluster Name**.
3. Pick a version from the **Kubernetes Version** dropdown. The list of offered versions changes over time, so choose from what's shown rather than from a version you read about elsewhere.
4. Optionally enable **High Availability** for a multi-node control plane, and **Vultr Firewall** to filter traffic.
5. Under Cluster Configuration, choose a **Node Pool Type** (the instance family) and a **Node Pool Plan** (the size within that family).
6. Give the pool a **Label** and set **Number of Nodes**.
7. Choose a **Vultr Location**. If the plan you picked isn't offered there, the form tells you to select a different plan for that location.
8. In **VPC Network**, select an existing VPC or create a new one.
9. Repeat with **Add Another Node Pool** if you want pools of different shapes in the same cluster.
10. Review the monthly cost in the **Summary** panel, then click **Deploy Now**.

You'll get a confirmation reading "VKE cluster created! Please allow a few minutes for your cluster to be ready," and the cluster then shows up in the Kubernetes list.

The console isn't the only door. Vultr documents four ways to create a cluster: the console, the API, the CLI and Terraform. If you want the cluster wired into an existing pipeline, the API route gives you the kubeconfig directly:

bash
curl -H "Authorization: Bearer ${VULTR_API_KEY}" \
  https://api.vultr.com/v2/kubernetes/clusters/{cluster-id}/config \
  -o kubeconfig.yaml


The same config file is downloadable from the console for an existing cluster. Terraform users get the same declarative workflow they use for the rest of Vultr's resources, and Cluster API compatibility means the YAML you already use to describe clusters elsewhere can target VKE through the Cluster API provider.

## Four decisions to make before you click Deploy

**How many nodes per pool.** A node pool with a single node throws an official warning: "Using only one instance can lead to downtime. To avoid this, increase your number of nodes." Vultr's own guidance is at least two nodes for production workloads. One node is fine for kicking the tyres; it is not fine for anything that pages you.

**Whether to turn on HA.** High availability places the control plane across multiple nodes instead of a single one. If a control plane node fails, the remaining nodes keep the Kubernetes API available, so you can still manage and schedule workloads while the failure is dealt with. It's a checkbox in the creation form, and it's the difference between an API outage and an inconvenience when something goes wrong underneath you.

**Which VPC you attach.** An existing VPC can only be attached to a new VKE cluster if both sit in the same region. Leave the field alone and VKE provisions a new VPC in the cluster's region automatically. That default is convenient, and it also means a cluster created without thinking about networking won't share a network with the rest of your infrastructure. If your workloads need to reach existing private resources, decide this at creation time rather than after.

**What serves your ingress traffic.** VKE does not ship an ingress controller. Vultr Load Balancers work with whatever controller you deploy, and Vultr names Nginx, HAProxy and Traefik as common choices. The Cloud Controller Manager handles the plumbing: it watches your services and provisions a managed load balancer when you expose a LoadBalancer or Ingress service. So the load balancer appears on your bill without you ordering it, and the controller on top of it is your install, your config and your upgrade cycle.

If public node IPs are a concern, Vultr also documents a VPC-only cluster pattern where access goes through a load balancer or NAT Gateway port forwarding rules.

## Storage: two classes, and the region decides which one you get

The CSI driver exposes two storage classes, and they aren't interchangeable:

| Storage class | Media | Volume size range | Availability | Default behaviour |
| --- | --- | --- | --- | --- |
| `vultr-block-storage` | NVMe | 10 GB to 100 TB | Some locations | The CSI uses NVMe by default when it's available |
| `vultr-block-storage-hdd` | HDD | 40 GB to 40 TB | Majority of locations | Used when NVMe isn't offered |

Most locations support the HDD class, which gives you the wider object count but slower disks. NVMe is the one you want for databases and anything sensitive to small random I/O, and it's only in some regions, which is the detail that bites people who pick a location first and read the storage docs later.

You can check before deploying. The `/v2/regions` API endpoint returns two fields per region: `block_storage_storage_opt` for HDD and `block_storage_high_perf` for NVMe. Query it once, note which of your candidate regions returns NVMe, and you won't discover the mismatch after your StatefulSet is running.

## What's managed, what's yours

Vultr manages the control plane and integrates the cluster with its other services. The pieces you'll actually touch:

- **Vultr Cloud Controller Manager** connects Vultr features to the cluster, monitors node state, assigns IP addresses and deploys managed load balancers for LoadBalancer and Ingress services.
- **Vultr CSI driver** provisions persistent volumes against Vultr block storage, which is how the two storage classes above reach your pods.
- **Cluster API support** lets you create and manage VKE clusters through the same provider model used on other platforms.
- **ETCD is encrypted and backed up regularly** on VKE, according to Vultr's FAQ. You don't get direct access to it, so application-level backups remain your responsibility.
- **GitOps and observability** aren't bundled, but the cluster is a normal conformant target: Argo CD and Flux for declarative deployments, Prometheus, Grafana and OpenTelemetry for metrics and traces. Horizontal Pod Autoscaling and cluster autoscaling work the way Kubernetes expects.

The counterweight to all that is a short list of things VKE won't do. There's no bare metal support: Kubernetes on Vultr runs on cloud server instances, and bare metal servers can't join a cluster as workers. There's no preconfigured ingress controller. And if you want to run the control plane yourself, that's a different product: Condor, Vultr's self-managed Terraform-based option, lets you configure the control plane and the other components however you like.

## FAQ

**Is the control plane really free?**
Yes. Vultr's pricing FAQ says the managed control plane is included free of charge and that you pay for worker nodes, load balancers and block storage. The bill is entirely made of the resources attached to the cluster.

**Can I add bare metal servers as worker nodes?**
No. Kubernetes on Vultr uses cloud server instances, and bare metal isn't supported for cluster nodes.

**Does VKE come with an ingress controller?**
No. You install your own; Nginx, HAProxy and Traefik are the ones Vultr lists as popular. Load balancers provisioned by the Cloud Controller Manager will front whichever controller you choose.

**Can I reuse a VPC from another region?**
No. An existing VPC can only be attached when it's in the same region as the new cluster. Otherwise VKE creates a fresh VPC in the cluster's region for you.

**How do I get the kubeconfig?**
Either from the console for an existing cluster, or through the API endpoint `/v2/kubernetes/clusters/{cluster-id}/config` with a bearer token, which writes the file straight to disk.

**What's the minimum sensible size for production?**
Two nodes per pool as a floor, per Vultr's own warning about single-node pools. If you also want the API to survive a control plane node failure, enable HA at creation.

## Picking the right shape for your case

A single-node cluster with one small pool is the cheapest way to learn how VKE behaves, and it's fine as long as nothing real depends on it. For production, two or more nodes per pool plus HA on the control plane covers the two failure modes you're most likely to hit: a node going down mid-week and a control plane node failing while you're on call. If you need heterogeneous capacity, say a couple of large nodes for stateful workloads and several small ones for stateless services, that's what multiple node pools are for.

Reach for Condor instead if you want to own the control plane configuration. Skip VKE entirely if you need bare metal workers or an ingress stack that's configured before you log in; those aren't gaps in the documentation, they're product boundaries.

Two things to confirm before you spend anything: that the node plan you want is offered in your chosen region, and that the region returns NVMe block storage if your workloads need it. Both checks take a minute and both are painful to fix later.

👉 [Start with the Vultr Kubernetes Engine product page and see current offers for new accounts](https://vultr.com/kubernetes?ref=9738262-9J)
