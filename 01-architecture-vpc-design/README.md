# Demo-01: EKS Architecture & VPC Design

## Overview

Every EKS cluster you create is shaped by two invisible design decisions made
before you click a single button: how AWS runs the Kubernetes control plane on
your behalf, and how your VPC is structured to receive worker nodes. Get these
wrong and you fight networking failures, IP exhaustion, broken load balancers,
and security gaps for the lifetime of the cluster. Get them right and the
cluster behaves predictably from 2 nodes to 200.

This demo is theory and architecture only -- **no AWS resources are created,
no cost is incurred**. It is the foundation every subsequent demo builds on.

**Real-world scenario:** You are joining a team that runs production workloads
on EKS. Before touching anything you need to answer: why are worker nodes in
private subnets? Why are two subnets the minimum? Why did the ALB fail silently
after Ingress was applied? What is the Cross-ENI in my subnet that I did not
create? Why does EKS need an NLB if kubelets use Cross-ENIs? What is EKS Auto
Mode and does it replace Managed Node Groups? This demo answers all of these.

**References used throughout this demo:**
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
  (referred to as `[BPG]` throughout this demo)

**What this demo covers:**
- What EKS is and the exact AWS-managed vs customer-managed boundary
- The two-VPC architecture -- AWS control plane VPC and your customer VPC
- All Kubernetes control plane components and what each does
- All Kubernetes worker node components and how they interface with the control plane
- Cross-ENIs -- what they are, how they work, one per AZ explained
- Why Cross-ENIs and not VPC Peering, Transit Gateway, or PrivateLink
- The NLB vs Cross-ENI question -- two traffic paths, two different audiences
- High availability at component level and cluster level
- All four data plane options including the new EKS Auto Mode (re:Invent 2024)
- VPC design -- CIDR sizing, subnet layout, IP exhaustion planning
- VPC CNI flat networking -- how pods get real VPC IPs
- Subnet tagging -- why missing tags silently break ALB and NLB
- API server endpoint modes -- public, private, public+private
- Security groups in EKS -- which SG controls what

---

## Prerequisites

**Knowledge (no AWS resources needed):**
- Basic AWS: VPC, Subnets, IGW, NAT Gateway, EC2, IAM, ENI
- Basic Kubernetes: control plane vs worker node concepts
- Basic networking: CIDR notation, IP addressing, routing

---

## Lab Objectives

By the end of this demo you will be able to:
1. ✅ Explain every component in both the control plane and worker node
2. ✅ Trace the exact path of a kubectl command from laptop to etcd and back
3. ✅ Explain the two-VPC model and why Cross-ENIs bridge them
4. ✅ Explain why Cross-ENIs are used instead of VPC Peering, TGW, or PrivateLink
5. ✅ Explain what the NLB does vs what Cross-ENIs do -- two separate traffic paths
6. ✅ Describe HA at both component and cluster level
7. ✅ Compare all four data plane options including EKS Auto Mode
8. ✅ Design a correctly sized VPC for EKS from scratch
9. ✅ Apply the three subnet tags required before cluster creation

---

## Part 1: What is Amazon EKS?

Amazon EKS (Elastic Kubernetes Service) is a **managed Kubernetes control plane
service**. AWS runs, patches, scales, and maintains the Kubernetes control plane
so you never touch it. You bring the worker nodes -- AWS manages everything above
them at the platform level.

### The Exact Management Boundary

This boundary matters because it defines what you are responsible for when
something goes wrong. In a self-managed Kubernetes cluster on EC2, everything
from the hardware upward is your problem. With EKS, the line is drawn at the
node — everything below the kubelet is AWS's responsibility.

```
  Self-Managed Kubernetes on EC2          Amazon EKS
  ─────────────────────────────           ─────────────────────────────────
  YOUR full responsibility:               AWS responsibility:
    etcd cluster -- sizing, backups,        etcd (automated backup, multi-AZ
    restores, Raft quorum issues            Raft, auto-recovery)
    API server -- HA, certs, rotation       API servers (min 2, multi-AZ,
    Controller Manager                      cert auto-rotation, SLA-backed)
    Scheduler                               Controller Manager + Scheduler
    kube-proxy on all nodes                 NLB fronting API servers
    CNI plugin management                   Control plane health checks
    Control plane upgrades                  Kubernetes version upgrades
    API server load balancer                  (control plane side only)
    etcd backup and restore                 Cross-ENIs into your VPC
    Kubernetes version upgrades
    All of the above + your workloads       

                                          YOUR responsibility:
                                            Worker nodes (EC2 or Fargate)
                                            Node-level OS/kubelet upgrades
                                            Your workloads and manifests
                                            Networking config (VPC, subnets, security groups)
                                            IAM roles and RBAC
                                            Storage (EBS, EFS)
                                            Add-ons (LB Controller, etc.)
```

### Why EKS Over Self-Managed?

```
  Concern                   Self-Managed           EKS
  ──────────────────────────────────────────────────────────────────
  etcd backups              Your job               AWS automated
  API server HA             You build it           Auto, across 2+ AZs
  etcd quorum management    You manage 3+ nodes    AWS manages 3 nodes
  Control plane upgrades    Manual, risky           Console click or CLI
  Certificate rotation      Manual or cert-manager  AWS automated
  API server SLA            None                   99.95% SLA
  Control plane cost        EC2 + engineering time  $0.10/hour
  Kubernetes conformance    Your certification       AWS certified
  Security patches          Your schedule           AWS schedule
```

### What You Pay For

```
  EKS control plane:
    Standard support (first 14 months):   $0.10/hr per cluster (~$72/month)
    Extended support (next 12 months):    $0.60/hr per cluster (~$432/month)
    No Free Tier -- charged from cluster creation to deletion

  Worker nodes:
    Managed Node Groups / Self-Managed:   Standard EC2 pricing (no extra fee)
    EKS Auto Mode nodes:                  EC2 price + Auto Mode mgmt fee
                                          e.g. m5.large = EC2 + ~$0.01152/hr
    Fargate pods:                         $0.04048/vCPU-hr + $0.004445/GB-hr

  NAT Gateway:
    Per gateway:                          $0.045/hr (~$32/month)
    Data processed:                       $0.045/GB
```
> **Cost discipline for this lab:** EKS charges $0.10/hr from the moment the
> cluster exists regardless of workload. Always delete the cluster between
> sessions. A 3-hour lab costs ~$0.30 control plane + ~$0.13 EC2 + ~$0.14
> NAT = ~$0.60 total.Set an AWS Budgets alert at $10 to catch any forgotten-running resources.


### What the $0.10/hr Control Plane Fee Covers (and What It Does Not)

This is one of the most misunderstood aspects of EKS pricing. The $0.10/hr
covers a specific, bounded set of resources -- everything else is billed
separately under your account.

```
  INCLUDED in $0.10/hr (AWS pays for these, they do not appear in your bill):
  ─────────────────────────────────────────────────────────────────────────────
    EC2 instances running the API servers in the AWS-managed VPC
    EC2 instances running the etcd cluster (3 nodes, 3 AZs)
    EC2 instances running Controller Manager and Scheduler
    EBS volumes for etcd data storage and WAL (write-ahead log)
    NLB fronting the API servers (the EKS endpoint NLB)
    NAT Gateways inside the AWS-managed VPC (control plane outbound)
    Cross-ENIs (Elastic Network Interfaces in your subnets) -- free
    Automated etcd backups and snapshot storage
    Security patches for all control plane components
    HA infrastructure (multi-AZ, auto-replacement of failed components)
    AWS Technical Support for control plane issues (all support tiers)

  NOT INCLUDED (appear in your AWS bill separately):
  ─────────────────────────────────────────────────────────────────────────────
    EC2 instances for your worker nodes (Managed Node Group or self-managed)
    NAT Gateways in YOUR VPC (you pay $0.045/hr per gateway)
    EBS volumes attached to your worker nodes (root volumes, PVCs)
    ALB / NLB provisioned by AWS Load Balancer Controller for your apps
    Data transfer through your NAT Gateways ($0.045/GB processed)
    Cross-AZ data transfer between your worker nodes and pods ($0.01/GB)
    Internet data transfer from your VPC (egress to internet)
    CloudWatch logs (control plane logs if enabled -- $0.50/GB ingestion)
    ECR storage and image pull data transfer
    EKS add-ons compute (CoreDNS, VPC CNI, kube-proxy run on your nodes
      using your node's CPU and memory -- they consume your EC2 capacity)
    EFS, EBS CSI driver storage
    Route 53, Secrets Manager, and all other AWS services you use
```

**The invisible NAT Gateway distinction:**

```
  AWS-managed VPC (control plane):
    Has its own NAT Gateways for control plane outbound connectivity
    (e.g. pulling container images for API server, calling AWS APIs)
    These NAT Gateways are INSIDE the $0.10/hr -- you pay nothing extra

  YOUR VPC:
    NAT Gateways are created by YOU for your worker node outbound access
    (pulling container images from ECR/Docker Hub, calling AWS APIs from pods)
    These appear in your bill as Amazon VPC NAT Gateway charges
    $0.045/hr per gateway + $0.045/GB data processed
    2 NAT Gateways (one per AZ for HA) = $0.09/hr = ~$65/month just for NAT

  Why you need NAT Gateways in YOUR VPC:
    Worker nodes are in private subnets (no public IP)
    Nodes need outbound internet to:
      Pull container images from Docker Hub, Quay, ECR public
      Call AWS APIs (EC2, ECR, CloudWatch, Secrets Manager, etc.)
      Download OS patches
    NAT Gateways translate the node's private IP to a public IP for egress
    Inbound connections to nodes are NOT allowed through NAT (only egress)
```

### When Do Control Plane Components Use the AWS-Managed NAT Gateways?

The NAT Gateways inside the AWS-managed VPC are used by control plane
components when they need to make outbound connections that exit the
control plane VPC. There are specific, bounded scenarios when this happens.

```
  Control plane component outbound traffic (uses AWS-managed NAT GW):

  1. API Server --> AWS APIs (ECR, S3, STS)
     The API server pulls its own admission webhook container images
     from ECR when webhooks are configured.
     Also calls AWS STS to validate IAM authentication tokens.

  2. Controller Manager --> AWS APIs
     The Node controller calls EC2 API to verify node existence.
     The Service controller calls Elastic Load Balancing API to
     provision/update load balancers when you create a Service of
     type LoadBalancer. This is the AWS cloud-controller-manager.

  3. etcd --> S3 (backups)
     EKS automated etcd backups are written to S3 buckets in the
     AWS-managed account. This traffic exits the control plane VPC
     through the managed NAT Gateway to reach S3.

  4. Control plane --> AWS Health (status reporting)
     EKS reports cluster health events to the AWS Health service.

  Does this cost you anything?
    No. The NAT Gateways in the AWS-managed VPC are internal AWS
    infrastructure. Their hourly cost and data processing charges
    are absorbed into your $0.10/hr control plane fee.
    You never see a line item for control plane NAT Gateway charges.
```

### Kubernetes Version Support -- Standard vs Extended

EKS supports every Kubernetes minor version for a total of **26 months**.
That 26 months is split into two distinct phases with different pricing,
different patch coverage, and different auto-upgrade behavior.

```
  PHASE 1 -- STANDARD SUPPORT
  ────────────────────────────────────────────────────────────────────
  Duration:   14 months from the date EKS releases that K8s version
  Price:      $0.10/hr per cluster (the normal EKS fee)
  What AWS patches:
    - Kubernetes control plane (API server, etcd, controllers)
    - VPC CNI, kube-proxy, CoreDNS add-ons
    - EKS-optimized AMIs (Amazon Linux 2023, Bottlerocket, Windows)
    - EKS Fargate nodes
  Why 14 months:
    The upstream Kubernetes project supports each minor version for
    approximately 14 months before stopping community patches.
    EKS standard support matches this window exactly.

  PHASE 2 -- EXTENDED SUPPORT (automatic -- no action needed to enter it)
  ────────────────────────────────────────────────────────────────────
  Duration:   12 months immediately after standard support ends
  Price:      $0.60/hr per cluster  (6x the standard rate)
  What AWS patches (reduced scope vs standard):
    - Kubernetes control plane  (yes -- AWS continues control plane patches)
    - VPC CNI, kube-proxy, CoreDNS  (yes -- critical patches only)
    - EKS-optimized AMIs  (yes -- Amazon Linux 2023, Bottlerocket, Windows)
    - EKS Fargate nodes  (yes)
    NOT covered:
    - EBS CSI driver, EFS CSI driver  (no extended support patches)
    - ADOT, GuardDuty agent  (no extended support patches)
    - AWS Marketplace add-ons  (no extended support patches)
  Auto-upgrade behavior:
    At the end of extended support (month 26), AWS auto-upgrades your
    CONTROL PLANE to the next oldest supported version.
    Node groups and self-managed nodes are NOT auto-upgraded.
    You must upgrade nodes manually after the control plane upgrades.

  AFTER EXTENDED SUPPORT ENDS (month 26+)
  ────────────────────────────────────────────────────────────────────
  You cannot create new clusters on an expired version.
  Existing control planes are auto-upgraded by AWS to the earliest
  currently supported version (gradual rollout, not immediate).
  After the control plane upgrade: update add-ons and nodes manually.
```

**The full 26-month timeline on one example (K8s 1.31):**

```
  Sep 2024   K8s 1.31 released on EKS         --> $0.10/hr starts
             Standard support begins
             AWS Health Dashboard: 60-day notice sent ~12 months in

  Nov 2025   Standard support ends (14 months later)
             Extended support begins AUTOMATICALLY
             Price jumps to $0.60/hr           --> 6x cost begins here

  Nov 2026   Extended support ends (12 more months)
             Cannot create new clusters on 1.31
             Existing clusters: control plane auto-upgraded by AWS
             Your node groups: remain on 1.31 -- you must upgrade manually
```

**The cost math if you never upgrade:**

```
  Cluster on K8s 1.31 from Sep 2024 to Nov 2026 (26 months):

  Months 1-14  (standard):   $0.10/hr x 730hr x 14 = $1,022
  Months 15-26 (extended):   $0.60/hr x 730hr x 12 = $5,256
  Total:                     $6,278 in control plane fees alone
  Average effective rate:    $0.33/hr over 26 months

  If you upgrade at month 13 (one month before standard ends):
  Months 1-14 on 1.31:       $0.10/hr x 730hr x 14 = $1,022
  Upgrade to 1.32, restart 14-month standard window: $0.10/hr continues
  Total for same period:     $1,022 -- no extended support charges
```

**How to check your cluster's support status:**

```
  AWS Console:
    EKS --> Clusters --> your cluster --> Overview tab
    Look for "Kubernetes version" and "Version status" fields
    Status will show: Standard support  or  Extended support

  AWS CLI:
    aws eks describe-cluster-versions --region us-east-1
    # Shows all versions with releaseDate, endOfStandardSupportDate,
    # endOfExtendedSupportDate, and status field

  Check your specific cluster:
    aws eks describe-cluster --name my-cluster \
      --query 'cluster.{Version:version,Status:status}'

  AWS Health Dashboard:
    Sends notifications ~60 days before standard support ends
    Appears in: AWS Console --> Health --> Health events
    Also sent to the account email address
```

**Upgrade policy setting (controls auto-upgrade behavior):**

```
  Default for all new and existing clusters: EXTENDED
  (cluster automatically enters extended support, does not auto-upgrade
  the control plane at end of standard support)

  Change to STANDARD if you want the cluster to auto-upgrade the
  control plane rather than enter extended support:
    aws eks update-cluster-config \
      --name my-cluster \
      --upgrade-policy supportType=STANDARD

  With STANDARD policy:
    At end of standard support: control plane is auto-upgraded to next version
    No extended support charges -- but you must be ready for the upgrade
    Suitable for non-production clusters where unexpected upgrades are OK

  With EXTENDED policy (default):
    At end of standard support: enters extended support at $0.60/hr
    12 months later: control plane auto-upgraded, extended support ends
    Suitable for production clusters where you control the upgrade window
    but MUST budget for the $0.60/hr rate during the extended window

  For this lab series: use EXTENDED policy (default)
  Delete the cluster between sessions -- version lifecycle does not matter
  for short-lived lab clusters
```

> **`[BPG]` Upgrades:** "We recommend you proactively update your clusters
> to use the latest available version. A minor version is under standard
> support for the first 14 months after it's released in Amazon EKS."

> **Extended support cost trap `[BPG]`:** If you create a cluster and never
> upgrade the Kubernetes version, after 14 months the fee jumps from $0.10/hr
> to $0.60/hr -- a 6x increase. Always stay within standard support by
> upgrading on schedule. This is one of the most common sources of unexpected
> EKS spend in production.

---

## Part 1b: Kubernetes Versions, EKS Versions, and Their Alignment

Understanding the relationship between upstream Kubernetes versions and EKS
versions is essential for planning upgrades and avoiding surprise charges.

### Upstream Kubernetes Release Cadence

```
  The Kubernetes project releases a new MINOR version approximately
  every 4 months (3 minor versions per year).

  Versioning format:  MAJOR.MINOR.PATCH
    1.35.2 = Major 1, Minor 35, Patch 2

  Minor version example:  1.31, 1.32, 1.33, 1.34, 1.35
    Each minor version adds new features and API changes.
    Upgrading minor versions requires planning (API deprecations, etc.)
    Kubernetes upstream supports 3 minor versions at any time.

  Patch version example:  1.31.0, 1.31.1, 1.31.2, 1.31.3
    Bug fixes and security patches only.
    Backwards compatible -- upgrade freely within a minor version.
    Kubernetes releases patches approximately every 2-4 weeks.
    EKS applies patch releases to your control plane automatically
    (you do not need to trigger patch upgrades -- AWS handles them).
```

### EKS Platform Versions -- The AWS Layer on Top

EKS adds an additional versioning concept called the **EKS platform version**.
This represents the capabilities and configuration of the AWS-managed control
plane layer, independently of the Kubernetes minor version.

```
  EKS platform version format:  eks.N  (e.g. eks.21, eks.22)

  Full EKS version identifier:  K8s_version + platform_version
  Example:  1.31 / eks.21

  What an EKS platform version increment means:
    AWS-managed configuration changes (API server flags enabled/disabled)
    Control plane infrastructure updates (OS patches on control plane nodes)
    New EKS-specific features enabled (e.g. new admission controllers)
    Security patches applied to the control plane infrastructure

  Platform version upgrades happen automatically.
  You do not trigger them -- AWS applies them as a rolling update.
  Your cluster goes from eks.21 to eks.22 transparently.
  The Kubernetes minor version does NOT change during a platform upgrade.

  Check your current platform version:
    aws eks describe-cluster --name my-cluster \
      --query 'cluster.{K8sVersion:version,Platform:platformVersion}'
    # Output: {"K8sVersion": "1.31", "Platform": "eks.21"}
```

### EKS vs Upstream Kubernetes -- Key Differences

```
  ┌────────────────────────┬──────────────────────────┬──────────────────────────┐
  │ Aspect                 │ Upstream Kubernetes      │ Amazon EKS               │
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Version support        │ 3 minor versions         │ 4+ minor versions        │
  │ window                 │ (~12-14 months)          │ (26 months per version)  │
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Release lag            │ Source of truth          │ Typically 1-3 months     │
  │                        │                          │ after upstream GA        │
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Security patches       │ Upstream stops at end    │ EKS backports patches    │
  │ after EOL              │ of 14-month window       │ for 12 more months       │
  │                        │                          │ (extended support)       │
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Patch upgrades         │ Manual                   │ Auto-applied by EKS      │
  │ (x.y.Z)                │                          │ (you don't trigger)      │
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Minor upgrades         │ Manual                   │ Manual (you trigger)     │
  │ (x.Y.z)                │                          │ Auto only after 26 months│
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Kubernetes conformance │ Source                   │ AWS certified conformant │
  │                        │                          │ All GA features supported│
  ├────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Feature flags          │ Standard upstream flags  │ EKS-specific flags via   │
  │                        │                          │ platform version layer   │
  └────────────────────────┴──────────────────────────┴──────────────────────────┘
```

### Version Skew Policy -- Nodes vs Control Plane

This matters critically during upgrades. Kubernetes defines which version
combinations are valid between the control plane and worker nodes.

```
  Official Kubernetes version skew rules:
    kube-apiserver (control plane):    must be upgraded FIRST
    kubelet (worker nodes):            can be up to 3 minor versions OLDER
                                       than kube-apiserver
    kube-proxy:                        can be up to 3 minor versions OLDER
    kube-controller-manager:           must match or be 1 minor older than
                                       kube-apiserver
    kube-scheduler:                    must match or be 1 minor older than
                                       kube-apiserver

  Example -- valid configuration:
    API server:    1.35  (control plane)
    kubelet:       1.33  (3 minor versions behind -- allowed)
    kube-proxy:    1.33  (3 minor versions behind -- allowed)

  Example -- INVALID configuration:
    API server:    1.35
    kubelet:       1.32  (4 minor versions behind -- NOT allowed)

  EKS upgrade order (ALWAYS in this sequence):
    1. Upgrade EKS control plane (API server, etcd, controllers)
    2. Upgrade EKS add-ons (CoreDNS, VPC CNI, kube-proxy)
    3. Upgrade worker node groups (Managed Node Groups or Self-Managed)
    4. Verify node kubelet versions with: kubectl get nodes

  Skipping steps or reversing order can break the cluster.
  Never upgrade nodes BEFORE upgrading the control plane.
```

### Currently Supported Versions (as of May 2026)

```
  Check the live calendar: https://endoflife.date/amazon-eks
  Or: aws eks describe-cluster-versions --region us-east-1

  General guidance (versions change every ~4 months):
    Latest standard support: typically the 4 most recent minor versions
    Extended support:        the next 1-2 older minor versions
    Expired (auto-upgraded): anything older than 26 months

  Always check before creating a cluster -- create on the LATEST
  standard support version to maximize your 14-month window.
```

---

## Part 2: The Two-VPC Architecture

This is the most important mental model for EKS. Every cluster involves
**two completely separate VPCs** — one owned and operated by AWS, one owned
and operated by you. They serve entirely different purposes and are connected
by a specific, carefully designed mechanism.

```
+-----------------------------------------------------------------------+
|  AWS-MANAGED VPC  (runs in AWS's own account -- invisible to you)     |
|                                                                       |
|  You cannot see this VPC in your AWS console.                         |
|  You cannot SSH into these servers.                                   |
|  AWS manages everything inside it.                                    |
|                                                                       |
|                                                                       |
|  +-----------------------------+  +------------------------------+    |
|  |  AZ: us-east-1a             |  |  AZ: us-east-1b              |    |
|  |  +-------------------------+|  |  +-------------------------+ |    |
|  |  |  kube-apiserver          ||  |  |  kube-apiserver        | |    |
|  |  |  (API Server replica 1) ||  |  |  (API Server replica 2) | |    |
|  |  +-------------------------+|  |  +-------------------------+ |    |
|  |  +-------------------------+|  |  +-------------------------+ |    |
|  |  |  etcd node (Raft mbr 1) ||  |  |  etcd node (Raft mbr 2) | |    |
|  |  +-------------------------+|  |  +-------------------------+ |    |
|  |  +-------------------------+|  |                            | |    |
|  |  |  kube-controller-manager||  |  +-------------------------+ |    |
|  |  |  (runs all controllers) ||  |  |  kube-scheduler         | |    |
|  |  +-------------------------+|  |  +-------------------------+ |    |
|  +-----------------------------+  +------------------------------+    |
|                                                                       |
|  AZ: us-east-1c:  [ etcd node (Raft member 3 -- quorum node) ]        |
|                                                                       |
|  +------------------------------------------------------------------+ |
|  |  NLB  (Network Load Balancer -- fronts ALL API server replicas)  | |
|  |  DNS: https://XXXXX.gr7.us-east-1.eks.amazonaws.com              | |
|  |  WHO uses this: kubectl / CI-CD / AWS Console / eksctl           | |
|  |  WHO does NOT use this: kubelets (they use Cross-ENIs instead)   | |
|  +------------------------------------------------------------------+ |
|                                                                       |
+----------------------------+------------------+-----------------------+
                             |   Cross-ENIs     |
                             |  (one per AZ)    |
                             |  AWS-managed     |
                             |  in YOUR subnets |
+----------------------------+------------------+-----------------------+
|  YOUR VPC  10.0.0.0/16  (visible and managed in your AWS account)     |
|                                                                       |
|  WORKER NODE COMPONENTS:                                              |
|                                                                       |
|  +------------------------------------------------------------------+ |
|  |  PRIVATE SUBNETS                                                 | |
|  |  +----------------------------+  +----------------------------+  | |
|  |  | priv-1a  10.0.10.0/22      |  | priv-1b  10.0.11.0/22      |  | |
|  |  |                            |  |                            |  | |
|  |  | EC2 Worker Node            |  | EC2 Worker Node            |  | |
|  |  |  +----------------------+  |  |  +----------------------+  |  | |
|  |  |  | kubelet              |  |  |  | kubelet              |  |  | |
|  |  |  | kube-proxy           |  |  |  | kube-proxy           |  |  | |
|  |  |  | VPC CNI (aws-node)   |  |  |  | VPC CNI (aws-node)   |  |  | |
|  |  |  | container runtime    |  |  |  | container runtime    |  |  | |
|  |  |  +----------------------+  |  |  +----------------------+  |  | |
|  |  |  Pods (real VPC IPs)       |  |  Pods (real VPC IPs)       |  | |
|  |  |                            |  |                            |  | |
|  |  |  Cross-ENI (from AWS VPC)  |  |  Cross-ENI (from AWS VPC)  |  | |
|  |  +----------------------------+  +----------------------------+  | |
|  +------------------------------------------------------------------+ |
|                                                                       |
|  +------------------------------------------------------------------+ |
|  |  PUBLIC SUBNETS                                                  | |
|  |  Internet-facing ALB/NLB / NAT Gateways / Bastion (optional)     | |
|  +------------------------------------------------------------------+ |
|                                                                       |
|  Internet Gateway                                                     |
+-----------------------------------------------------------------------+
                             ^
                          Internet
                   (kubectl / users / CI-CD)
```

> **`[BPG]` Networking:** "Operating an EKS cluster requires knowledge of
> both AWS VPC networking and Kubernetes networking. We recommend you
> understand the EKS control plane communication mechanisms before you
> start designing your VPC or deploying clusters into existing VPCs."

### What Each VPC Contains and Why

**AWS-Managed Control Plane VPC:**

- Exists in an AWS-owned AWS account — it does not appear in your console
- Contains the Kubernetes API servers (minimum 2, in distinct AZs)
- Contains the etcd cluster (exactly 3 nodes, across 3 distinct AZs)
- Contains a Network Load Balancer that fronts the API servers — this NLB's
  DNS name is the endpoint your kubeconfig and kubectl point to
- Contains NAT Gateways for control plane outbound connectivity
- All control plane resources run in private subnets inside this VPC
- You pay for the control plane ($0.10/hour) but cannot see or touch it

**Your Customer VPC (visible in your AWS account):**

- Where your EC2 worker nodes live
- Where your pods run and receive IP addresses
- Where load balancers are provisioned for your applications
- Where all your application traffic flows
- You design, create, tag, and own this VPC entirely

---

## Part 3: Control Plane Components -- What Each Does

All control plane components run inside the AWS-managed VPC. You never SSH
into them, never patch them, and never restart them. AWS manages all of this
under the $0.10/hr fee.

```
  +------------------------------------------------------------------+
  |  kube-apiserver  (THE central hub -- every component talks here) |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Receives and validates ALL API requests (kubectl, CI-CD,    |
  |      other control plane components, kubelets)                   |
  |    - Authenticates via: IAM (aws-auth), OIDC, client certs       |
  |    - Authorizes via: Kubernetes RBAC (Roles, ClusterRoles)       |
  |    - Validates resource schemas against API definitions          |
  |    - Writes accepted state to etcd                               |
  |    - Serves WATCH streams to controllers and kubelets            |
  |    - Scales horizontally -- EKS runs min 2 replicas, multi-AZ    |
  |    - Sits behind an NLB for external callers                     |
  |    - Connects to worker nodes via Cross-ENIs for kubelet push    |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  etcd  (THE only stateful component -- the cluster brain)        |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Stores ALL cluster state: Deployments, Pods, Services,      |
  |      ConfigMaps, Secrets, RBAC rules -- everything               |
  |    - Distributed key-value store using the Raft consensus        |
  |      algorithm across 3 nodes in 3 AZs                           |
  |    - API server is the ONLY component that talks to etcd         |
  |      directly -- all others go through the API server            |
  |    - If etcd loses quorum: API server becomes read-only          |
  |      Existing pods keep running. No new scheduling possible.     |
  |    - EKS automates backup, restoration, and recovery of etcd     |
  |    - You cannot access etcd directly -- AWS manages it           |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  kube-controller-manager  (runs all built-in control loops)      |
  |                                                                  |
  |  A single binary that embeds 30+ controllers, each running its   |
  |  own reconciliation loop. Key controllers:                       |
  |                                                                  |
  |    Deployment controller:                                        |
  |      Watches Deployment objects. Creates/updates ReplicaSets.    |
  |      Ensures desired replica count matches actual.               |
  |                                                                  |
  |    ReplicaSet controller:                                        |
  |      Watches ReplicaSet objects. Creates/deletes Pods to match   |
  |      the desired replica count.                                  |
  |                                                                  |
  |    Node controller:                                              |
  |      Watches node heartbeats from kubelets (via API server).     |
  |      Marks nodes NotReady after 40s of missed heartbeats.        |
  |      Evicts pods from NotReady nodes after 5 minutes.            |
  |                                                                  |
  |    Service Account controller:                                   |
  |      Creates default ServiceAccounts in new namespaces.          |
  |                                                                  |
  |    PersistentVolume controller:                                  |
  |      Binds PVCs to PVs. Triggers dynamic provisioning.           |
  |                                                                  |
  |    EndpointSlice controller:                                     |
  |      Populates EndpointSlice objects as pods come and go.        |
  |      kube-proxy watches these to update iptables rules.          |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  kube-scheduler  (decides which pod runs on which node)          |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Watches for Pods with no nodeName (unscheduled)             |
  |    - Runs a two-phase algorithm for every unscheduled pod:       |
  |                                                                  |
  |    Phase 1 -- Filtering (find feasible nodes):                   |
  |      Remove nodes where pod does not fit:                        |
  |        Insufficient CPU or memory requests                       |
  |        NodeSelector / nodeAffinity mismatch                      |
  |        Taint with no matching toleration                         |
  |        Volume affinity or zone mismatch                          |
  |        Max pod count exceeded (ENI limit on t3.small = 11)       |
  |                                                                  |
  |    Phase 2 -- Scoring (rank feasible nodes):                     |
  |      Score remaining nodes on:                                   |
  |        Spread (topology spread constraints)                      |
  |        Bin-packing efficiency                                    |
  |        Affinity preferences                                      |
  |        Node resource balance                                     |
  |                                                                  |
  |    - Writes chosen nodeName to Pod spec in etcd (via API server) |
  |    - kubelet on that node sees the assignment and starts the pod |
  +------------------------------------------------------------------+
```

### How These Components Interface

```
  kubectl apply -f deployment.yaml
         |
         | HTTPS to NLB endpoint
         v
  kube-apiserver
    |-- Authenticates (IAM / OIDC / cert)
    |-- Authorizes (RBAC)
    |-- Validates (schema check)
    |-- Writes Deployment to etcd
         |
         | etcd change triggers WATCH notification
         v
  kube-controller-manager (Deployment controller)
    |-- Sees new Deployment, creates ReplicaSet
    |-- ReplicaSet controller sees RS, creates N Pod specs
    |-- Writes Pod specs to etcd (no nodeName yet)
         |
         | etcd change triggers WATCH notification
         v
  kube-scheduler
    |-- Sees unscheduled Pods
    |-- Runs filtering + scoring
    |-- Writes nodeName to each Pod in etcd
         |
         | etcd change triggers WATCH notification on worker node
         v
  kubelet (on selected worker node)
    |-- Sees pod assigned to its node
    |-- Calls container runtime (containerd) to pull image + start container
    |-- Reports pod status back to API server every few seconds
    |-- API server writes status to etcd
```

---

## Part 4: Worker Node Components -- What Each Does

These components run on every EC2 worker node inside your VPC. Unlike the
control plane, you can see and access them.

```
  +------------------------------------------------------------------+
  |  kubelet  (the node agent -- the most important worker component)|
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Registers the node with the API server at startup           |
  |      Reports: node name, IP, capacity (CPU/mem/pods), labels     |
  |    - Sends heartbeats to the API server every 10 seconds         |
  |      (default nodeStatusUpdateFrequency)                         |
  |      If heartbeat stops: node marked Unknown after 40s,          |
  |      pods evicted after 5 minutes (nodemonitor grace period)     |
  |    - Watches for Pods assigned to this node via API server WATCH |
  |    - Pulls container images via the container runtime interface  |
  |      (CRI -- containerd on modern EKS AMIs, previously Docker)   |
  |    - Starts and stops containers via containerd                  |
  |    - Runs liveness and readiness probes                          |
  |    - Mounts ConfigMaps, Secrets, and PersistentVolumes into pods  |
  |    - Reports pod status (Running, Failed, Completed) to API server|
  |    - Enforces resource limits (CPU throttling, OOM kills)         |
  |                                                                  |
  |  How kubelet reaches the API server:                             |
  |    Public+Private mode: HTTPS to Cross-ENI IP (in-VPC path)      |
  |    Public Only mode: HTTPS out through NAT GW to NLB endpoint    |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  kube-proxy  (Service networking -- programs iptables/IPVS)      |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Watches Service and EndpointSlice objects in the API server |
  |    - When a Service is created (e.g. ClusterIP 10.100.1.50:80):  |
  |      Programs iptables DNAT rules on every node so that traffic  |
  |      to 10.100.1.50:80 is load-balanced to backing pod IPs       |
  |    - When a pod is added/removed: updates iptables rules         |
  |    - Makes Service ClusterIPs reachable from any pod on any node |
  |    - Does NOT handle pod-to-pod routing -- that is VPC CNI's job |
  |                                                                  |
  |  Modes: iptables (default on EKS), IPVS (better at large scale)  |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  VPC CNI  (aws-node DaemonSet -- pod IP assignment)              |
  |                                                                  |
  |  Two sub-components:                                             |
  |    CNI binary:  invoked by kubelet when a pod is created/deleted |
  |                 programs veth pairs and routes for the pod       |
  |    ipamd:       long-running daemon managing the ENI warm pool   |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Attaches secondary ENIs to the node via EC2 API             |
  |    - Assigns secondary IPs to ENIs via EC2 API                   |
  |    - Maintains a warm pool of pre-allocated IPs                  |
  |    - Assigns an IP from the warm pool to each new pod            |
  |    - Programs routes so traffic to pod IP reaches the pod veth   |
  |    - Releases IPs back to pool when pods terminate               |
  |    - Requires IAM permissions (IRSA or node role) to call EC2 API|
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  containerd  (container runtime -- runs the actual containers)   |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Pulls container images from ECR / Docker Hub / other OCI    |
  |    - Creates and starts container processes                      |
  |    - Manages container lifecycle (start, stop, kill)             |
  |    - Manages container filesystem layers (overlay2)              |
  |    - Communicates with kubelet via the CRI (gRPC) interface      |
  |    - EKS AMIs use containerd since Kubernetes 1.24               |
  |      (Docker was the runtime before 1.24 -- now deprecated)      |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  CoreDNS  (cluster DNS -- deployed as 2-replica Deployment)      |
  |                                                                  |
  |  NOT a DaemonSet -- runs as a Deployment with 2 replicas,        |
  |  typically placed on different nodes for HA.                     |
  |                                                                  |
  |  Responsibilities:                                               |
  |    - Resolves Kubernetes service names to ClusterIPs             |
  |      e.g. myapp.default.svc.cluster.local -> 10.100.1.50         |
  |    - All pods have /etc/resolv.conf pointing to CoreDNS ClusterIP|
  |    - Forwards external names (google.com) to VPC DNS resolver    |
  |    - VPC DNS resolver at 169.254.169.253 (or VPC base + 2)       |
  |    - Caches responses to reduce DNS query volume                 |
  +------------------------------------------------------------------+
```

### How Worker Components Interface with the Control Plane

```
  Worker Node (running)                   Control Plane (AWS VPC)
  ──────────────────────                  ──────────────────────────────
  kubelet
    │
    │ POST /api/v1/nodes          ──────► kube-apiserver (register)
    │                                      writes node to etcd
    │ HTTPS heartbeat every 10s   ──────► kube-apiserver
    │                                      kube-controller-manager
    │                                        (node controller watches)
    │ WATCH /api/v1/pods          ──────► kube-apiserver WATCH stream
    │   (filtered to this node)             (long-lived HTTP connection)
    │ <── Pod spec arrives ────────────── kube-scheduler wrote nodeName
    │
    │ calls containerd to start pod
    │
    │ POST /api/v1/pods/.../status ─────► kube-apiserver (status update)
    │                                      writes to etcd

  kube-proxy
    │ WATCH /api/v1/services      ──────► kube-apiserver WATCH stream
    │ WATCH /discovery/endpointslices ──► kube-apiserver WATCH stream
    │ <── EndpointSlice update ────────── EndpointSlice controller
    │ programs local iptables rules

  VPC CNI (aws-node ipamd)
    │ WATCH /api/v1/pods          ──────► kube-apiserver WATCH stream
    │ calls EC2 API to manage ENIs/IPs    (not via API server)
    │ CNI binary invoked by kubelet
    │   when pod is created/deleted
```

### etcd — The Cluster Brain

etcd is the only stateful component in Kubernetes. Every object you create
(Deployments, Services, Pods, ConfigMaps, Secrets) is stored in etcd.
If etcd loses quorum, the API server becomes read-only. No new pods can be
scheduled, no changes can be made — existing workloads keep running but
the cluster is effectively frozen.

```
  Raft consensus requires quorum: floor(N/2) + 1 nodes must agree

  3 etcd nodes (EKS default):
    Quorum required: 2 nodes
    Can tolerate: 1 node failure
    AZ failure impact: 1 etcd node lost -- quorum maintained (2 of 3 remain)
    All 3 nodes lost: quorum lost -- API server read-only

  Why AWS uses 3 nodes and 3 AZs:
    Any single AZ failure loses 1 etcd node
    2 remaining nodes in 2 AZs maintain quorum
    Control plane remains fully functional through a single AZ outage
```

### What Happens When a Node Cannot Reach the API Server

Understanding this failure path is critical for debugging cluster issues.

```
  Normal operation:
    kubelet sends heartbeat to API server every 10 seconds
    API server updates node's .status.conditions = Ready

  API server unreachable (Cross-ENI issue, SG block, DNS failure):
    kubelet cannot send heartbeat
    After 40 seconds: API server marks node as "Unknown"
    After 5 minutes: Node controller starts evicting pods from the node
    kubectl get nodes  shows:  STATUS = NotReady

  Common causes of API server unreachability:
    1. Cross-ENI security group was modified (deleted cluster SG rule)
    2. VPC DNS not enabled (enableDnsHostnames or enableDnsSupport = false)
    3. API endpoint changed and kubeconfig not updated
    4. Private endpoint mode with no VPN/Direct Connect for external access
    5. Node's subnet has no route to the Cross-ENI's subnet
       (this does not apply within the same subnet but relevant cross-subnet)
```

**Key component reference:**

| Component          | Purpose                                            | Location                   |
|--------------------|----------------------------------------------------|----------------------------|
| API Server         | All cluster state reads and writes, auth/authz     | AWS VPC (min 2, multi-AZ)  |
| etcd               | Cluster state storage (the only stateful piece)    | AWS VPC (3 nodes, 3 AZs)   |
| Controller Manager | Reconciliation loops for all built-in resources    | AWS VPC                    |
| Scheduler          | Assigns unscheduled pods to nodes                  | AWS VPC                    |
| NLB                | Fronts API servers, provides stable DNS endpoint   | AWS VPC                    |
| kubelet            | Runs on every node, manages pod lifecycle          | Your EC2 nodes             |
| kube-proxy         | Programs iptables for Service networking           | Your EC2 nodes (DaemonSet) |
| VPC CNI (aws-node) | Assigns VPC IPs to pods, manages ENI pool          | Your EC2 nodes (DaemonSet) |
| CoreDNS            | In-cluster DNS for service name resolution         | Your nodes (Deployment)    |

---

## Part 5: Cross-ENIs -- The Bridge Between the Two VPCs

### What a Cross-ENI Is

When you create an EKS cluster, EKS provisions **Elastic Network Interfaces
(ENIs) directly inside your private subnets**. These are called Cross-ENIs
(also called X-ENIs or Cross-Account ENIs in official AWS documentation).

A Cross-ENI:
- Is owned and managed by AWS EKS (you cannot delete or modify it)
- Physically lives inside your subnet (has an IP from your CIDR, e.g. 10.0.10.4)
- Is attached to the API server instance on the AWS-managed side
- Appears in your EC2 console under Network Interfaces
  (description contains "Amazon EKS" -- never delete these)

```
  AWS-managed VPC
  +----------------------------------------+
  |  API Server EC2 instance               |
  |  eth0: 172.31.x.x  (internal AWS IP)   |
  |  eth1: 10.0.10.4   <── Cross-ENI       |
  |         |                              |
  |         | This IP 10.0.10.4 comes      |
  |         | from YOUR subnet CIDR        |
  +---------|------------------------------+
            | (not a tunnel -- just eth1)
  Your VPC -- Private Subnet 10.0.10.0/22
  +---------+------------------------------+
  |  Cross-ENI  IP: 10.0.10.4              |
  |  Owned by EKS / lives in your subnet   |
  |  Visible in EC2 --> Network Interfaces |
  |  Never delete ENIs with "Amazon EKS"   |
  |  in their description                  |
  +----------------------------------------+
            |  local subnet routing
  +--------------------------------------------+
  |  Worker Node   IP: 10.0.10.15              |
  |  +---------------------------------------+ | 
  |  | kubelet sends heartbeats and receives | |
  |  | pod specs via the Cross-ENI IP        | |
  |  | (10.0.10.4) -- it looks like a local  | |
  |  | VPC address because it IS one         | |
  |  +---------------------------------------+ |
  +--------------------------------------------+
```
From the worker node's perspective, the API server is just another IP
address inside the same VPC subnet. There is no tunnel, no gateway, no
special routing — just a local network hop to 10.0.10.4 which happens
to be the Cross-ENI bridging over to the AWS-managed control plane.


### What a Cross-ENI Is -- The Physical Reality

This is confusing because it seems paradoxical: an ENI that physically lives
in your subnet but is attached to a server in a completely different VPC.
Understanding how this is technically possible requires understanding what
an ENI actually is at the AWS infrastructure level.

**First: what is an ENI technically?**

```
  An Elastic Network Interface (ENI) is a virtual network card.
  In AWS's physical infrastructure, an ENI is a software construct --
  a record in AWS's network control plane that says:

    "This network interface has IP address 10.0.10.4,
     is in subnet subnet-0abc123 (10.0.10.0/22),
     is in VPC vpc-0xyz789 (10.0.0.0/16),
     and can have MAC address aa:bb:cc:dd:ee:ff"

  Crucially: an ENI is not physically located inside the EC2 instance.
  It is a virtual construct maintained by the AWS Nitro hypervisor.
  The hypervisor routes network packets to/from the ENI based on its
  subnet's routing rules -- regardless of which physical host the
  attached EC2 instance runs on.
```

**The Cross-ENI architecture:**

```
  AWS Physical Infrastructure (simplified):

  ┌──────────────────────────────────────────────────────────────────┐
  │  AWS Nitro Hypervisor Layer  (controls all VPC networking)       │
  │                                                                  │
  │  Routing rule:                                                   │
  │    "Any packet destined for 10.0.10.4 (Cross-ENI IP)            │
  │     forward it to --> API Server instance in AWS-managed VPC"    │
  │                                                                  │
  │  Routing rule:                                                   │
  │    "Any packet FROM the API Server with source 10.0.10.4         │
  │     treat it as originating from subnet 10.0.10.0/22"            │
  └──────────────────────────────────────────────────────────────────┘

  Your VPC routing table:
    Destination 10.0.10.0/22 --> local  (subnet local routing)
    10.0.10.4 is IN this local range --> treated as local

  Your Worker Node (10.0.10.15):
    "I need to reach 10.0.10.4"
    Looks up routing table --> local subnet
    Sends packet directly -- no gateway needed
    Hypervisor intercepts at destination
    Delivers packet to API Server in AWS-managed VPC

  API Server (in AWS-managed VPC):
    Receives packet from 10.0.10.15 (worker node)
    Sends response with SOURCE IP = 10.0.10.4 (the Cross-ENI IP)
    Hypervisor routes response back to 10.0.10.15

  From the worker node's perspective:
    10.0.10.4 responded -- it IS a local subnet neighbor
    The worker node has no knowledge that the API server is in a
    different VPC or even on a different physical host in a different AZ
```

**The key insight -- ENIs are software, not hardware:**

```
  Traditional physical networking:
    A network card (NIC) is physically inside the server.
    The IP address is bound to that physical card.
    The card and server are in the same physical location.

  AWS virtual networking (ENI):
    The ENI record is stored in AWS's network control plane.
    The EC2 instance has a reference to the ENI (an attachment).
    The hypervisor enforces the attachment.
    The PHYSICAL LOCATION of the EC2 instance does not constrain
    which subnet the ENI can be in.

  Cross-ENI specifically:
    The ENI record says: "I belong to subnet 10.0.10.0/22 in YOUR VPC"
    The attachment says: "I am attached to API Server EC2 in AWS VPC"
    The hypervisor enforces BOTH simultaneously.
    From your VPC's routing perspective: 10.0.10.4 is a local address.
    From the API server's perspective: it has a network interface with IP 10.0.10.4.

  This is the technical foundation that makes Cross-ENIs possible:
  AWS's virtualized networking layer can attach a virtual network interface
  from one VPC to an EC2 instance running in a completely different VPC,
  because the association is maintained in software, not hardware.
```

**What you see in your AWS console:**

```
  EC2 --> Network Interfaces --> filter by Description

  You will see an ENI with:
    Status:       in-use
    Description:  Amazon EKS my-cluster
    Subnet:       subnet-0abc123 (your private subnet)
    Private IP:   10.0.10.4
    VPC:          vpc-0xyz789 (YOUR VPC)
    Security Groups: eks-cluster-sg-my-cluster-XXXXXXXX
    Attachment owner: XXXXXXXXXXXXXXXXX (an AWS account you don't own)

  The "Attachment owner" being a different account is the tell.
  This ENI is in your VPC/subnet but attached by an external AWS account.
  This is why you cannot delete it -- the attachment is controlled by EKS,
  not by you, even though the ENI appears in your account's network interface list.

  If you try to delete it:
    Error: "You are not authorized to manage an ENI that is attached
    to an instance in another account."
```

### How Many Cross-ENIs Are Created -- The AZ Rule

> **`[BPG]` Subnets:** "EKS places a X-ENI in each subnet specified during
> cluster creation (also called cluster subnets). The Kubernetes API server
> uses these Cross-Account ENIs to communicate with nodes deployed on the
> customer-managed cluster VPC subnets."

EKS creates **exactly one Cross-ENI per AZ** for the subnets you provide at
cluster creation. This is a per-AZ resource, NOT a per-node resource.

```
  2-AZ cluster (minimum recommended):
    You provide: subnet-priv-1a (10.0.10.0/22)
                 subnet-priv-1b (10.0.11.0/22)
    EKS creates: 1 Cross-ENI in subnet-priv-1a  --> IP: 10.0.10.4
                 1 Cross-ENI in subnet-priv-1b  --> IP: 10.0.11.4
    Total Cross-ENIs: 2

  3-AZ cluster:
    You provide: subnet-priv-1a (10.0.10.0/22)
                 subnet-priv-1b (10.0.11.0/22)
                 subnet-priv-1c (10.0.12.0/22)
    EKS creates: 1 Cross-ENI in subnet-priv-1a  --> IP: 10.0.10.4
                 1 Cross-ENI in subnet-priv-1b  --> IP: 10.0.11.4
                 1 Cross-ENI in subnet-priv-1c  --> IP: 10.0.12.4
    Total Cross-ENIs: 3

  Important: ALL worker nodes in the same AZ share the SAME Cross-ENI.
    AZ-a has 10 worker nodes:
      All 10 kubelets connect to 10.0.10.4 (the single AZ-a Cross-ENI)
      The API server on the other end handles all 10 connections
      There is no per-node Cross-ENI

  Why one per AZ (not one per node)?
    A Cross-ENI is not a per-connection resource -- it is a network interface
    that routes packets. The API server handles thousands of concurrent kubelet
    connections over a single Cross-ENI interface. The AZ boundary is what
    matters for HA, not the node count.
```

### Which Subnets Are Used for Cross-ENIs and Can You Control It?

This is one of the most important and least documented aspects of EKS cluster
creation. The answer directly affects your IP planning.

**What determines which subnets receive Cross-ENIs:**

```
  When you create a cluster, you specify a set of subnets to EKS.
  These are called CLUSTER SUBNETS (or control plane subnets).
  EKS places exactly ONE Cross-ENI in each distinct AZ across the
  cluster subnets you provide.

  Example 1 -- You provide 2 subnets in 2 AZs:
    subnet-priv-1a  (10.0.10.0/22, AZ: us-east-1a)
    subnet-priv-1b  (10.0.11.0/22, AZ: us-east-1b)

    EKS creates:
      Cross-ENI in subnet-priv-1a  --> IP: 10.0.10.4
      Cross-ENI in subnet-priv-1b  --> IP: 10.0.11.4
    Total: 2 Cross-ENIs

  Example 2 -- You provide 3 subnets but only 2 AZs:
    subnet-priv-1a    (10.0.10.0/22, AZ: us-east-1a)
    subnet-priv-1b    (10.0.11.0/22, AZ: us-east-1b)
    subnet-priv-1b-2  (10.0.20.0/22, AZ: us-east-1b)  <-- same AZ as 1b

    EKS creates:
      Cross-ENI in subnet-priv-1a    --> IP: 10.0.10.4  (one for AZ-a)
      Cross-ENI in ONE of the AZ-b subnets (EKS picks, not you)
    Total: still 2 Cross-ENIs (one per AZ -- not one per subnet)

  Example 3 -- You provide 4 subnets in 3 AZs (recommended production):
    subnet-priv-1a  (AZ: us-east-1a)
    subnet-priv-1b  (AZ: us-east-1b)
    subnet-priv-1c  (AZ: us-east-1c)
    Plus any additional subnets in same or different AZs

    EKS creates:
      Cross-ENI in AZ-a subnet  --> one IP consumed
      Cross-ENI in AZ-b subnet  --> one IP consumed
      Cross-ENI in AZ-c subnet  --> one IP consumed
    Total: 3 Cross-ENIs
```

**For the 2-AZ lab setup (HA Topology from Part 8):**

```
  VPC: 10.0.0.0/16
  Public subnets:   10.0.1.0/24 (AZ-a)   10.0.2.0/24 (AZ-b)
  Private subnets:  10.0.10.0/22 (AZ-a)  10.0.11.0/22 (AZ-b)

  Which subnets are specified at cluster creation?
    Recommended: the PRIVATE subnets only
    (subnet-priv-1a and subnet-priv-1b)

  Where Cross-ENIs are placed:
    Cross-ENI in subnet-priv-1a (10.0.10.0/22) --> gets IP e.g. 10.0.10.4
    Cross-ENI in subnet-priv-1b (10.0.11.0/22) --> gets IP e.g. 10.0.11.4

  Why private subnets (not public)?
    Cross-ENIs carry cluster control traffic (kubelet <--> API server).
    Placing them in public subnets would mean they are routable from
    the internet via the IGW (security risk).
    In private subnets: Cross-ENIs are only reachable from inside the VPC.
    Worker nodes (also in private subnets) can reach them directly.

  Can you specify a mix of public and private subnets for the cluster?
    Yes -- EKS allows it technically.
    Not recommended: Cross-ENIs in public subnets are reachable from internet.
    Subnets containing Cross-ENIs should always be private.
```

**Can you control which exact subnet in an AZ gets the Cross-ENI?**

```
  If you provide only ONE subnet per AZ: EKS places the Cross-ENI there.
  You have full control by providing exactly one subnet per AZ.

  If you provide MULTIPLE subnets in the same AZ:
    EKS picks one -- you cannot control which one.
    EKS may distribute Cross-ENIs across subnets for load, but this
    is an AWS internal implementation detail not guaranteed in docs.

  Best practice: provide exactly one private subnet per AZ as cluster subnets.
  This gives you deterministic Cross-ENI placement and IP planning.
```

> **`[BPG]` Subnets:** "Kubernetes worker nodes can run in the cluster subnets,
> but it is not recommended. During cluster upgrades Amazon EKS provisions
> additional ENIs in the cluster subnets. When your cluster scales out, worker
> nodes and pods may consume the available IPs in the cluster subnet. Hence
> consider using dedicated cluster subnets with /28 netmask."

**The "cluster subnet" vs "node subnet" distinction -- what it means:**

```
  "Cluster subnets" = subnets specified at CLUSTER CREATION TIME
    Purpose: Cross-ENI placement only
    EKS official recommendation: dedicated /28 subnets (just for Cross-ENIs)
    Each /28 has 11 usable IPs -- more than enough for Cross-ENIs
    Cross-ENIs consume only 1-2 IPs per AZ

  "Node subnets" = subnets specified at NODE GROUP creation time
    Purpose: Worker node EC2 instances and pod IPs
    Must be LARGE (see Part 10 for /22 requirement)
    Can be the same as cluster subnets (common but not recommended)
    OR separate subnets (recommended for large clusters)

  For this lab series:
    We use the same private subnets for BOTH cluster creation and node groups.
    This is fine for a learning environment with 2 nodes.
    For production: use dedicated /28 subnets for cluster creation (Cross-ENIs)
    and separate /22+ subnets for node groups.

  Why does it matter?
    During EKS control plane upgrades, EKS provisions ADDITIONAL Cross-ENIs
    temporarily (for rolling upgrade of API servers).
    If your cluster subnet is also your node subnet (/22 shared):
      Nodes + pods + upgrade Cross-ENIs all compete for the same IP pool.
      On a busy cluster at scale, upgrade can temporarily cause IP exhaustion.
    With dedicated /28 cluster subnets:
      Cross-ENIs get their own tiny isolated IP space.
      No competition with node or pod IPs during upgrades.
```

### What Happens to AZ-a Nodes if the AZ-a Cross-ENI Fails?

```
  Scenario: Cross-ENI in AZ-a becomes unreachable

  AZ-a nodes:
    kubelet heartbeats stop reaching API server
    After 40s: API server marks AZ-a nodes as Unknown
    After 5 min: Pods evicted from AZ-a nodes
    AZ-a nodes cannot receive new pod assignments

  AZ-b and AZ-c nodes (if 3-AZ cluster):
    Completely unaffected -- they use their own Cross-ENIs
    Cluster continues scheduling pods to AZ-b and AZ-c
    Scheduler detects AZ-a is unavailable and avoids it

  Recovery:
    EKS auto-recovers Cross-ENIs -- this is part of the managed service
    You cannot manually intervene on Cross-ENIs

  Prevention: always use subnets in at least 2 AZs so that one AZ Cross-ENI
  failure does not take down the entire cluster's node connectivity.
```

> **`[BPG]` Subnets:** "Amazon EKS recommends you specify subnets in at
> least two availability zones when you create a cluster."

---

## Part 6: The NLB vs Cross-ENIs -- Two Traffic Paths, Two Audiences

This is one of the most common sources of confusion in EKS networking.
The NLB and Cross-ENIs serve completely different audiences.

```
  TRAFFIC PATH 1: External Clients --> NLB --> API Server
  ─────────────────────────────────────────────────────────

  WHO uses the NLB endpoint:
    kubectl (your laptop or CI-CD pipeline)
    AWS Console (when you browse EKS resources)
    eksctl (cluster lifecycle management)
    Helm, Terraform, ArgoCD (Kubernetes API clients)
    Any external caller who needs to talk to the Kubernetes API

  Traffic flow:
    kubectl get pods
      |
      | HTTPS to https://XXXXX.gr7.us-east-1.eks.amazonaws.com
      | (the NLB DNS name in your kubeconfig)
      v
    NLB  (in the AWS-managed VPC)
      |  load balances across API server replicas
      v
    kube-apiserver  (replica 1 or 2)
      |  reads/writes via etcd
      v
    Response back to kubectl


  TRAFFIC PATH 2: Worker Nodes (kubelets) --> Cross-ENI --> API Server
  ─────────────────────────────────────────────────────────────────────

  WHO uses Cross-ENIs:
    kubelet heartbeats (every 10 seconds per node)
    kubelet WATCH streams (long-lived connections waiting for pod assignments)
    kubelet status updates (pod started, failed, etc.)
    kube-proxy WATCH streams (Service/EndpointSlice changes)
    VPC CNI WATCH streams (pod assignments for IP management)

  Traffic flow (Public+Private mode -- recommended):
    kubelet heartbeat
      |
      | HTTPS to Cross-ENI IP  (e.g. 10.0.10.4)
      | Traffic stays ENTIRELY inside your VPC
      | No internet, no NAT Gateway, no NLB
      v
    Cross-ENI  (IP: 10.0.10.4, lives in your subnet)
      |  connects through to API server in AWS-managed VPC
      v
    kube-apiserver
      |  updates node status in etcd
      v
    Response back to kubelet via same Cross-ENI path
```

### Why Two Separate Paths?

```
  NLB pros for external clients:
    Publicly accessible (can use kubectl from anywhere)
    Stable DNS name (never changes, even during upgrades)
    Load balanced (distributes kubectl across API server replicas)
    Supports access control via publicAccessCidrs restriction

  Cross-ENI pros for internal cluster traffic:
    Traffic never leaves your VPC (security + compliance)
    Lower latency (no internet round-trip)
    No data transfer cost (in-VPC traffic is free)
    Not affected by public endpoint restrictions (whitelist doesn't apply)
    Operates even if the NLB or public internet is disrupted

  Combined: kubelets always use Cross-ENIs (in Public+Private mode).
  NLB handles only the human and tooling traffic from outside the cluster.
  This is why Public+Private is the recommended mode -- best of both.
```

### What the NLB in the AWS-Managed VPC Actually Does

Yes -- the NLB in the control plane VPC has exactly one job: **it is a
stable DNS endpoint that fronts the Kubernetes API server replicas**. But
understanding the full scope of what that means is important.

```
  The NLB provides:

  1. STABLE DNS NAME
     The NLB gets a fixed DNS name when the cluster is created:
       https://XXXXXXXXXXXXXX.gr7.us-east-1.eks.amazonaws.com
     This DNS name never changes for the lifetime of the cluster.
     Even during upgrades when API server EC2 instances are replaced,
     the NLB endpoint remains stable.
     Your kubeconfig always points to this DNS name.

  2. LOAD BALANCING ACROSS API SERVER REPLICAS
     EKS runs minimum 2 API server replicas across 2 distinct AZs.
     The NLB distributes HTTPS connections across all healthy replicas.
     If one API server becomes unhealthy: NLB removes it from rotation.
     You never configure this -- AWS manages the NLB target group.

  3. HEALTH CHECKING
     The NLB runs health checks against each API server replica.
     Unhealthy replicas are removed from rotation automatically.
     EKS replaces the failed replica -- the NLB starts routing to
     the new one once health checks pass.

  4. TLS TERMINATION ENDPOINT
     All kubectl HTTPS connections terminate at the NLB.
     The NLB forwards the connection to an API server replica.
     The API server handles authentication, authorization, and the
     actual request processing.

  What the NLB does NOT do:
    It does NOT handle traffic between your nodes and the API server.
    Kubelets do NOT connect through the NLB (in Public+Private mode).
    The NLB does NOT balance your application traffic.
    The NLB does NOT appear in your VPC or your billing.
    You cannot see, modify, or configure this NLB.

  Important distinction -- two separate NLBs exist in EKS:
    1. NLB in the AWS-managed VPC (control plane) -- described above
       Invisible to you, included in $0.10/hr, fronts API servers
    2. NLB in YOUR VPC (created by AWS Load Balancer Controller)
       Visible to you, billed to you, fronts your application pods
       Created when you apply a Service of type: LoadBalancer
       These are completely separate and have no relationship
```

---

## Part 7: Why Cross-ENIs -- Not VPC Peering, Transit Gateway, or PrivateLink

AWS had several connectivity options available when designing EKS. Each was
evaluated and rejected for specific, verifiable technical reasons.

### The Five Hard Requirements

```
  1. No CIDR overlap constraint
     AWS runs millions of EKS clusters. Customer VPC CIDRs are chosen
     by customers — AWS cannot control or predict them. Any connectivity
     solution requiring non-overlapping CIDRs fails at global scale.

  2. Bi-directional — control plane must initiate to nodes
     The API server must PUSH pod specifications to kubelet on worker nodes.
     kubelet must PULL instructions FROM the API server.
     Any solution that only allows one side to initiate connections fails.

  3. Fully managed and invisible to the customer
     The customer should not need to configure route tables, attachments,
     peering connections, or any network infrastructure to connect their
     nodes to the control plane. It must work automatically.

  4. Cost absorbed into the EKS price
     AWS charges $0.10/hour for the control plane. The connectivity
     mechanism must be implementable within that pricing model at scale.

  5. Dynamic — must track nodes as they come and go
     Worker nodes are EC2 instances. They are launched and terminated
     constantly, especially with autoscaling. The connectivity mechanism
     must work for any node in the provided subnets without per-node config.
```

### Option 1: VPC Peering — Why It Was Not Used

VPC Peering connects two VPCs at the routing layer, allowing resources in
each VPC to communicate as if they were on the same network.

```
  AWS-managed VPC  <──── VPC Peering ────>  Your VPC

  Problems:

  1. CIDR OVERLAP KILLS IT AT SCALE
     VPC Peering requires non-overlapping CIDR blocks between the two VPCs.
     AWS-managed VPCs use private IP ranges (e.g. 10.x.x.x, 172.16.x.x).
     Customer VPCs also commonly use the same ranges.
     With millions of clusters, CIDR conflicts would be unavoidable.

     Example of the failure:
       AWS control plane VPC: 10.0.0.0/16
       Your VPC:              10.0.0.0/16  (very common default)
       Result: peering rejected -- CIDRs overlap, routing is ambiguous

  2. CUSTOMER MUST CONFIGURE ROUTE TABLES
     VPC Peering creates the peering connection, but the customer must
     manually add routes to their route tables pointing to the peer VPC.
     This cannot be done automatically without customer involvement.
     AWS cannot modify your route tables without explicit permission.

  3. BROAD BLAST RADIUS
     A peered VPC opens network-level access between the two VPCs at the
     routing layer. Any resource in the AWS-managed VPC could potentially
     reach any resource in your VPC -- not just the API server talking
     to your kubelets. This violates the principle of minimal exposure.

  4. NON-TRANSITIVE -- BREAKS MULTI-CLUSTER TOPOLOGIES
     VPC Peering is non-transitive: A peers with B, B peers with C,
     but A cannot reach C. This complicates multi-cluster and hub-spoke
     architectures that organizations commonly build.
```

### Option 2: Transit Gateway — Why It Was Not Used

Transit Gateway is a regional network hub that connects many VPCs and
on-premises networks through a single gateway.

```
  AWS-managed VPC  ──┐
                     ├── Transit Gateway ── Your VPC
  Other VPCs       ──┘

  Problems:

  1. COST MODEL DOES NOT WORK AT SCALE
     Transit Gateway charges per attachment: $0.05/hour per VPC attachment.
     For millions of EKS clusters, this cost is enormous -- and AWS absorbs
     the control plane infrastructure cost into the $0.10/hour EKS fee.
     Adding $0.05/hour per attachment on top would be prohibitive.

  2. SAME CIDR OVERLAP PROBLEM AS PEERING
     Transit Gateway also requires non-overlapping CIDRs across all attached
     VPCs for routing to work correctly. Same failure mode as VPC Peering
     at global scale with customer-chosen CIDR ranges.

  3. MASSIVE OVERKILL FOR THE USE CASE
     Transit Gateway is architected for connecting dozens or hundreds of VPCs
     across multiple accounts and regions. EKS only needs one targeted
     connection: API server <--> specific worker nodes in specific subnets.
     Using TGW for this is engineering a highway interchange to connect
     two adjacent rooms.

  4. CUSTOMER MUST CREATE AND CONFIGURE ATTACHMENTS
     Transit Gateway requires explicit attachment creation, route table
     configuration on both sides, and propagation rules. Cannot be set up
     transparently from the customer's perspective.
```

### Option 3: PrivateLink — Why It Was Not Used

AWS PrivateLink exposes a specific service endpoint from one VPC into
another VPC via an interface endpoint ENI -- without full VPC connectivity.

```
  Provider VPC  ──> NLB ──> PrivateLink ──> Interface ENI in consumer VPC

  This is conceptually closest to Cross-ENIs, but has key differences.

  Problems:

  1. ONE-WAY DIRECTION -- PROVIDER CANNOT INITIATE
     PrivateLink is consumer-initiated only. The consumer calls the service.
     The provider cannot initiate connections back to the consumer.
     EKS requires the API server (in the AWS VPC) to INITIATE connections
     to worker node kubelets (in your VPC) for:
       - Pushing pod specs and configuration changes
       - kubectl exec (opens a channel from API server to kubelet)
       - kubectl logs (API server streams from kubelet)
       - kubectl port-forward (API server proxies to kubelet)
     PrivateLink cannot support the control plane reaching INTO your VPC
     to communicate with individual kubelets.

  2. FIXED TARGET -- CANNOT TRACK DYNAMIC NODES
     PrivateLink routes traffic to a fixed NLB target. Worker nodes are
     created and destroyed dynamically by Auto Scaling. There is no
     mechanism for PrivateLink to track individual kubelet endpoints
     across an ever-changing fleet of EC2 instances.

  3. CIDR OVERLAP CONSTRAINT
     PrivateLink Interface Endpoints consume IPs from the consumer subnet.
     The consumer VPC and provider VPC can have overlapping CIDRs because
     PrivateLink uses private DNS and ENIs rather than direct routing --
     this part actually works. But the unidirectional limitation is
     the fundamental blocker.
```

### Why Cross-ENIs Satisfy All Requirements

```
  REQUIREMENT 1: No CIDR overlap constraint
    Cross-ENI solution:
      The Cross-ENI gets ONE IP from your existing subnet CIDR.
      The AWS-managed VPC CIDR is irrelevant -- it never participates
      in your routing table. The only address that matters is the
      Cross-ENI IP (e.g. 10.0.10.4) which is just another IP in
      your subnet that you already own.
      No overlap constraint. Works with any VPC CIDR.

  REQUIREMENT 2: Bi-directional -- control plane initiates to nodes
    Cross-ENI solution:
      The API server instance in the AWS VPC has eth1 = Cross-ENI
      at 10.0.10.4 inside your subnet.
      API server can INITIATE connections to 10.0.10.15 (a worker node)
      -- both are in the same subnet, plain L3 routing, no restriction.
      Worker node kubelet can INITIATE connections to 10.0.10.4 (Cross-ENI)
      -- same subnet, no restriction.
      Fully bidirectional.

  REQUIREMENT 3: Fully managed, invisible to the customer
    Cross-ENI solution:
      EKS creates the Cross-ENI automatically when you create the cluster.
      No route table changes needed -- the Cross-ENI IP is already in
      your subnet's local routing domain.
      The only customer action: provide the subnet IDs at cluster creation.
      Everything else: AWS handles.

  REQUIREMENT 4: Cost absorbed into EKS pricing
    Cross-ENI solution:
      ENIs are free to attach. The only ongoing cost of a Cross-ENI
      is one IP address consumed from your subnet CIDR.
      No per-attachment fee, no per-GB charge.
      Fully absorbable within the $0.10/hour EKS price.

  REQUIREMENT 5: Dynamic -- works for any node in the subnet
    Cross-ENI solution:
      Any EC2 instance launched in the provided subnet can automatically
      communicate with the Cross-ENI. No per-node configuration.
      A new Auto Scaling node that joins the subnet can reach the API
      server through the Cross-ENI immediately -- no provisioning step.
```

### The Fundamental Insight

```
  Cross-ENI is not a VPC-to-VPC connection.
  It is a network interface that belongs to YOUR subnet.

  The control plane instance in the AWS VPC has:
    eth0  -->  its own AWS VPC network  (internal control plane traffic)
    eth1  -->  YOUR subnet              (talks to your worker nodes)

  From your worker node's perspective:
    The API server is just another device at 10.0.10.4.
    No tunnels. No routing complexity. No overlapping CIDR issues.
    It is literally a local subnet neighbor.

  This is the minimum viable surface area to connect two VPCs:
    One IP address per AZ, inside your existing subnet,
    fully managed by AWS, with zero customer configuration.
```

### Summary Comparison

```
  +-----------------+----------+----------+------------+-------------+
  | Requirement     | Peering  |   TGW    | PrivateLink | Cross-ENI  |
  +-----------------+----------+----------+------------+-------------+
  | No CIDR overlap | FAILS    | FAILS    | OK         | OK          |
  | at scale        |          |          |            | (1 IP from  |
  |                 |          |          |            | your subnet)|
  +-----------------+----------+----------+------------+-------------+
  | Fully AWS managed| No        | No        | Partial   | Yes       |
  | (no customer     | (needs    | (needs    |           |           |
  | config)          | routes)   | attach)   |           |           |
  +------------------+-----------+-----------+-----------+-----------+
  | Bi-directional  | Yes      | Yes      | FAILS      | Yes         |
  | (CP initiates   |          |          | (one-way)  |             |
  | to nodes)       |          |          |            |             |
  +-----------------+----------+----------+------------+-------------+
  | Zero customer   | FAILS    | FAILS    | Partial    | Yes         |
  | configuration   | (routes) | (attach) |            |             |
  +-----------------+----------+----------+------------+-------------+
  | Cost absorbed   | Yes      | FAILS    | Partial    | Yes         |
  | into EKS fee    |          | $0.05/hr |            | (free)      |
  +-----------------+----------+----------+------------+-------------+
  | Dynamic for     | N/A      | N/A      | FAILS      | Yes         |
  | any node        |          |          | (fixed NLB)| (subnet     |
  |                 |          |          |            | scope)      |
  +-----------------+----------+----------+------------+-------------+
  | Scales to       | FAILS    | FAILS    | FAILS      | Yes         |
  | millions of     | (CIDR)   | (CIDR)   | (direction)|             |
  | clusters        |          |          |            |             |
  +-----------------+----------+----------+------------+-------------+
  
```

---

## Part 8: High Availability -- Component Level and Cluster Level

Understanding HA in EKS means knowing which failures AWS protects you against
and which ones you must design around yourself.

### Component-Level HA (AWS-managed)

```
  kube-apiserver:
    EKS runs MINIMUM 2 replicas in distinct AZs
    NLB health-checks each replica and routes only to healthy ones
    If one API server replica fails: NLB stops routing to it
    The other replica handles all traffic
    EKS auto-replaces the failed replica
    You never see this happen -- control plane SLA: 99.95% uptime

  etcd:
    EKS runs exactly 3 nodes across 3 distinct AZs
    Uses Raft consensus: quorum = floor(3/2) + 1 = 2 nodes

    Raft quorum math:
      3 nodes: can tolerate 1 node failure (2 of 3 remain = quorum)
      3 nodes: cannot tolerate 2 simultaneous failures
      An AZ failure loses 1 etcd node -- 2 remaining = quorum maintained
      All 3 AZs failing simultaneously = quorum lost = API server read-only

    What happens when etcd loses quorum:
      API server enters read-only mode
      You can still run: kubectl get (reads from API server cache)
      You cannot run: kubectl apply / delete / create
      Existing running pods CONTINUE running (kubelet is independent)
      No new scheduling, no changes to any resource

  kube-controller-manager:
    Runs in active-standby mode (leader election via etcd)
    Only one instance is active at a time
    If active instance fails: standby wins leader election in seconds
    Transparent to you

  kube-scheduler:
    Same active-standby pattern as controller-manager
    Failover in seconds, transparent to you
```

### kube-controller-manager and kube-scheduler -- Active/Standby Details

```
  Both components use Kubernetes leader election to implement active/standby.
  Only ONE instance is active at any time -- the active leader.
  All other instances are in standby, watching for the leader to disappear.

  HOW LEADER ELECTION WORKS:
  ──────────────────────────────────────────────────────────────────────
  Kubernetes uses a Lease object in etcd to implement leader election.
  The current leader writes a renewal timestamp to the Lease every 2 seconds
  (default leaseDuration = 15s, renewalDeadline = 10s, retryPeriod = 2s).

  If the active leader fails:
    Other instances notice the Lease has not been renewed (after 15s)
    They race to claim the Lease by writing their identity
    First one to write wins -- becomes the new active leader
    Total failover time: typically 15-20 seconds

  Is this HOT STANDBY?
    YES -- it is hot standby.
    Standby instances are running, healthy, and connected to the API server.
    They are reading cluster state (watching resources) continuously.
    They are NOT processing events or making API calls (only the leader does).
    When failover happens: the new leader starts processing immediately.
    No warm-up time needed -- it already has cluster state from its watch streams.

  WHICH AZS do they run in?
    EKS runs controller-manager and scheduler inside the AWS-managed VPC.
    AWS does not publicly document the exact AZ placement of these components.
    However, EKS guarantees: if the API server replicas are across 2 AZs,
    the controller-manager and scheduler are also placed for HA.
    In practice: EKS places one active + at least one standby, distributed
    across the same AZs as the API server replicas.
    You never see this -- it is completely managed by EKS.

  FAILOVER TIMELINE:
    t=0:      Active controller-manager instance fails (hardware, OS, etc.)
    t=0-15s:  Standby instances notice Lease not renewed
    t=15s:    Standby instances race to claim Lease
    t=~16s:   New leader elected, starts processing control loops
    t=~18s:   Controller reconciliation running again on new leader
    Impact:   Any Deployments, ReplicaSets, Services in transition may have
              a 15-20 second delay before their controller acts on changes.
              Already-running pods: completely unaffected.
              New pod scheduling: delayed by up to ~16-18 seconds.
              This is transparent in practice -- rarely noticeable.
```

### Cluster-Level HA (your responsibility)

```
  Cross-ENI HA:
    EKS creates one Cross-ENI per AZ you provide
    If one AZ's Cross-ENI is unreachable:
      Nodes in that AZ lose API server connectivity
      Nodes in other AZs are unaffected
    Protection: always provide subnets in 2+ AZs

  Worker Node HA:
    Nodes are EC2 instances -- they can fail, be terminated, or be
    replaced during scaling events
    Protection: use Managed Node Groups with min=2 across 2+ AZs
    EKS Managed Node Groups use ASGs that span multiple AZs
    If one node fails: ASG launches a replacement

  Pod HA:
    A single-replica Deployment has zero HA
    If its node fails: pod is evicted, new pod scheduled elsewhere
    Recovery time: node detection (40s) + eviction + scheduling + image pull
    Protection: run minimum 2 replicas per Deployment
    For critical workloads: use PodDisruptionBudgets (PDB)
    For AZ spread: use topologySpreadConstraints

  NAT Gateway HA:
    A single NAT Gateway in AZ-a is a single point of failure
    If AZ-a fails: nodes in AZ-b lose outbound internet access
    Even if AZ-b NAT GW is fine, its route table points to AZ-a
    Protection: one NAT Gateway per AZ, each AZ's private route table
    points to its own NAT Gateway

  HA Topology for This Lab (minimum viable production-like):
  ──────────────────────────────────────────────────────────
    VPC: 2 AZs (us-east-1a, us-east-1b)
    2 public subnets:  one per AZ (for NAT GW and LBs)
    2 private subnets: one per AZ (for nodes and pods)
    2 NAT Gateways:    one per AZ (one in each public subnet)
    2 Cross-ENIs:      created by EKS, one per AZ
    2 worker nodes:    one per AZ (minimum for HA)
    2 CoreDNS replicas: spread across both nodes
    API server:        2+ replicas in 2 AZs (AWS managed)
    etcd:              3 nodes in 3 AZs (AWS managed)
```

```
  Full 3-AZ HA Layout (production reference):

  +------------------------------------------------------------------+
  |  YOUR VPC  10.0.0.0/16                                          |
  |                                                                  |
  |  AZ-1a              AZ-1b              AZ-1c                    |
  |  pub 10.0.1.0/24    pub 10.0.2.0/24    pub 10.0.3.0/24          |
  |  NAT-GW-a           NAT-GW-b           NAT-GW-c                 |
  |                                                                  |
  |  priv 10.0.10.0/22  priv 10.0.11.0/22  priv 10.0.12.0/22        |
  |  Node-1 (AZ-a)      Node-2 (AZ-b)      Node-3 (AZ-c)            |
  |  Cross-ENI 10.4     Cross-ENI 11.4     Cross-ENI 12.4           |
  |                                                                  |
  |  Route tables:                                                   |
  |    priv-1a: 0.0.0.0/0 --> NAT-GW-a                             |
  |    priv-1b: 0.0.0.0/0 --> NAT-GW-b                             |
  |    priv-1c: 0.0.0.0/0 --> NAT-GW-c                             |
  |                                                                  |
  |  AZ failure (e.g. AZ-a goes down):                              |
  |    NAT-GW-a gone: AZ-b and AZ-c unaffected (own NAT GWs)        |
  |    Cross-ENI-a gone: AZ-b and AZ-c unaffected (own Cross-ENIs)  |
  |    Node-1 gone: workloads rescheduled to Node-2 and Node-3       |
  |    API server (AWS): 2+ replicas still in AZ-b/c                |
  |    etcd (AWS): 2 of 3 nodes still in AZ-b/c = quorum maintained |
  +------------------------------------------------------------------+
```

> **`[BPG]` Reliability:** "Use at least two availability zones for your
> worker nodes. This ensures that workloads can continue to run even if
> one AZ becomes unavailable. Use multiple replicas for critical workloads.
> Configure PodDisruptionBudgets to prevent too many pods from being
> unavailable simultaneously."

---

## Part 9: Data Plane Options -- All Four Modes

EKS supports four data plane models. Understanding the trade-offs determines
which you use for which workload type.

```
  +----------------+  +----------------+  +----------------+  +------------+
  | MANAGED NODE   |  |  EKS AUTO MODE |  |    FARGATE     |  |   SELF-    |
  | GROUPS         |  |  (NEW 2024)    |  |  (serverless)  |  |  MANAGED   |
  +----------------+  +----------------+  +----------------+  +------------+
  |                |  |                |  |                |  |            |
  | AWS manages:   |  | AWS manages:   |  | No EC2 nodes   |  | You manage |
  |  OS patching   |  |  Node lifecycle|  | AWS runs each  |  | everything |
  |  AMI updates   |  |  AMI (Bottle-  |  | pod in isolated|  |  OS patch  |
  |  Node replace  |  |   rocket only) |  | Firecracker VM |  |  AMI       |
  |  ASG lifecycle |  |  Karpenter     |  |                |  |  kubelet   |
  |                |  |  (managed)     |  | You define     |  |  CNI       |
  | You control:   |  |  LB Controller |  | per pod:       |  |  Upgrades  |
  |  Instance type |  |  (managed)     |  |  vCPU          |  |            |
  |  Min/max nodes |  |  EBS CSI driver|  |  Memory        |  | Full ctrl  |
  |  Launch tplt   |  |  (managed)     |  |                |  | Full resp  |
  |  Spot/OD       |  |  Node count    |  | No SSH         |  |            |
  |  Labels/taints |  |  (auto via     |  | No DaemonSets  |  | Use for:   |
  |                |  |   Karpenter)   |  | No privileged  |  |  GPU nodes |
  | DaemonSets: YES|  |                |  | No hostNetwork |  |  Custom AMI|
  | SSH: YES        |  | DaemonSets: NO |  |                |  |  FIPS/HSM  |
  | Spot: YES       |  | SSH: NO        |  | Cost:          |  |            |
  | Custom AMI: YES |  | Custom AMI: NO |  | $0.04048/vCPU  |  | Not used   |
  |                |  | Spot: YES      |  | /hr            |  | this series|
  | [*] This lab   |  |                |  | $0.004445/GB   |  |            |
  |  (t3.small)    |  | Pricing:       |  | /hr            |  |            |
  |                |  | EC2 price +    |  |                |  |            |
  |                |  | Auto Mode fee  |  | Demo-05        |  |            |
  |                |  | (~$0.01/hr/    |  |                |  |            |
  |                |  |  node extra)   |  |                |  |            |
  |                |  |                |  |                |  |            |
  |                |  | Demo-XX        |  |                |  |            |
  +----------------+  +----------------+  +----------------+  +------------+
```

### EKS Auto Mode -- Deep Dive (re:Invent 2024)

EKS Auto Mode was announced at AWS re:Invent 2024 and is now GA in all
standard AWS regions. It represents the next step in the EKS shared
responsibility model -- AWS takes over the data plane in addition to the
control plane.

```
  Without Auto Mode (traditional EKS):
    AWS manages:  Control plane (API server, etcd, controllers)
    YOU manage:   Node groups, Karpenter, LB Controller, CNI updates,
                  OS patching, AMI selection, autoscaling config,
                  EBS CSI driver, CoreDNS updates, kube-proxy updates

  With Auto Mode:
    AWS manages:  Control plane + ALL of the above
    YOU manage:   Your applications and workload manifests
                  Kubernetes version upgrades (cluster level)
                  Custom NodePools/NodeClasses if you need them
```

**How EKS Auto Mode works under the hood:**

```
  Foundation: EC2 Managed Instances
    Auto Mode uses a new EC2 feature: Managed Instances.
    You delegate operational control of the EC2 instance to EKS.
    AWS can patch, replace, and upgrade the instance without your action.
    You retain billing ownership -- EC2 costs appear in your account.
    Compute Savings Plans and Reserved Instances apply to Auto Mode nodes.

  Node provisioning: Managed Karpenter
    Auto Mode uses Karpenter (open source) but runs it off-cluster.
    Karpenter components do NOT appear in your kube-system namespace.
    AWS manages the Karpenter deployment, scaling, and upgrades.
    Behavior: when pods are Pending, Auto Mode provisions exactly the
    right EC2 instance type for those pods within seconds.

  Two default NodePools (pre-configured by AWS):
    general-purpose:  AMD/x86 instances, On-Demand + Spot, general apps
    system:           Reserved for cluster system components (CoreDNS, etc.)
    You can add custom NodePools alongside defaults for specific needs.
    Do NOT edit the default NodePools -- add new ones instead.

  Node OS: Bottlerocket (locked down)
    Purpose-built container OS by AWS
    Read-only root filesystem
    Minimal attack surface (no SSH, no package manager on running nodes)
    Auto-patched and replaced -- nodes have a MAX lifetime of 21 days

  Key operational differences from Managed Node Groups:
    No SSH to nodes (even with Systems Manager -- disabled)
    No custom AMI -- Bottlerocket only
    DaemonSets do NOT run on Auto Mode nodes
    Components like LB Controller, EBS CSI driver run OFF-CLUSTER
      (you cannot kubectl get pods -n kube-system and see them)
    Monitoring via kubectl, CloudWatch Logs, EKS console only

  You CAN mix Auto Mode nodes with Managed Node Groups in the same cluster.
  Useful pattern: Auto Mode for general workloads + Managed NG for DaemonSets,
  GPU nodes, or workloads needing direct node access.
```

**Auto Mode pricing:**
```
  Total cost per Auto Mode node = EC2 On-Demand price + Auto Mode fee
  Auto Mode fee varies by instance type, billed per second (1 min minimum)
  Example:
    m5.large:   EC2 $0.096/hr  +  Auto Mode ~$0.01152/hr  = ~$0.108/hr
    c6a.2xlarge: EC2 $0.306/hr +  Auto Mode ~$0.03672/hr  = ~$0.343/hr
  Auto Mode fee applies even when using Spot or Reserved Instances for EC2.
  For >150 nodes across your org: contact AWS account team for pricing.
```

> **`[BPG]` Auto Mode:** "Auto Mode is geared towards users that want the
> benefits of Kubernetes and EKS but need to minimize operational burden
> around Kubernetes like upgrades and installation/maintenance of critical
> platform pieces like auto-scaling, load balancing, and storage."

**When to choose each data plane:**

```
  Managed Node Groups:
    You need DaemonSets (Fluent Bit, Prometheus node-exporter)
    You need SSH access or SSM to nodes for debugging
    You need a custom AMI (FIPS, CIS, security scanning agents)
    You need GPU or specialized hardware
    You want maximum visibility into the data plane
    --> DEFAULT CHOICE for teams learning EKS (this lab series)

  EKS Auto Mode:
    You want to minimize operational overhead dramatically
    You are comfortable with no SSH/DaemonSets on managed nodes
    You want automatic right-sizing and cost optimization
    You have standard workloads that don't need custom AMIs
    --> IDEAL FOR: teams that want production-ready clusters fast
    --> Future demo: Demo-XX in this series

  Fargate:
    You want completely serverless pods with per-pod VM isolation
    Your workloads are stateless with predictable resource needs
    You run batch jobs or event-driven processing
    You are OK with higher per-unit cost for lighter operations
    --> Demo-05 in this series

  Self-Managed:
    You have compliance requirements needing specific OS configurations
    You need HSM drivers, FIPS mode, or custom kernel modules
    You have existing Ansible/Packer automation for node builds
    --> Not used in this lab series
```

### Managed Node Groups vs EKS Auto Mode -- Complete Comparison

Both provide AWS-managed node lifecycle but with fundamentally different
models. The right choice depends on your operational maturity, workload
requirements, and cost priorities.

```
  ┌─────────────────────────────┬──────────────────────────┬──────────────────────────┐
  │ Aspect                      │ Managed Node Groups      │ EKS Auto Mode             │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ COMPUTE MANAGEMENT                                                                 │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Node provisioning           │ You define instance type  │ Auto Mode picks optimal   │
  │                             │ min/max in node group     │ type via managed Karpenter│
  │                             │ ASG config                │ No instance type to choose│
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Scaling mechanism           │ Cluster Autoscaler or    │ Managed Karpenter         │
  │                             │ Karpenter (you deploy)    │ (AWS deploys and runs it) │
  │                             │ Runs in your cluster      │ Runs OFF cluster (aws mgd)│
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Node OS / AMI               │ You choose:               │ Bottlerocket ONLY         │
  │                             │ Amazon Linux 2023 (AL2023)│ AWS manages and updates   │
  │                             │ Bottlerocket              │ You cannot customise it   │
  │                             │ Windows Server            │                           │
  │                             │ Custom AMI (your own)     │                           │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ OS / AMI patching           │ EKS releases new AMIs     │ AWS auto-patches and      │
  │                             │ You update node group     │ replaces nodes (max 21    │
  │                             │ to roll to new AMI        │ day node lifetime enforced│
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Node replacement            │ You trigger draining and  │ AWS replaces automatically│
  │                             │ rolling update            │ (on schedule or on patch) │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Max node lifetime           │ None (node runs until     │ 21 days maximum           │
  │                             │ you replace or terminate) │ Enforced by Auto Mode     │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ ACCESS AND VISIBILITY                                                               │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ SSH access to nodes         │ Yes (if you add SSH key   │ NO -- not possible        │
  │                             │ to launch template)       │                           │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ AWS Systems Manager (SSM)   │ Yes (SSM agent on nodes)  │ NO -- disabled by AWS     │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ DaemonSets on these nodes   │ Yes -- fully supported    │ NO -- DaemonSets do NOT   │
  │                             │ (Fluent Bit, Prometheus   │ run on Auto Mode nodes    │
  │                             │  node-exporter, etc.)     │                           │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ kubectl exec to pods        │ Yes                       │ Yes                       │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Node visible in kubectl     │ Yes (kubectl get nodes)   │ Yes (kubectl get nodes)   │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ KUBERNETES FEATURES                                                                 │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Custom node labels/taints   │ Yes -- full control       │ Limited -- via NodePool   │
  │                             │                           │ spec only                 │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Spot instances              │ Yes -- configure in ASG   │ Yes -- Auto Mode picks    │
  │                             │ or launch template        │ Spot automatically        │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ GPU / specialized hardware  │ Yes -- choose GPU instance│ Yes -- GPU instance types │
  │                             │ type in node group        │ available in NodePool     │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Privileged pods             │ Yes (if pod security      │ NO -- disabled by default │
  │                             │ policy allows)            │ on Auto Mode nodes        │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ hostNetwork pods            │ Yes                       │ NO                        │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ hostPath volumes            │ Yes                       │ NO                        │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ NETWORKING                                                                          │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ VPC CNI (pod networking)    │ Standard VPC CNI          │ VPC CNI managed by AWS    │
  │                             │ (you manage version)      │ (auto-updated)            │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ AWS LB Controller           │ You deploy and manage     │ Managed by AWS (off-cluster│
  │                             │ (runs in kube-system)     │ not visible in kubectl)   │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ COST                                                                                │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ EC2 compute charge          │ EC2 On-Demand/Spot/RI     │ EC2 On-Demand/Spot/RI     │
  │                             │ Standard EC2 pricing      │ Standard EC2 pricing      │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Auto Mode management fee    │ NONE                      │ Additional fee per node   │
  │                             │                           │ (charged by second)       │
  │                             │                           │ e.g. m5.large: ~$0.01/hr │
  │                             │                           │ on top of EC2 price       │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Savings Plans apply?        │ Yes -- Compute SP applies │ Compute SP applies to EC2 │
  │                             │ to EC2 portion            │ portion ONLY. Auto Mode   │
  │                             │                           │ fee not discounted by SPs │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Karpenter / Cluster Autoscal│ You pay for pods running  │ Included in Auto Mode fee │
  │ er pod cost                 │ Karpenter/CA on your nodes│                           │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Node right-sizing           │ Manual -- you choose      │ Automatic -- Auto Mode    │
  │                             │ instance type. May        │ picks exact right-sized   │
  │                             │ over/under provision      │ instance per workload     │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ OPERATIONAL                                                                         │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Add-on management           │ You install, configure,   │ AWS manages: VPC CNI, LB  │
  │                             │ upgrade: VPC CNI, LB Ctrl │ Controller, EBS CSI driver│
  │                             │ EBS CSI driver, etc.      │ These are off-cluster     │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ EKS upgrade impact          │ You trigger node group    │ AWS auto-replaces nodes   │
  │ (data plane)                │ rolling update after CP   │ after CP upgrade (if      │
  │                             │ upgrade                   │ Auto Mode nodes enabled)  │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Debugging capability        │ High -- SSH, SSM, node    │ Low -- kubectl exec only  │
  │                             │ inspection, tcpdump, etc. │ No node-level access      │
  ├─────────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Best for                    │ Teams that need full      │ Teams that want minimal   │
  │                             │ visibility and control    │ ops burden. Comfortable   │
  │                             │ DaemonSets required       │ with reduced visibility.  │
  │                             │ Custom AMI needed         │ Standard workloads only.  │
  │                             │ Learning EKS (this lab)   │                           │
  └─────────────────────────────┴──────────────────────────┴──────────────────────────┘
```

**Cost example -- 2 nodes running 8 hours/day for 30 days:**

```
  Instance: m5.large (2 vCPU, 8GB RAM)
  EC2 On-Demand price: $0.096/hr (us-east-1)
  EKS Auto Mode fee for m5.large: ~$0.01152/hr (from AWS pricing page)

  Managed Node Group (2 x m5.large):
    EC2 cost:  $0.096 x 2 x 8hr x 30 = $46.08
    Extras:    Karpenter/CA pod CPU/mem on your nodes (~negligible)
    Total:     ~$46/month

  EKS Auto Mode (2 x m5.large):
    EC2 cost:  $0.096 x 2 x 8hr x 30 = $46.08
    Auto fee:  $0.01152 x 2 x 8hr x 30 = $5.53
    Total:     ~$52/month

  Auto Mode premium: ~$5.53/month (~12% more) for these 2 nodes
  At scale (20 nodes, 24/7):
    Auto Mode fee: $0.01152 x 20 x 730hr = $168/month extra
    vs Managed NG: $0 extra (Karpenter pod cost ~$1-2/month negligible)

  The Auto Mode premium vs saving on operational engineering time:
    If Auto Mode saves 4 hours/month of a senior engineer's time at
    $150/hr fully loaded: saves $600/month in labor
    At $168/month fee: net positive for teams with limited platform eng
```

**Can you mix them?**

```
  YES -- you can mix Managed Node Groups and Auto Mode in the same cluster.

  Common pattern:
    Auto Mode NodePool:    general application workloads (80% of pods)
    Managed NG:            DaemonSet-requiring pods (Fluent Bit, Prometheus
                           node-exporter, security agents)
                           Workloads needing custom AMI
                           Workloads needing privileged pods or hostNetwork

  Scheduling:
    Use node selectors or taints to direct pods to the right node type.
    Auto Mode nodes have taint: eks.amazonaws.com/compute-type=auto
    Add toleration to pods that should run on Auto Mode nodes.
    Managed NG nodes have no Auto Mode taint -- DaemonSets run there.
```

---

## Part 10: VPC Design for EKS

VPC design is the most consequential pre-cluster decision you make. Unlike
almost everything else in AWS, VPC CIDR blocks and subnet structures cannot
be changed after creation without significant disruption. Design for growth.

### The Recommended Layout

```
 +------------------------------------------------------------------------+
|                  VPC  10.0.0.0/16  (65,536 IPs)                         |
|                                                                         |
|  +---------------------------------+  +------------------------------+  |
|  |      AZ: us-east-1a             |  |      AZ: us-east-1b          |  |
|  |                                 |  |                              |  |
|  |  +---------------------------+  |  |  +------------------------+  |  |
|  |  |  PUBLIC   10.0.1.0/24     |  |  |  |  PUBLIC  10.0.2.0/24   |  |  |
|  |  |  251 usable IPs           |  |  |  |  251 usable IPs        |  |  |
|  |  |                           |  |  |  |                        |  |  |
|  |  |  What lives here:         |  |  |  |  What lives here:      |  |  |
|  |  |  - Internet-facing ALB    |  |  |  |  - Internet-facing ALB |  |  |
|  |  |  - Internet-facing NLB    |  |  |  |  - Internet-facing NLB |  |  |
|  |  |  - NAT Gateway (AZ-a)     |  |  |  |  - NAT Gateway (AZ-b)  |  |  |
|  |  |  - Bastion host (optional)|  |  |  |                        |  |  |
|  |  |                           |  |  |  |                        |  |  |
|  |  |  What does NOT live here: |  |  |  |                        |  |  |
|  |  |  - Worker nodes           |  |  |  |                        |  |  |
|  |  |  - Pods                   |  |  |  |                        |  |  |
|  |  |  - Databases              |  |  |  |                        |  |  |
|  |  +------------+-------------+   |  |  +-----------+------------+  |  |
|  |               | outbound        |  |              | outbound      |  |
|  |               | via NAT GW      |  |              | via NAT GW    |  |
|  |  +------------v-------------+   |  |  +-----------v------------+  |  |
|  |  |  PRIVATE  10.0.10.0/22   |   |  |  |  PRIVATE 10.0.11.0/22  |  |  |
|  |  |  1,019 usable IPs        |   |  |  |  1,019 usable IPs      |  |  |
|  |  |                          |   |  |  |                        |  |  |
|  |  |  What lives here:        |   |  |  |  What lives here:      |  |  |
|  |  |  - EC2 Worker Nodes      |   |  |  |  - EC2 Worker Nodes    |  |  |
|  |  |  - Pods (VPC CNI IPs)    |   |  |  |  - Pods (VPC CNI IPs)  |  |  |
|  |  |  - Internal ALB / NLB    |   |  |  |  - Internal ALB / NLB  |  |  |
|  |  |  - Cross-ENI (by EKS)    |   |  |  |  - Cross-ENI (by EKS)  |  |  |
|  |  |  - RDS / ElastiCache     |   |  |  |  - RDS / ElastiCache   |  |  |
|  |  +--------------------------+   |  |  +------------------------+  |  |
|  +---------------------------------+  +------------------------------+  |
|                      ^                                                  |
|              Internet Gateway                                           |
+-------------------------------------------------------------------------+
```

> **`[BPG]` Subnets:** "Kubernetes worker nodes can run in the cluster
> subnets, but it is not recommended. During cluster upgrades Amazon EKS
> provisions additional ENIs in the cluster subnets. When your cluster scales
> out, worker nodes and pods may consume the available IPs in the cluster
> subnet. Hence consider using dedicated cluster subnets with /28 netmask."
>
> This means: use dedicated subnets for Cross-ENIs (/28 is fine) and separate
> subnets (/22 or larger) for worker nodes and pods.

**Why worker nodes must be in private subnets:**

Placing worker nodes in public subnets (with public IPs) exposes them directly
to the internet. Any misconfigured security group rule, any vulnerability in
the kubelet or container runtime, any exposed NodePort service becomes directly
reachable from the internet. Private subnets with a NAT Gateway for outbound
traffic give you egress (for pulling images, calling AWS APIs) without inbound
exposure. This is a non-negotiable security baseline.

**Why public subnets are still needed:**

Internet-facing load balancers (ALB, NLB) must have network interfaces in
public subnets to receive traffic from the internet. The load balancer lives
in the public subnet and proxies traffic to pods in the private subnet. This
is the correct traffic path for production: Internet → ALB (public) → Pod (private).

### The IP Exhaustion Problem — Why Private Subnets Must Be Large

This is the most commonly underestimated VPC design mistake for EKS.

EKS uses the **AWS VPC CNI plugin** which assigns a **real VPC IP address to
every pod**. This is fundamentally different from overlay networking used by
other CNI plugins (Flannel, Calico VXLAN, Weave).

```
  Traditional overlay networking (Flannel, Calico VXLAN): -- NOT used by EKS:
  ─────────────────────────────────────────────────────
  Worker node:    10.0.10.5  (1 VPC IP consumed from your subnet)
  Pod 1:          192.168.0.1  (overlay IP -- lives in a tunnel)
  Pod 2:          192.168.0.2  (overlay IP -- lives in a tunnel)
  Pod 3:          192.168.0.3  (overlay IP -- lives in a tunnel)

  VPC IPs consumed: 1 per node, regardless of how many pods run on it
  A /24 subnet (251 IPs) can support 251 nodes with unlimited pods each

  ─────────────────────────────────────────────────────
  AWS VPC CNI (EKS default -- native VPC networking):
  ─────────────────────────────────────────────────────
    Worker node:    10.0.10.5  (1 VPC IP consumed)
    Pod 1:          10.0.10.25 (1 VPC IP consumed -- REAL subnet IP)
    Pod 2:          10.0.10.26 (1 VPC IP consumed -- REAL subnet IP)
    Pod 3:          10.0.10.27 (1 VPC IP consumed -- REAL subnet IP)
    Warm pool IPs:  10.0.10.28, 10.0.10.29 (reserved -- not yet pods)
    VPC IPs/node:   1 + pods + warm pool
    /24 subnet:     only ~18-20 t3.small nodes supported -- NOT 251!
```

> **`[BPG]` VPC CNI:** "Amazon VPC CNI allocates a warm pool of ENIs and
> secondary IP addresses from the subnet attached to the node's primary ENI.
> Please consider VPC and Subnet recommendations before deploying EKS clusters.
> We strongly recommend checking the subnets for available IP addresses."

### How the VPC CNI Warm Pool Works

The VPC CNI does not wait until a pod needs an IP to request one. It
pre-allocates a pool of IPs on each node so that pod startup latency is
minimized. This warm pool is configurable but the defaults consume IPs beyond
what the actual running pod count would suggest.

```
  t3.small: max 3 ENIs, max 4 IPs per ENI = 12 total IPs
  (1 primary ENI + 2 secondary ENIs)
  Node IP uses 1 of the 12 → 11 available for pods
  Max pods per t3.small: 11

  At startup, before any pods are scheduled:
    Node IP:    10.0.10.5  (1 IP used)
    Warm pool:  10.0.10.25, 10.0.10.26  (2 IPs pre-reserved)
    Total consumed immediately: 3

  After pods are scheduled:
    Node IP:    10.0.10.5
    Pod 1:      10.0.10.25
    Pod 2:      10.0.10.26
    Pod 3:      10.0.10.27
    Warm pool:  10.0.10.28, 10.0.10.29  (still 2 pre-reserved)
    Total consumed: 6
```

**CIDR sizing reference:**

```
  Instance type: t3.small
  Max ENIs: 3, Max IPs per ENI: 4
  Max pods per node: (3 x (4-1)) + 2 = 11
  VPC IPs consumed per node (approx): 12-14  (node + pods + warm pool)

   Private subnet /24 (251 IPs):
    ~18 t3.small nodes maximum
    Too small for any real workload -- exhausts quickly

  Private subnet /22 (1,019 IPs):
    ~72 t3.small nodes maximum
    Good for this lab and small-medium production clusters

  Private subnet /20 (4,091 IPs):
    ~293 t3.small nodes maximum
    Good for medium-large production clusters

  Private subnet /18 (16,379 IPs):
    ~1,170 t3.small nodes maximum
    Large enterprise clusters
```
> **EKS Best Practice:** Use **/22 or larger** private subnets. Plan for the
> warm pool overhead on top of your actual pod count. IP exhaustion in a
> private subnet causes pod scheduling failures -- pods get stuck in Pending
> with the event "0/N nodes are available: N node(s) had untolerated taint
> {node.kubernetes.io/not-ready}" -- even when nodes have free CPU and memory.
> This is one of the hardest production problems to diagnose under pressure.

### What Happens When IP Exhaustion Occurs

```
  Symptom visible in kubectl:
    kubectl get pods
    NAME          READY   STATUS    RESTARTS   AGE
    myapp-abc     0/1     Pending   0          5m

    kubectl describe pod myapp-abc
    Events:
      Warning  FailedScheduling   Failed to assign an IP address to the pod.
               Network: failed to allocate for range 0: no IP addresses available

  Root cause: private subnet CIDR exhausted
  Fix: expand VPC CIDR + add larger subnet (disruptive)
       or migrate nodes to a new VPC with larger subnets (very disruptive)

  Prevention: size /22 or larger from the start
```

### CIDR Plan for This Lab Series

```
  VPC CIDR:              10.0.0.0/16

  Public Subnets  (Load Balancers, NAT Gateways -- small is fine):
    us-east-1a:          10.0.1.0/24    (251 IPs)
    us-east-1b:          10.0.2.0/24    (251 IPs)

  Private Subnets  (Worker Nodes, Pods -- must be large):
    us-east-1a:          10.0.10.0/22   (1,019 IPs)
    us-east-1b:          10.0.11.0/22   (1,019 IPs)

  Reserved for future AZ expansion:
    us-east-1c public:   10.0.3.0/24
    us-east-1c private:  10.0.16.0/22   (non-overlapping with above)

  Kubernetes Service CIDR (virtual -- not from VPC):
    10.100.0.0/16        (specified at cluster creation -- Demo-02)
    These are ClusterIP addresses -- they exist only in iptables rules,
    never appear in your VPC routing table or as real ENIs
```

> **`[BPG]` Subnets:** "Plan your VPC and Subnet CIDR carefully. Avoid
> the complexity of using multiple CIDRs in a VPC and CNI custom networking.
> Size your subnets to accommodate future growth in both nodes and pods."

### What "Cluster Subnets" Means -- And Why It Matters

The EKS Best Practices Guide uses the term "cluster subnets" but does not
always define it clearly. Here is the precise definition:

```
  "Cluster subnets" = subnets specified at cluster CREATION time
                      (the --subnets parameter in eksctl, or
                       the subnetIds in the EKS CreateCluster API call)

  These are SEPARATE from "node subnets" = subnets specified when
  creating a Managed Node Group or Self-Managed node group.

  What EKS uses cluster subnets for:
    ONLY for Cross-ENI placement (one Cross-ENI per AZ of cluster subnets)
    Also used for additional ENIs during control plane upgrades

  What EKS does NOT use cluster subnets for:
    Pod IP assignment (VPC CNI uses node subnets for that)
    Worker node EC2 instance placement (node groups specify their own subnets)
    Load balancer placement (ALB Controller uses subnet tags for that)
```

**Two valid approaches for this lab:**

```
  Approach A (what we use in this lab -- simpler):
    Cluster subnets:   priv-1a (10.0.10.0/22) + priv-1b (10.0.11.0/22)
    Node group subnets: SAME -- priv-1a + priv-1b
    Cross-ENIs consume 2 IPs from the shared /22 subnets
    Fine for learning -- 2 IPs out of 1,019 per subnet is negligible

  Approach B (recommended for production):
    Cluster subnets:   priv-cluster-1a (10.0.200.0/28, 11 IPs) +
                       priv-cluster-1b (10.0.201.0/28, 11 IPs)
    These tiny /28 subnets ONLY hold Cross-ENIs (1-2 IPs used)
    Node group subnets: separate priv-1a (10.0.10.0/22) + priv-1b (10.0.11.0/22)
    Pod IPs come from the /22 subnets entirely
    During upgrades: additional temporary Cross-ENIs go into the /28 -- no
    competition with pod IPs

  We will use Approach A in this lab series for simplicity.
  Approach B is noted as the production best practice per [BPG].
```

---

## Part 11: VPC CNI -- Flat Networking In Depth

The AWS VPC CNI (Container Network Interface) is a DaemonSet called `aws-node`
that runs on every worker node. It is responsible for the entire pod IP lifecycle.

### The ENI and Secondary IP Model

Every EC2 instance can have multiple ENIs (Elastic Network Interfaces) attached,
and each ENI can have multiple private IP addresses. The VPC CNI exploits this
to assign real VPC IPs to pods without any overlay networking.

```
  Worker Node  (t3.small)
  +--------------------------------------------------------+
  |                                                        |
  |  Primary ENI (eth0):                                   |
  |    Primary IP:   10.0.10.5   (the node's own IP)       |
  |    Secondary IP: 10.0.10.25  (assigned to Pod 1)       |
  |    Secondary IP: 10.0.10.26  (assigned to Pod 2)       |
  |    Secondary IP: 10.0.10.27  (assigned to Pod 3)       |
  |    Secondary IP: 10.0.10.28  (warm pool -- not a pod)  |
  |                                                        |
  |  Secondary ENI (eth1) -- if more pods need IPs:        |
  |    Primary IP:   10.0.10.30  (ENI attachment IP)       |
  |    Secondary IP: 10.0.10.31  (assigned to Pod 4)       |
  |    Secondary IP: 10.0.10.32  (assigned to Pod 5)       |
  |    ...                                                 |
  |                                                        |
  |  aws-node DaemonSet:                                   |
  |    Calls EC2 API to attach secondary ENIs              |
  |    Calls EC2 API to assign secondary IPs               |
  |    Maintains warm pool for fast pod startup            |
  |    Programs veth pairs and routing for each pod        |
  |    Watches kubelet for pod assignments                 |
  +--------------------------------------------------------+
```

### What Happens When a Pod Starts

```
  1. Scheduler assigns pod to Node A
  2. kubelet on Node A starts pod creation
  3. kubelet calls CNI plugin (aws-node via CNI binary)
  4. aws-node checks warm pool for available IPs
  5. If warm pool has IPs:
       Assigns next available IP (e.g. 10.0.10.28) to pod
       Creates veth pair: one end in pod network namespace,
                          one end in node network namespace
       Programs iptables/routing so traffic to 10.0.10.28
       arrives at the pod's veth interface
       Updates warm pool -- requests more IPs from EC2 API to refill
  6. If warm pool is empty:
       Makes EC2 API call to assign new secondary IP to ENI
       Waits for EC2 API response (~1-2 seconds delay)
       Assigns the new IP to the pod
       Pod startup latency increases by 1-2 seconds
  7. Pod receives IP 10.0.10.28, starts running
```

The warm pool eliminates the EC2 API call latency from the pod startup
critical path. This is why the VPC CNI pre-allocates IPs -- trading a small
amount of IP address waste for consistent sub-second pod startup.

### Flat networking traffic paths

```
  Pod-to-Pod (same node):
    Pod A (10.0.10.25) --> Pod B (10.0.10.26)
    Traffic: veth A --> node bridge --> veth B
    No encapsulation, no overlay, no tunnel
    Latency: microseconds

  Pod-to-Pod (different nodes, same AZ):
    Pod A (10.0.10.25) on Node 1 --> Pod C (10.0.10.45) on Node 2
    Traffic: veth A --> eth0 (10.0.10.5) --> VPC routing --> 10.0.10.45
    Native VPC routing -- same as EC2-to-EC2 in same subnet
    No encapsulation overhead
    Latency: sub-millisecond

  Pod-to-Pod (different nodes, different AZ):
    Pod A (10.0.10.25) on Node 1 (AZ-a) --> Pod C (10.0.11.45) on Node 2 (AZ-b)
    Traffic: VPC cross-AZ routing
    No encapsulation -- native VPC
    Latency: single-digit milliseconds (normal cross-AZ)
    Note: cross-AZ data transfer costs $0.01/GB -- this adds up at scale

  Pod-to-AWS Service (e.g. RDS):
    Pod A (10.0.10.25) --> RDS (10.0.11.200)
    RDS security group sees source IP 10.0.10.25 (the real pod IP)
    No SNAT masquerading -- the pod's identity is preserved
    You can write RDS security group rules that target specific pod IPs
    or pod security groups
```

**Why flat networking matters for security and operations:**

```
  With overlay networking (VXLAN):
    Traffic from Pod A looks like it comes from Node A's IP in VPC flow logs
    RDS security group sees node IP, not pod IP
    Hard to trace which pod generated which traffic
    Security group rules target nodes, not pods

  With VPC CNI (flat):
    VPC Flow Logs show actual pod IPs
    RDS security group can allow specific pod security groups (via EKS SG for pods)
    Kubernetes NetworkPolicy uses real IPs -- Calico/VPC CNI enforces directly
    Easier audit trail -- pod IP = pod identity in all logs
```

### Instance Type Limits on Pod Count

Every EC2 instance type has a maximum ENI count and a maximum IP per ENI count.
The VPC CNI uses these limits to determine how many pods can run on a node.

```
  Formula: max pods = (max ENIs) x (max IPs per ENI - 1) + 2
  The -1 is because one IP per ENI is used as the primary ENI/attachment IP
  The +2 accounts for kube-proxy and VPC CNI pods (exempt from this limit)

  Instance   Max ENIs  Max IPs/ENI  Max Pods (formula)
  ─────────────────────────────────────────────────────
  t3.small   3         4            11
  t3.medium  3         6            17
  t3.large   3         12           36
  m5.large   3         10           29
  m5.xlarge  4         15           58
  m5.2xlarge 4         15           58
  c5.large   3         10           29
  c5.xlarge  4         15           58
```

> **Important:** When you see pods Pending with "too many pods" on a node,
> the limit is not CPU/memory -- it is ENI/IP count. You can override this
> with prefix delegation (VPC CNI feature that assigns /28 blocks instead of
> individual IPs, increasing pod density significantly), but this is an
> advanced topic for later demos.

> **`[BPG]` VPC CNI:** "The VPC CNI add-on consists of the CNI binary
> and the ipamd plugin. The CNI assigns an IP address from the VPC network
> to a Pod. The ipamd manages AWS Elastic Networking Interfaces (ENIs) to
> each Kubernetes node and maintains the warm pool of IPs."

**What happens when IP exhaustion occurs:**

```
  Symptoms you will see:
    kubectl get pods
    NAME          READY   STATUS    RESTARTS   AGE
    myapp-abc     0/1     Pending   0          5m

    kubectl describe pod myapp-abc
    Events:
      Warning  FailedScheduling  0/2 nodes available:
               2 Insufficient pods. (max pods per node exceeded due to IP limit)

    OR:
      Events:
               failed to assign an IP to pod: no available IPs

  Root cause:
    Your private subnet CIDR has no more free IP addresses.
    Even if nodes have free CPU and memory, pods cannot start.

  Fix (disruptive):
    Add a secondary CIDR to the VPC and create new larger subnets.
    This is called VPC CNI custom networking.
    Or: migrate to a new VPC with larger subnets (very disruptive).

  Prevention:
    Size your private subnets at /22 minimum before day one.
    Check available IPs: EC2 --> Subnets --> Available IPv4 addresses column.
```

---

## Part 12: Subnet Tagging

This is one of the most common sources of "why isn't my ALB being created?"
confusion. Subnet tags are not optional decoration -- they are discovery
metadata that the AWS Load Balancer Controller reads to decide where to
place load balancers.

### The Three Required Tags

```
  +------------------------------------------------------------------+
  |  TAG 1: Cluster Discovery (ALL subnets -- public and private)     |
  |                                                                  |
  |  Key:   kubernetes.io/cluster/<cluster-name>                     |
  |  Value: owned                                                    |
  |                                                                  |
  |  Why:   The ALB Controller and EKS itself query subnets by this  |
  |         tag to discover which subnets belong to the cluster.     |
  |         Without it, the controller cannot find any subnets.      |
  |                                                                  |
  |  Value options:                                                  |
  |    owned  = this cluster is the sole owner of this subnet        |
  |    shared = multiple clusters share this subnet                  |
  |             (rare -- use owned for single-cluster setups)        |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  TAG 2: Internet-Facing Load Balancers (PUBLIC subnets only)     |
  |                                                                  |
  |  Key:   kubernetes.io/role/elb                                   |
  |  Value: 1                                                        |
  |                                                                  |
  |  Why:   ALB Controller places internet-facing ALBs and NLBs      |
  |         only in subnets tagged with this key.                    |
  |         Annotation on Ingress: scheme: internet-facing           |
  |         Without this tag: "Could not find any suitable subnets"  |
  +------------------------------------------------------------------+

  +------------------------------------------------------------------+
  |  TAG 3: Internal Load Balancers (PRIVATE subnets only)           |
  |                                                                  |
  |  Key:   kubernetes.io/role/internal-elb                          |
  |  Value: 1                                                        |
  |                                                                  |
  |  Why:   ALB Controller places internal ALBs and NLBs only in     |
  |         subnets tagged with this key.                            |
  |         Annotation on Ingress: scheme: internal                  |
  |         Without this tag: internal LBs also fail to provision    |
  +------------------------------------------------------------------+
```

### The Exact Failure Without Tags

```
  Missing kubernetes.io/role/elb on public subnets:

  kubectl apply -f ingress.yaml
  # Ingress object created but ALB never provisions

  kubectl describe ingress my-ingress
  Events:
    Warning  FailedDeployModel  Failed deploy model due to
             InvalidSubnets: No subnets found for ALB.
             Subnets must have tags: {kubernetes.io/cluster/my-cluster: owned,
             kubernetes.io/role/elb: 1}

  The ingress object exists. The ALB does not. Your app is unreachable.
  There is no error when applying the YAML -- only when describing later.
  This silent failure is the reason for calling it out explicitly.
```

### Tag Application Strategy

```
  Apply tags during VPC creation (not after):
    Adding tags to subnets post-creation requires finding all subnets
    and tagging individually. Easy to miss one in a multi-subnet setup.
    If using eksctl: tags are applied automatically when cluster config
    specifies the VPC -- you do not add them manually.
    If creating VPC manually or with CloudFormation: add tags in the
    subnet resource definition, not as an afterthought.

  eksctl automatically applies:
    kubernetes.io/cluster/<name> = owned  (on all subnets you specify)
    kubernetes.io/role/elb = 1            (on subnets tagged as public)
    kubernetes.io/role/internal-elb = 1   (on subnets tagged as private)

  Manual console creation (Demo-02):
    You will add these tags manually in the subnet configuration step.
    This makes the requirement visible and teaches the underlying mechanic.
    eksctl is used afterward to show the automated equivalent.
```

> **EKS Best Practice:** Apply all three subnet tags before cluster creation.
> The cluster creation process itself reads these tags. If you add them after
> the cluster is created, you may need to restart the ALB Controller pod for
> it to pick up the new tags.

> **`[BPG]` Load Balancing:** Apply all subnet tags before creating the
> cluster. `eksctl` applies these automatically from a cluster config file.
> For manually created VPCs, add tags explicitly to each subnet.

---

## Part 13: API Server Endpoint Modes

This is a security-critical, cluster-creation-time decision. The endpoint
mode controls who can reach the Kubernetes API server and from where.
It directly affects your security posture and operational complexity.

### Three Modes Explained

```
  MODE 1 -- PUBLIC ONLY (AWS default -- not recommended for production)
  ───────────────────────────────────────────────────────────────────────

  kubectl (your laptop):
    Internet --> NLB (public endpoint) --> API Server
    Anyone with the endpoint URL and valid credentials can reach it

  kubelet (worker node):
    HTTPS --> leaves your VPC --> AWS backbone network --> NLB --> API Server
    Kubelet heartbeats, pod specs, logs go OUTSIDE your VPC
    (still on AWS backbone, not open internet, but leaves your VPC)

  Risk: API server endpoint is publicly reachable
        If credentials are compromised, attacker has full internet access
        Kubelet traffic is not confined to your VPC

  ───────────────────────────────────────────────────────────────────────
  MODE 2 -- PUBLIC + PRIVATE (recommended for this lab series)
  ───────────────────────────────────────────────────────────────────────

  kubectl (your laptop):
    Internet --> NLB (public endpoint, RESTRICT TO YOUR IP) --> API Server
    Restrict with: publicAccessCidrs: ["YOUR.IP.ADDRESS/32"]

  kubelet (worker node):
    HTTPS --> Cross-ENI (private, inside your VPC) --> API Server
    Kubelet traffic NEVER leaves your VPC
    No data transfer costs for kubelet-to-API-server traffic

  Benefit: Best of both worlds
    You can use kubectl from your laptop (public endpoint)
    Worker nodes communicate securely within the VPC (private path)
    Restrict the public endpoint to your office/home IP CIDR

  ───────────────────────────────────────────────────────────────────────
  MODE 3 -- PRIVATE ONLY (maximum security)
  ───────────────────────────────────────────────────────────────────────

  kubectl (your laptop):
    MUST be inside your VPC or connected via VPN / Direct Connect
    Cannot reach the API server from the open internet at all

  kubelet (worker node):
    HTTPS --> Cross-ENI (private) --> API Server
    Same as Mode 2 for node traffic

  Use case:
    Regulated industries (PCI-DSS, HIPAA) requiring no internet exposure
    Requires setting up AWS Client VPN or Direct Connect to use kubectl
    No kubectl from laptop without VPN -- significantly more complex
```

### Comparing All Three

```
  +-----------------+----------------+------------------+------------------+
  | Aspect          | Public Only    | Public+Private   | Private Only     |
  +-----------------+----------------+------------------+------------------+
  | kubectl from    | Internet       | Internet         | VPN / DirectConn |
  | laptop          |                | (restrict CIDR)  | required         |
  +-----------------+----------------+------------------+------------------+
  | kubelet traffic | Leaves VPC     | Stays in VPC     | Stays in VPC     |
  +-----------------+----------------+------------------+------------------+
  | API server      | Public internet| Public internet  | Not publicly     |
  | reachable from  | (any IP)       | (your CIDR only) | reachable        |
  +-----------------+----------------+------------------+------------------+
  | Attack surface  | High           | Low              | Minimal          |
  +-----------------+----------------+------------------+------------------+
  | Data transfer   | Kubelet pays   | Kubelet free     | Kubelet free     |
  | cost (kubelet)  | cross-AZ rates | (in-VPC)         | (in-VPC)         |
  +-----------------+----------------+------------------+------------------+
  | Operational     | Simple         | Simple           | Complex          |
  | complexity      |                |                  | (VPN needed)     |
  +-----------------+----------------+------------------+------------------+
  | Used in this    | No             | Yes (all demos)  | No               |
  | lab series      |                |                  |                  |
  +-----------------+----------------+------------------+------------------+
```

> **EKS Best Practice:** Use Public + Private mode with `publicAccessCidrs`
> restricting the public endpoint to your IP address or your organization's
> CIDR range. This gives you convenient kubectl access while ensuring all
> kubelet-to-API-server traffic stays inside your VPC (more secure and avoids
> cross-AZ data transfer charges on kubelet heartbeats).

### How to Restrict the Public Endpoint CIDR

After cluster creation (can also be set at creation time):

```
  AWS Console:
    EKS --> Clusters --> <cluster-name> --> Networking --> Manage networking
    API server endpoint access: Public and private
    Public access CIDRs: 203.0.113.10/32  (your public IP)

  CLI:
    aws eks update-cluster-config \
      --name my-cluster \
      --resources-vpc-config \
        endpointPublicAccess=true,\
        endpointPrivateAccess=true,\
        publicAccessCidrs="203.0.113.10/32"

  Find your public IP:
    curl https://checkip.amazonaws.com
```

### A Common Mistake -- Locking Yourself Out

```
  Scenario:
    You switch to Private Only mode.
    You are working from home, not on VPN.
    kubectl suddenly stops working.

    kubectl get nodes
    --> Unable to connect to the server: dial tcp: connection refused

  Root cause:
    API server is now private. No public endpoint exists.
    Your laptop has no VPN connection to the VPC.

  Fix:
    Go to AWS Console (browser -- not kubectl)
    EKS --> Clusters --> Networking --> Manage networking
    Re-enable public access temporarily
    Set up VPN before switching back to private-only

  Prevention:
    Never switch to Private Only without VPN already configured and tested.
    Always test in Public+Private mode first, verify VPN works, then switch.
```
---

## Part 14: Security Groups in EKS

EKS creates and manages security groups automatically. Understanding which
SG controls which traffic path prevents two of the most common EKS mistakes:
(1) accidentally blocking node-to-control-plane traffic, and (2) not knowing
where to add rules for application traffic.

### The Security Group Hierarchy

```
  Internet
     |
     v
  +----------------------------+
  |   ALB / NLB Security Group |  Created by: AWS Load Balancer Controller
  |   Inbound:  80, 443        |  Applied to: Load balancer ENIs
  |   Outbound: to pods        |  Controls:   internet --> load balancer
  +------------+---------------+
               | to pods
               v
  +----------------------------+
  |  Node Security Group       |  Created by: EKS Managed Node Group
  |  Inbound:  from ALB SG     |  Applied to: EC2 worker node instances
  |            from cluster SG |  Controls:   LB --> node, node outbound
  |  Outbound: 443 to AWS APIs |               to internet (via NAT GW)
  |            all to cluster SG|
  +------------+---------------+
               | to control plane
               v
  +----------------------------------------------+
  |  Cluster Security Group                      |
  |  Name: eks-cluster-sg-<cluster-name>-<id>    |
  |  Created by: EKS (automatically)             |
  |  Applied to: Cross-ENIs + Worker Nodes       |
  |                                              |
  |  Default rules:                              |
  |    Inbound:  all traffic from itself         |
  |              (self-referencing rule)         |
  |    Outbound: all traffic to itself           |
  |              (self-referencing rule)         |
  |                                              |
  |  What this allows:                           |
  |    Worker nodes --> Cross-ENI (API server)   |
  |    Cross-ENI --> Worker nodes (kubelet push) |
  |    Node-to-node traffic (for pod networking) |
  +----------------------------------------------+
               |
               v
  +----------------------------+
  |  Cross-ENIs                |
  |  (owned by EKS, in your   |
  |   private subnet)          |
  +----------------------------+
               |
               v
       Control Plane API Servers
```

### The Cluster Security Group -- Do Not Touch This

The cluster security group (`eks-cluster-sg-<name>-<random-id>`) uses a
**self-referencing rule**: members of the security group can communicate
with other members of the same security group on all ports and protocols.

```
  Cluster SG members:
    - Every worker node EC2 instance (attached automatically by EKS)
    - Every Cross-ENI (attached by EKS when creating the cluster)

  Because all members share the cluster SG, they can all talk to each other:
    Worker node --> Cross-ENI --> API Server:  allowed (both in cluster SG)
    API Server (via Cross-ENI) --> kubelet:    allowed (both in cluster SG)
    Worker node --> Worker node:               allowed (both in cluster SG)
                                               (enables pod-to-pod routing)
```

**What happens if you delete or misconfigure the cluster security group:**

```
  Deleted/misconfigured cluster SG:
    kubelet heartbeats stop reaching the API server
    API server cannot push pod specs to nodes
    Existing pods keep running (they are already started)
    New pod scheduling stops
    kubectl get nodes  shows: STATUS = NotReady (after 40 seconds)
    kubectl get pods   shows: new pods stuck in Pending

  Recovery:
    Re-attach the correct cluster SG to all nodes
    This cannot be done without downtime on the affected nodes
    Prevention: NEVER modify or delete eks-cluster-sg-* security groups
```

> **EKS Best Practice:** Only add application-level security group rules to
> the **Node Security Group** (created by the Managed Node Group). Never
> add or remove rules from the cluster security group. If you need custom
> pod-level networking rules, use Kubernetes NetworkPolicy or the EKS
> Security Groups for Pods feature (covered in a later demo).


> **`[BPG]` Security Groups:** "Amazon EKS creates a cluster security group
> that allows all traffic between the EKS control plane and managed node
> groups. Do not delete or modify this security group -- it is required for
> cluster communication."

---

## Part 15: Complete Architecture -- The Full Target State
This is the complete architecture that all 25 demos in this series
progressively build toward. Every component shown will be introduced,
configured, and demonstrated step by step.

```
                               Internet
                        (Users / kubectl / CI-CD)
                                  |
                                  v
+-----------------------------------------------------------------------+
|                   YOUR VPC  10.0.0.0/16   us-east-1                  |
|                                                                       |
|  PUBLIC SUBNETS  (10.0.1.0/24  and  10.0.2.0/24)                     |
|  +--------------------------------------------------------------+     |
|  | Internet-Facing ALB (Demo-05) / NLB (Demo-04)               |     |
|  +--------------------------------------------------------------+     |
|  NAT-GW-a (AZ-a)                         NAT-GW-b (AZ-b)             |
|                |  outbound internet for nodes/pods/add-ons            |
|  PRIVATE SUBNETS  (10.0.10.0/22  and  10.0.11.0/22)                  |
|  +--------------------------------------------------------------+     |
|  | MANAGED NODE GROUP  (t3.small / 2 AZs / min 2 nodes)        |     |
|  |  Node AZ-a  10.0.10.x          Node AZ-b  10.0.11.x         |     |
|  |  kubelet / kube-proxy           kubelet / kube-proxy         |     |
|  |  VPC CNI (aws-node)             VPC CNI (aws-node)           |     |
|  |  containerd                     containerd                   |     |
|  +--------------------------------------------------------------+     |
|  | PODS (real VPC IPs via VPC CNI)                              |     |
|  | App Pods / ArgoCD / Prometheus / Grafana / KEDA ...          |     |
|  +--------------------------------------------------------------+     |
|  | EKS ADD-ONS  (kube-system namespace)                         |     |
|  | CoreDNS / VPC CNI / kube-proxy / AWS LB Controller          |     |
|  | ExternalDNS / Metrics Server / Cluster Autoscaler            |     |
|  +--------------------------------------------------------------+     |
|  Cross-ENI (AZ-a  10.0.10.4)      Cross-ENI (AZ-b  10.0.11.4)        |
+-----------------------------------------------------------------------+
          |  Cross-ENIs  (kubelet traffic -- private path)
          |  NLB endpoint  (kubectl / CI-CD -- public path)
          v
+-----------------------------------------------------------------------+
|  AWS-MANAGED CONTROL PLANE  (AWS account -- not visible to you)       |
|  kube-apiserver (HA / multi-AZ)                                       |
|  etcd  (3 nodes / 3 AZs / Raft quorum)                                |
|  kube-controller-manager  (active-standby)                            |
|  kube-scheduler  (active-standby)                                     |
|  NLB  (fronts API servers for external clients)                       |
+-----------------------------------------------------------------------+

AWS SERVICES ACROSS DEMOS:
+-----------+ +----------+ +--------------+ +---------+ +------------+
|    IAM    | |   ECR    | |   Secrets    | |   RDS   | | CloudWatch |
|  + IRSA   | | (images) | |   Manager    | |  MySQL  | | + FluentBit|
+-----------+ +----------+ +--------------+ +---------+ +------------+
+-----------+ +----------+ +--------------+ +---------+
|  Route 53 | |   EBS    | |     EFS      | |   SQS   |
| (Demo-19) | | (Demo-07)| |  (Demo-08)   | | (Demo-23|
+-----------+ +----------+ +--------------+ +---------+
```
---

## Key Concepts Summary

**Two VPCs -- always remember:**
```
  AWS-managed VPC  --> Control plane (API server + etcd) -- invisible to you
  Your VPC         --> Worker nodes, pods, load balancers -- you own this
  Bridge           --> Cross-ENIs: AWS-provisioned ENIs in YOUR private subnets
```

**Why Cross-ENIs and not VPC Peering / TGW / PrivateLink:**
```
  VPC Peering:    CIDR overlap breaks at scale + customer must configure routes
  Transit Gateway: Cost ($0.05/hr/attach) + CIDR overlap + overkill + customer config
  PrivateLink:    One-way only (CP cannot initiate to kubelets) + fixed targets
  Cross-ENI:      One IP from your subnet / bidirectional / zero customer config
                  No CIDR constraint / dynamically works for any node in subnet
```

**VPC CNI = real VPC IPs for every pod:**
```
  Every pod consumes one IP from your private subnet CIDR
  t3.small: max 11 pods per node, ~12-14 IPs consumed per node (incl. warm pool)
  Size private subnets at /22 minimum -- /20 or larger for production
  IP exhaustion causes Pending pods even with free CPU and memory
```

**Subnet tagging -- all three required before creating the cluster:**
```
  All subnets:     kubernetes.io/cluster/<name> = owned  (discovery)
  Public subnets:  kubernetes.io/role/elb = 1            (internet-facing LB)
  Private subnets: kubernetes.io/role/internal-elb = 1   (internal LB)
  Missing tags:    ALB/NLB silently fails to provision, no error on kubectl apply
```

**API endpoint mode -- use Public + Private:**
```
  Public Only     --> kubelet traffic leaves VPC, API server open to internet
  Public+Private  --> kubelet stays in VPC, kubectl from internet (restrict CIDR)
  Private Only    --> maximum security, VPN required for kubectl
```

**Cluster security group -- never modify:**
```
  Auto-created by EKS as eks-cluster-sg-<name>-<id>
  Self-referencing rule: all cluster members can talk to each other
  Deleting or modifying rules = nodes lose API server connectivity
  Add application rules to the Node Security Group instead
```


---

## Common Mistakes and How to Avoid Them

```
  MISTAKE 1: Using /24 private subnets
  ─────────────────────────────────────
  Symptom:  Pods stuck in Pending, "no IP addresses available"
            Happens weeks after cluster creation when load increases
  Root cause: VPC CNI exhausted the /24 (251 IPs) with node+pod+warm pool IPs
  Fix:      Migrate to a new, larger VPC (very disruptive)
  Prevention: Use /22 minimum -- size for growth, not current state

  MISTAKE 2: Missing subnet tags
  ─────────────────────────────────────
  Symptom:  kubectl apply -f ingress.yaml -- no error
            kubectl get ingress -- no ADDRESS after 5 minutes
            kubectl describe ingress -- "no suitable subnets found"
  Root cause: kubernetes.io/role/elb tag missing from public subnets
  Fix:      Add the tag, restart ALB Controller pod
  Prevention: Apply tags before cluster creation -- check with eksctl

  MISTAKE 3: Modifying the cluster security group
  ─────────────────────────────────────────────────
  Symptom:  kubectl get nodes shows NotReady
            kubectl describe node shows "node not ready" events
  Root cause: Removed a rule from eks-cluster-sg-* -- broke kubelet connectivity
  Fix:      Re-add the self-referencing allow-all rule to cluster SG
  Prevention: Only modify the Node Security Group for application rules

  MISTAKE 4: Switching to Private Only mode without VPN
  ───────────────────────────────────────────────────────
  Symptom:  kubectl suddenly fails -- "Unable to connect to server"
  Root cause: API server now private, laptop not on VPN
  Fix:      Use AWS Console (browser) to re-enable public access
  Prevention: Set up and test VPN before switching to private-only mode

  MISTAKE 5: Expecting EC2 default metrics without agent for memory/disk
  ───────────────────────────────────────────────────────────────────────
  Symptom:  No mem_used_percent or disk_used_percent in CloudWatch
  Root cause: AWS hypervisor cannot see inside the guest OS
  Fix:      Install CloudWatch Agent on nodes (covered in EKS logging demo)
  Prevention: Understand the two-pipeline model -- hypervisor vs agent

  MISTAKE 6: Assuming Cross-ENIs are customer resources
  ───────────────────────────────────────────────────────
  Symptom:  You see an ENI in your subnet you did not create
            You delete it thinking it is orphaned
            Cluster nodes lose API server connectivity immediately
  Root cause: The ENI is a Cross-ENI managed by EKS
  Fix:      Restore the cluster (severe -- may require recreation)
  Prevention: Never delete ENIs with description containing "Amazon EKS"
              Check: EC2 --> Network Interfaces --> filter by Description
```


```
  MISTAKE 1: Using /24 private subnets
  Symptom:   Pods stuck in Pending -- "no IP addresses available"
             Happens weeks after creation when load increases
  Root cause: VPC CNI exhausted the /24 with node+pod+warm pool IPs
  Fix:       Add secondary CIDR + custom networking (complex + disruptive)
  Prevention: Use /22 minimum before day one -- check VPC CNI docs

  MISTAKE 2: Missing subnet tags
  Symptom:   kubectl apply -f ingress.yaml -- no error
             kubectl get ingress -- no ADDRESS after 5 minutes
             kubectl describe ingress -- "no suitable subnets found"
  Root cause: kubernetes.io/role/elb missing from public subnets
  Fix:       Add the tag, restart ALB Controller pod
  Prevention: Apply all three tags before cluster creation

  MISTAKE 3: Deleting a Cross-ENI
  Symptom:   kubectl get nodes shows NotReady for an entire AZ
             kubectl describe node shows "connection refused to API server"
  Root cause: Cross-ENI deleted from EC2 Network Interfaces console
              All nodes in that AZ lost control plane connectivity
  Fix:       EKS may auto-recover -- if not, cluster recreation required
  Prevention: NEVER delete ENIs with "Amazon EKS" in the Description field

  MISTAKE 4: Modifying the cluster security group
  Symptom:   Nodes suddenly NotReady, no recent events logged
             kubelet heartbeats stop reaching API server
  Root cause: Removed the self-referencing rule from eks-cluster-sg-*
  Fix:       Re-add the self-referencing allow-all rule
  Prevention: Only add rules to the Node Security Group for app traffic
              Never touch eks-cluster-sg-* rules

  MISTAKE 5: Switching to Private Only mode without VPN
  Symptom:   kubectl suddenly fails -- "Unable to connect to server"
  Root cause: API server now private, laptop not on VPN
  Fix:       Use AWS Console (browser) to re-enable public access
  Prevention: Set up and test VPN before switching to private-only mode

  MISTAKE 6: Running worker nodes in public subnets
  Symptom:   No immediate failure -- but nodes have public IPs
  Root cause: Nodes placed in public subnets during cluster creation
  Fix:       Rebuild node group in private subnets
  Prevention: Always place worker nodes in private subnets only
```

---

## EKS Best Practices Summary for This Demo

All sourced from the [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

| # | Best Practice                                 | Impact of Ignoring                              |
|---|-----------------------------------------------|-------------------------------------------------|
| 1 | Subnets in at least 2 AZs                    | Single AZ failure isolates all nodes            |
| 2 | Worker nodes in private subnets              | Nodes directly exposed to internet attacks      |
| 3 | /22 or larger private subnets                | IP exhaustion causes pod scheduling failures    |
| 4 | Apply subnet tags before cluster creation    | ALB/NLB silently fails to provision             |
| 5 | Public+Private API endpoint mode             | Either security risk or operational complexity  |
| 6 | Restrict public endpoint to your CIDR        | API server reachable from any internet IP       |
| 7 | Never modify the cluster security group      | Nodes lose control plane connectivity           |
| 8 | Use Managed Node Groups                      | Manual node lifecycle management burden         |
| 9 | enableDnsHostnames + enableDnsSupport in VPC | Node registration and CoreDNS resolution fails  |
|10 | Never delete Cross-ENIs                      | Immediate loss of node-to-API-server connectivity|
|11 | Never use access keys on worker nodes        | Credential leak risk -- use IAM roles (IRSA)    |
|12 | NAT Gateway per AZ (not shared)             | AZ failure takes down all outbound traffic      |

| # | Best Practice | Source | Impact of Ignoring |
|---|---|---|---|
| 1 | Subnets in at least 2 AZs | `[BPG]` Networking | AZ failure isolates all nodes |
| 2 | Worker nodes in private subnets | `[BPG]` Security | Nodes exposed to internet |
| 3 | /22 or larger private subnets | `[BPG]` VPC CNI | IP exhaustion -- Pending pods |
| 4 | Apply subnet tags before cluster creation | `[BPG]` Load Balancing | ALB/NLB silently fails |
| 5 | Public+Private API endpoint + CIDR restrict | `[BPG]` Cluster Access | API server open to internet |
| 6 | Never modify the cluster security group | `[BPG]` Security Groups | Nodes lose control plane connectivity |
| 7 | Never delete Cross-ENIs | `[BPG]` Networking | Entire AZ loses API server connectivity |
| 8 | Use Managed Node Groups (or Auto Mode) | `[BPG]` Reliability | Manual node lifecycle burden |
| 9 | One NAT Gateway per AZ | `[BPG]` Reliability | AZ failure = loss of outbound internet |
| 10 | enableDnsHostnames + enableDnsSupport = true | `[BPG]` Networking | Node registration and CoreDNS fail |
| 11 | Min 2 replicas per critical Deployment | `[BPG]` Reliability | Single node failure = service outage |
| 12 | Stay on standard support Kubernetes version | `[BPG]` Upgrades | 6x cost increase after 14 months |

---

## Cost Estimate

| Resource        | Cost              | Notes                                              |
|-----------------|-------------------|----------------------------------------------------|
| This demo       | **$0.00**         | Theory only -- no resources created                |
| VPC             | **$0.00**         | VPC itself is always free                          |
| Internet Gateway| **$0.00**         | IGW is free -- you pay for data through it         |
| NAT Gateway     | $0.045/hr + data  | $32/month per NAT GW if left running -- DELETE     |
| EKS Cluster     | $0.10/hr          | No Free Tier -- ~$2.40/day -- DELETE after each lab|
| EC2 t3.small x2 | $0.0416/hr total  | 2 nodes combined -- ~$1/day                        |
| Cross-ENIs      | $0.00             | Included in EKS pricing                            |

**Estimated cost for a 3-hour lab session (starting from Demo-02):**

```
  EKS control plane:  $0.10/hr x 3hr = $0.30
  2x t3.small nodes:  $0.042/hr x 3hr = $0.13
  NAT Gateway:        $0.045/hr x 3hr = $0.14  (plus data transfer)
  Total:              ~$0.60 per 3-hour session
```

> **Cost tips:**
> - Delete the cluster, NAT Gateway, and nodes after every session
> - Use the cleanup steps at the end of each demo religiously
> - Set up AWS Budgets alert at $10 to catch unexpected charges
> - With $100 credit: ~160 lab sessions of 3 hours each if cleaned up


## Data Transfer Costs in EKS -- A Complete Breakdown

Data transfer costs are the most common "surprise" on an EKS bill. The control
plane fee is visible. EC2 instance costs are predictable. Data transfer costs
are invisible until the bill arrives and can add 20-40% to your total spend.

```
  DATA TRANSFER COST REFERENCE TABLE (us-east-1 region, May 2026)
  ─────────────────────────────────────────────────────────────────
  Traffic type                              Cost        Notes
  ─────────────────────────────────────────────────────────────────
  Pod-to-Pod (same node)                    FREE        veth pair, no network
  Pod-to-Pod (same AZ, different nodes)     FREE        VPC intra-AZ
  Pod-to-Pod (different AZ)                 $0.01/GB    each direction charged
  Node-to-Node (same AZ)                    FREE        VPC intra-AZ
  Node-to-Node (different AZ)               $0.01/GB    each direction charged
  kubelet to API server (Public+Private)    FREE        Cross-ENI = in-VPC
  kubelet to API server (Public Only)       FREE        no data transfer charge
                                                        (stays on AWS backbone)
  Pod to AWS service (same region, VPC EP)  FREE        VPC Endpoint = in-VPC
  Pod to AWS service (same region, NAT GW)  $0.045/GB   NAT GW processing fee
                                            + $0.09/GB  data transfer out
  Pod to internet egress                    $0.09/GB    internet data out
  Internet to ALB/NLB ingress               FREE        inbound is free
  ALB to Pod (same AZ)                      FREE        intra-AZ
  ALB to Pod (different AZ)                 $0.01/GB    cross-AZ
  ECR image pull (same region, NAT GW)      $0.045/GB   NAT processing
                                            + $0.09/GB  data out from ECR
  ECR image pull (same region, VPC EP)      FREE        via Interface Endpoint
  Cross-AZ etcd replication (control plane) FREE        absorbed in $0.10/hr
  ─────────────────────────────────────────────────────────────────
```

### The Cross-AZ Problem -- Where the Money Hides

```
  Scenario: 2-AZ cluster, Deployment with 2 replicas (one per AZ)
  Application: Service A calls Service B for every request

  Service A pod is in AZ-a  (10.0.10.25)
  Service B pod is in AZ-b  (10.0.11.45)

  By default, kube-proxy routes to ANY healthy pod regardless of AZ.
  Every A-->B call crosses an AZ boundary.

  Traffic per day:    1 million requests x 10KB average = 10GB/day
  Cross-AZ cost:      10GB x $0.01 x 2 directions = $0.20/day
  Monthly:            ~$6/month for this one service pair

  In a microservices cluster with 50 services each calling others:
  Monthly cross-AZ:   potentially $300-500/month on data transfer alone
  This is on top of EC2, NAT GW, and control plane costs.
```

### How to Minimize Cross-AZ Costs

```
  1. Topology Aware Routing (Kubernetes built-in)
     Routes Service traffic preferentially to pods in the same AZ.

     kubectl annotate service my-service \
       service.kubernetes.io/topology-mode=Auto

     Effect: kube-proxy prefers same-AZ endpoints when available.
     Not a hard guarantee -- falls back to any AZ if no local pod exists.

  2. Pod Affinity / Topology Spread
     Co-locate frequently communicating pods in the same AZ.

     spec:
       topologySpreadConstraints:
       - maxSkew: 1
         topologyKey: topology.kubernetes.io/zone
         whenUnsatisfiable: DoNotSchedule
         labelSelector:
           matchLabels:
             app: my-app

  3. VPC Endpoints for AWS Services (instead of NAT Gateway)
     Instead of: Pod --> NAT GW ($0.045/GB) --> internet --> ECR
     Use:        Pod --> VPC Endpoint (FREE) --> ECR

     Saves: NAT GW processing fee ($0.045/GB) AND data transfer out ($0.09/GB)
     Cost of Interface VPC Endpoint: $0.01/hr per AZ (~$7/month per service)
     Break-even: ~70GB/month per service where VPC Endpoint beats NAT GW

  4. One NAT Gateway Per AZ (avoid cross-AZ NAT)
     If your AZ-b node uses a NAT Gateway in AZ-a:
       AZ-b node --> AZ-a NAT GW = cross-AZ transfer ($0.01/GB)
       + NAT GW processing ($0.045/GB)
     With NAT GW per AZ:
       AZ-b node --> AZ-b NAT GW = intra-AZ (FREE)
       + NAT GW processing ($0.045/GB)
     One NAT GW per AZ costs $0.09/hr more ($65/month) but eliminates
     cross-AZ data transfer charges on all node outbound traffic.
     [BPG] recommends one NAT GW per AZ for this exact reason.
```

### Complete Cost Breakdown for a 3-Hour Lab Session

```
  Demo-02 onwards (cluster running with 2 t3.small nodes, 2 AZs):

  Fixed hourly costs:
    EKS control plane:          $0.10/hr
    2x t3.small EC2:            $0.042/hr
    2x NAT Gateways:            $0.090/hr  ($0.045 each)
  Total fixed/hr:               $0.232/hr

  3-hour session:               $0.70

  Variable costs (data transfer):
    Image pulls (Docker Hub):   ~50MB per node = 100MB = ~$0.004 NAT
    AWS API calls from nodes:   minimal (<1MB) via NAT = negligible
    Pod-to-pod cross-AZ:        negligible in lab (low traffic)
  Variable total for 3hr lab:   <$0.01

  Total 3-hour lab:             ~$0.71
  With $100 credit:             ~140 full lab sessions before credit runs out
```

> **`[BPG]` Cost Optimization Networking:** "Cross-AZ data transfer between
> pods costs $0.01/GB in each direction. In a busy cluster, this adds up fast.
> Use topology-aware routing and VPC Endpoints to minimize these charges."


---

## Lessons Learned

**The Two-VPC Model is a Security and Managed-Service Architecture Decision**

The control plane lives in an AWS-owned VPC not because of technical necessity
but because the separation protects you from accidentally modifying it and
removes the operational burden of managing it. It is both a security boundary
and a service delivery mechanism.

**Cross-ENIs Solve a Problem No Other VPC Connectivity Option Can**

VPC Peering and Transit Gateway fail on CIDR overlap at scale. PrivateLink
fails on bidirectionality. Cross-ENIs pass all five requirements because they
are not a VPC-to-VPC connection -- they are one IP address from your existing
subnet attached as an interface on the control plane side. Minimum viable
surface area, zero configuration, no cost.

**The NLB and Cross-ENIs Serve Different Audiences Simultaneously**

The NLB is for external API clients (kubectl, CI-CD). Cross-ENIs are for
internal cluster traffic (kubelets, kube-proxy, VPC CNI). Both are always
present. In Public+Private mode they operate in parallel on every cluster.

**All Four Control Plane Components Are Always Running -- You Just Cannot See Them**

The API server, etcd, Controller Manager, and Scheduler run continuously in
the AWS-managed VPC behind the $0.10/hr fee. When you run `kubectl apply`,
you touch all four in sequence. Understanding what each does turns debugging
from guesswork into a systematic trace.

**VPC CNI Changes the IP Addressing Math Completely**

Every engineer coming from non-AWS Kubernetes experience expects pod IPs to
be virtual overlay addresses. On EKS they are real VPC IPs. This single fact
drives the /22 subnet requirement, the CIDR sizing tables, and the IP
exhaustion failure mode. Internalize this on day one.

**EKS Auto Mode Shifts the Responsibility Boundary Further Right**

Traditional EKS: AWS owns the control plane, you own the data plane.
EKS Auto Mode: AWS owns both. If you want to minimize operational overhead
and are comfortable with Bottlerocket-only, no-SSH, no-DaemonSet nodes, Auto
Mode is a legitimate production choice. The ~$0.01/hr per node management fee
is often worth the reduced operational burden for teams without dedicated
platform engineers.

**High Availability is Layered -- AWS Handles Some Layers, You Handle Others**

AWS handles: API server HA (multi-AZ, NLB), etcd HA (3-node Raft, 3-AZ),
controller and scheduler failover (leader election). You handle: Cross-ENI
HA (use 2+ AZs), NAT Gateway HA (one per AZ), node HA (multi-AZ node group),
pod HA (min 2 replicas, PodDisruptionBudgets, topology spread). Neither side
can substitute for the other.

---

## What's Next

**Demo-02: Create the VPC + EKS Cluster + Managed Node Group**

You will build the exact VPC designed in this demo -- two public subnets,
two private subnets, Internet Gateway, two NAT Gateways (one per AZ for HA),
correct route tables, and all six subnet tags. Then create the EKS cluster
and Managed Node Group using the AWS Console first (to see every option) and
`eksctl` second (config-as-code approach). You will see the Cross-ENIs appear
in your subnet after cluster creation, connect kubectl, and verify every node
component is running.

**Demo-02b: EKS Cluster Types -- Production Setup Patterns** *(new)*

A dedicated demo walking through the four major production cluster patterns:
Standard (Managed Node Groups), Auto Mode, Fargate-only, and Mixed
(Managed NG + Fargate + Auto Mode). Each pattern is created, explored at the
VPC and Kubernetes component level, tested, verified, and torn down. This demo
answers: which cluster type is right for which organization and workload?


---

## Quick Reference

| Concept | Key fact |
|---|---|
| Control plane cost | $0.10/hr standard, $0.60/hr extended (after 14 months) |
| API servers | Min 2, across distinct AZs, behind NLB |
| etcd nodes | 3 nodes, 3 AZs, Raft (quorum = 2 of 3) |
| Cross-ENI count | One per AZ you provide at cluster creation |
| Cross-ENI sharing | ALL nodes in same AZ share the SAME Cross-ENI |
| NLB purpose | External clients (kubectl, CI-CD) --> API server |
| Cross-ENI purpose | Internal cluster traffic (kubelets) --> API server |
| Why not VPC Peering | CIDR overlap at scale + customer must configure routes |
| Why not Transit GW | $0.05/hr per attach + CIDR overlap + overkill |
| Why not PrivateLink | One-way only -- control plane cannot initiate to kubelets |
| Pod IPs | Real VPC IPs from your subnet CIDR (VPC CNI) |
| Max pods t3.small | 11 pods per node |
| VPC IPs per t3.small | ~12-14 (node + pods + warm pool) |
| Private subnet minimum | /22 (1,019 IPs) |
| Cluster discovery tag | kubernetes.io/cluster/<name> = owned |
| Internet-facing LB tag | kubernetes.io/role/elb = 1 (public subnets) |
| Internal LB tag | kubernetes.io/role/internal-elb = 1 (private subnets) |
| Recommended endpoint | Public + Private with publicAccessCidrs restriction |
| Node placement | Private subnets only |
| Recommended data plane | Managed Node Groups (this series) or Auto Mode |
| Min AZs | 2 (for Cross-ENI HA and etcd quorum resilience) |
| EKS Auto Mode | EC2 Managed Instances + managed Karpenter + Bottlerocket |
| Auto Mode extra cost | EC2 price + management fee (~$0.01/hr per node) |
| Auto Mode max node age | 21 days -- auto-replaced by AWS |
| VPC DNS requirements | enableDnsHostnames + enableDnsSupport = true |
| Cluster SG rule | Self-referencing -- never modify or delete |