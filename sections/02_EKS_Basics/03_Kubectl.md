# 3. kubectl
*Section 2: EKS Basics · ~7 min*

## The steering wheel for Kubernetes

If Kubernetes is the engine, **`kubectl`** is the steering wheel. It's the official CLI that talks directly to the Kubernetes API server, and it's how you create, inspect, and delete resources like Pods, Deployments, and Services.

Unlike `eksctl`, `kubectl` is **platform-agnostic** — the exact same commands work whether your cluster is on EKS, GKE, a local Minikube, or bare-metal on-prem servers.

## eksctl vs. kubectl — the restaurant analogy

| Tool | Focus | What it handles |
| --- | --- | --- |
| **`eksctl`** | Infrastructure | AWS servers, clusters, node groups |
| **`kubectl`** | Applications | Pods, Deployments, Services running inside the cluster |

- **`eksctl` is the construction company.** It builds the restaurant, wires the electricity, sets up the kitchen and tables — your **infrastructure**. You mostly use it once, up front, or when expanding.
- **`kubectl` is the restaurant manager.** Once the building exists, it hires staff, sets the menu, seats customers, cooks the food — your **applications**. You use it every single day.

## The command structure

Every `kubectl` command follows the same shape:

```
kubectl [command] [type] [name] [flags]
```

<img width="1040" height="435" alt="image" src="https://github.com/user-attachments/assets/a65132bf-8659-4aae-b8ca-b89901c4b3c9" />

- **command** — the action: `get`, `create`, `describe`, `delete`, `apply`, ...
- **type** — the resource type, in singular, plural, or abbreviated form (`pod`, `pods`, `po` are all the same thing).
- **name** — the specific resource's name. Omit it, and you get *all* resources of that type instead.
- **flags** — optional modifiers, e.g. `-f <file>` or `-o yaml`.

```mermaid
flowchart LR
    A[You type a kubectl command] --> B[kubectl parses syntax & flags]
    B --> C[Sends an HTTP request]
    C --> D[Kubernetes API Server]
    D --> E[Executes against the target resource]
    E --> F[Response returned to your terminal]
```

## Examples

| Command | What it does |
| --- | --- |
| `kubectl get pods` | Lists all pods in the current namespace |
| `kubectl get po` | Same thing, using the abbreviation |
| `kubectl get pod pod1` | Status of one specific pod |
| `kubectl get pod pod2 -o yaml` | Full YAML detail for one pod |
| `kubectl create -f manifest.yaml` | Creates whatever's defined in the file |

## Best practices

- **Learn the common abbreviations** — `po` (pods), `deploy` (deployments), `ns` (namespaces), `svc` (services), `rs` (replicasets) — they add up fast.
- **Append `-o yaml` when troubleshooting** — the default summary view hides most of the resource's actual configuration.
- **Lean on `--help` and the docs** rather than memorizing every command up front.

## Common mistakes

- **Mixing up `eksctl` and `kubectl`** — trying to provision infrastructure with `kubectl`, or deploy an app with `eksctl`.
- **Wrong namespace** — if a resource you expect isn't showing up, you're probably querying the wrong namespace.
- **Wrong flag order** — flags go at the end (`kubectl get pod -o yaml`), not before the command or type.

## Key takeaways
- `kubectl` talks to the Kubernetes API server and works identically on any Kubernetes cluster, anywhere.
- Syntax is always `kubectl [command] [type] [name] [flags]`; omitting the name returns everything of that type.
- `eksctl` builds the infrastructure (once, up front); `kubectl` runs your applications (every day).

## FAQ

**Q: What's the practical difference between `eksctl` and `kubectl`?**
A: `eksctl` provisions the AWS infrastructure the cluster runs on (EC2 nodes, VPCs, IAM). `kubectl` manages the Kubernetes resources — Pods, Deployments, Services — running *inside* that already-built cluster.

**Q: What happens if I run `kubectl get pod` with no name?**
A: You get a list of every pod in the active namespace.

**Q: Break down `kubectl describe deploy web-server -n production` — what's the command, type, name, and flags?**
A: Command: `describe`. Type: `deploy` (abbreviation for deployment). Name: `web-server`. Flags: `-n production`.

**Previous:** [← 2. eksctl](02_eksctl.md)
**Next:** [4. AWS CLI Installation & Configuration →](04_AWS_CLI_Installation_and_Configuration.md)
