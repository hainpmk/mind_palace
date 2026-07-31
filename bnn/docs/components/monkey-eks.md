# monkey-eks

- **Cloud**: AWS, `ap-southeast-1`
- **Management state**: manual — created via `eksctl` (confirmed via launch template names `eksctl-monkey-eks-nodegroup-*`), no Terraform/Helm management found. Slated for gradual adopt-on-touch management (Terraform/Helm/Ansible) per [ADR-0001](../adr/0001-adopt-on-touch-migration.md) — reference existing resources via data sources first, formally import only when there's a real reason to change a given piece.
- **Criticality**: High — this is the live production EKS cluster (see [mongodb.md](mongodb.md) for confirmed consumers running here)
- **Topology/mode**: 2 node groups (`ng-standard-4`, `ng-standard-8`), both running only in the private subnets
- **Owner/contact**: TBD
- **Dependencies**: see [mongodb.md](mongodb.md) — 37+ workloads here connect to `vm-mongo-master` (Azure)
- **Open questions**:
  - **EKS API server endpoint is publicly accessible** (`endpointPublicAccess: true`, `publicAccessCidrs: ["0.0.0.0/0"]`) — the Kubernetes control plane API itself is open to the internet, not just app-tier services. Separate issue from the Mongo migration work, same family as other exposure findings in this project — worth a security pass independent of any other timeline.

## Network (2026-07-30 discovery)

Confirmed via `aws eks describe-cluster`, `aws ec2 describe-subnets`, `aws eks describe-nodegroup`, `aws ec2 describe-launch-template-versions` — not assumed.

- **VPC**: `vpc-02f43d05b165c22c2`
- **Subnets**:

| Subnet ID | AZ | CIDR | Name | Public |
|---|---|---|---|---|
| `subnet-07c63dc267d112389` | ap-southeast-1a | 10.20.1.0/24 | `prod-public-1a` | yes |
| `subnet-084ddeb295f48b715` | ap-southeast-1b | 10.20.2.0/24 | `prod-public-1b` | yes |
| `subnet-0a442399274fa65ca` | ap-southeast-1a | 10.20.10.0/24 | `prod-private-1a` | no |
| `subnet-0c765ee85c0b56e36` | ap-southeast-1b | 10.20.11.0/24 | `prod-private-1b` | no |

Both node groups run only in the two private subnets (`prod-private-1a`, `prod-private-1b`) — new EC2 workloads meant to be reachable by the cluster (e.g. the mongo migration target) should also land in these private subnets, not the public ones.

- **Security groups**:
  - `sg-078e95cae35f029b2` (`eks-cluster-sg-monkey-eks-*`) — EKS-managed, applied to control-plane ENIs and managed workloads.
  - `sg-05fb3bf373c9eef7a` (`prod-app-sg`) — explicitly described as **"EKS Node + EC2 app tier"**. This is the intended shared security group for EC2 resources that need to talk to EKS workloads (e.g. the new mongo EC2 node) — EC2-and-EKS coexistence in one SG is an already-established pattern here, not something newly introduced.
- **NAT gateways** (2, one per AZ, in this VPC): `3.0.234.236`, `18.141.88.220` — these are the source IPs seen connecting to `vm-mongo-master` from this cluster (see [mongodb.md](mongodb.md)).
- **No AWS-side bastion exists yet** (checked via `aws ec2 describe-instances`/`describe-security-groups` for anything bastion-named — none found). One is being scaffolded as part of the mongo migration's IaC work (see `iac` repo, `modules/bastion`) — devs reach the new mongo node through it via **AWS SSM Session Manager** (IAM-based access, no inbound SSH port, no CIDR allowlist to maintain), not direct SSH+CIDR. Chosen specifically because engineer IPs are dynamic/remote — a static CIDR allowlist would break routinely.
