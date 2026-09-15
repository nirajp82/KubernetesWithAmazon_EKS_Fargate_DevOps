# 7. Creating Your First EKS Cluster
*Section 2: EKS Basics · ~10 min*

## Moment of truth: spinning up a real cluster

Everything so far has been theory and tooling. Time to actually create an EKS cluster with `eksctl`.

```bash
eksctl create cluster \
  --name eksctl-test \
  --nodegroup-name ng-default \
  --node-type t3.micro \
  --nodes 2
```

- `--name` — the cluster's name (`eksctl-test`).
- `--nodegroup-name` — the name of the worker-node group it creates. This is **optional**; you can create a cluster with no node group at all if you want.
- `--node-type` / `--nodes` — **always set these explicitly.** As covered in [lecture 2](02_eksctl.md), the default worker type is a non-free-tier `m5.large`. `t3.micro` keeps you on the Free Tier.

Copy that command into a terminal that has both the AWS CLI and `eksctl` installed, hit enter, and wait. This genuinely takes **5–10 minutes**, because `eksctl` is doing a lot under the hood — VPCs, subnets, security groups, and more.

> **Note on the `--version` flag:** the original recording pins this to Kubernetes 1.16, which is long past EKS's supported version window by now. In practice, either omit `--version` to get EKS's current default, or check the [AWS docs for currently supported EKS versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) before picking one.

## What's actually happening behind the scenes

`eksctl` doesn't talk to EKS directly — it submits **CloudFormation** stacks, exactly as described in [lecture 2](02_eksctl.md). For one `eksctl create cluster` call, you get (at least) two stacks:

- `eksctl-eksctl-test-cluster` — the EKS cluster itself.
- `eksctl-eksctl-test-nodegroup-ng-default` — the worker node group.

```mermaid
flowchart TD
    A["eksctl create cluster --name eksctl-test ..."] --> B[eksctl generates CloudFormation templates]
    B --> C["Stack 1: eksctl-eksctl-test-cluster<br/>VPC, subnets, security groups, EKS control plane"]
    B --> D["Stack 2: eksctl-eksctl-test-nodegroup-ng-default<br/>Auto Scaling Group, instance profile, EC2 nodes"]
    C --> E[Cluster visible in the EKS console]
    D --> E
    E --> F["kubectl is ready to use<br/>(kubeconfig updated automatically)"]
```

Open the **CloudFormation console** and you'll see both stacks. Under **Resources**, you'll find everything `eksctl` provisioned for you — security group ingress/egress rules, an Auto Scaling Group, an instance profile, and more. Under **Outputs**, you'll find things like the worker node's instance role and instance profile ARNs.

Open the **EKS console** and you'll see your cluster listed by name (`eksctl-test`), along with its Kubernetes version and other details.

## Verifying with kubectl

```bash
kubectl get all
```

Since you haven't deployed anything yet, this should return almost nothing — just the cluster's default `kubernetes` Service (a `ClusterIP` Service that exists in every namespace by default). If you see that, your cluster is live and `kubectl` is already talking to it.

### Wait — how does kubectl know *which* cluster to use?

Notice the `kubectl` command above never mentioned a cluster name. That's because `eksctl` already updated your local **kubeconfig** (`~/.kube/config`) for you when the cluster finished creating — the same file described in [lecture 5](05_kubectl_Installation_Windows.md). `kubectl` simply uses whichever cluster is the *current context* in that file.

If you have multiple clusters and need to switch between them, that's a slightly more advanced topic — a later lecture in this course covers kubeconfig contexts in depth. For now, just know: one cluster created, kubeconfig auto-updated, `kubectl` just works.

## The other way: using a config file

Typing every flag on the command line works, but it doesn't scale well once you have several node groups, or want the setup checked into version control. `eksctl` also accepts a YAML **config file** instead.

### Adding node groups to an existing cluster

Say you want to add two more node groups to your `eksctl-test` cluster — a regular ("unmanaged") node group and a Managed Node Group (the difference between the two gets its own lecture later — don't worry about it yet):

```yaml
# eksctl-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksctl-test
  region: us-west-2

nodeGroups:
  - name: ng-1-public
    instanceType: t3.micro
    desiredCapacity: 2

managedNodeGroups:
  - name: ng-2-managed
    instanceType: t3.micro
    desiredCapacity: 2
```

```bash
eksctl create nodegroup --config-file=eksctl-nodegroup.yaml
```

This targets the existing `eksctl-test` cluster in `us-west-2` and creates both node groups — each one backed by its own CloudFormation stack, same as before. Confirm it worked:

```bash
eksctl get nodegroup --cluster eksctl-test
```

You should now see **three** node groups: the original `ng-default` (from cluster creation), plus `ng-1-public` and `ng-2-managed`.

### Creating the whole cluster from a config file

The same idea works for cluster creation itself — instead of a long list of flags, point `eksctl` at a file:

```yaml
# eksctl-create-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksctl-test
  region: us-west-2

nodeGroups:
  - name: ng-default
    instanceType: t3.micro
    desiredCapacity: 2
```

```bash
eksctl create cluster --config-file=eksctl-create-cluster.yaml
```

This creates the exact same cluster as the very first command in this lecture — one node group, `ng-default`, with 2 `t3.micro` nodes. As mentioned in [lecture 2](02_eksctl.md), this config-file approach is the industry-standard way to manage `eksctl` clusters, since it's version-controllable like any other infrastructure-as-code.

## Cleaning up: delete the cluster

**This step matters.** A running cluster keeps billing you — don't skip it once you're done practicing.

```bash
# List every cluster currently running, to double-check what you're about to delete
eksctl get cluster

# Delete it
eksctl delete cluster --name eksctl-test
```

Exactly like creation, deletion works by tearing down the underlying CloudFormation stacks — the cluster, its node groups, and everything they provisioned. Confirm it's gone:

```bash
eksctl get cluster
# -> No clusters found
```

## Key takeaways
- `eksctl create cluster --name <name> --nodegroup-name <ng-name> --node-type t3.micro --nodes <n>` spins up a real EKS cluster in about 5–10 minutes; always set `--node-type` to stay Free Tier eligible.
- Behind the scenes, this submits **one CloudFormation stack per cluster and one per node group** — visible in the CloudFormation console.
- `eksctl` automatically updates your local **kubeconfig**, so `kubectl` works against the new cluster immediately with no extra setup.
- A YAML **config file** (`--config-file=...`) is the version-control-friendly alternative to typing every flag by hand, for both `eksctl create cluster` and `eksctl create nodegroup`.
- **Always run `eksctl delete cluster --name <name>` when you're done** — an idle cluster still costs money.

## FAQ

**Q: I never told `kubectl` which cluster to talk to — why does `kubectl get all` just work?**
A: `eksctl` writes the new cluster's connection details into your local `~/.kube/config` (kubeconfig) automatically as the last step of cluster creation, and sets it as the active context. `kubectl` always uses whatever the active context is.

**Q: What does `kubectl get all` show on a totally fresh cluster?**
A: Essentially nothing — just the built-in `kubernetes` Service that every namespace has by default. That's actually a good sign: it means your cluster and `kubectl` connection are both working, and you simply haven't deployed anything yet.

**Q: Do I have to give my node group a name?**
A: No — `--nodegroup-name` (and having a node group at all) is optional when creating a cluster. You can create a control-plane-only cluster and add node groups later.

**Q: What's the actual difference between `eksctl create cluster` with flags vs. with `--config-file`?**
A: They do the same thing — the config file is just a reusable, version-controllable way to express the same parameters, instead of retyping a long command every time.

**Previous:** [← 6. Installing eksctl](06_Eksctl_Install.md)
**Next:** [8. The Hidden Pod Limit Per Node →](08_PodLimitOnCluster.md)
