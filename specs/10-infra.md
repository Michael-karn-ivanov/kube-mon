# WS-10 — AWS Infrastructure (Terraform)

Read first: `00-overview.md`, then `01-contracts.md` (especially §9, the Terraform interface).

## 1. Mission
Write the Terraform that turns an empty AWS account into an EKS cluster ready for workloads. That means 3 nodes across 3 AZs, ECR repositories, working `LoadBalancer` Services (through the AWS Load Balancer Controller), working EBS-backed PVCs, and the namespaces `time-api` and `monitoring`.

## 2. You own
`infra/terraform/**`. Do not create or edit files anywhere else.

## 3. Deliverables
```
infra/terraform/
├── Makefile                # fmt-check, validate, lint (never apply)
├── .tflint.hcl             # aws ruleset enabled
├── README.md               # what each stack creates, inputs/outputs, how to run standalone
├── VERSIONS.md             # Terraform, providers, modules, EKS version, add-on versions, LBC chart
├── HANDOFF.md
├── bootstrap/
│   ├── versions.tf  main.tf  variables.tf  outputs.tf
├── cluster/
│   ├── versions.tf  backend.tf  providers.tf  variables.tf  outputs.tf
│   ├── vpc.tf  eks.tf  ecr.tf  iam.tf
│   ├── terraform.tfvars.example
│   └── policies/aws-load-balancer-controller.json   # vendored at the pinned LBC version
└── platform/
    ├── versions.tf  backend.tf  providers.tf  variables.tf  outputs.tf
    ├── remote_state.tf  lbc.tf  storage.tf  namespaces.tf
```

## 4. Requirements

### 4.1 General
- INF-1: Terraform ≥ 1.11. Pin the AWS provider and every module **exactly** (see contracts §10). Use `terraform-aws-modules/vpc/aws` and `terraform-aws-modules/eks/aws` at their latest major versions.
- INF-2: The input and output variables are **exactly** as in contracts §9. You may add optional variables with defaults, but you must not rename anything.
- INF-3: Use `default_tags` on the AWS provider with `var.tags`.
- INF-4: No hard-coded account IDs, regions or AZs. Discover AZs with `data.aws_availability_zones` and take the first 3.

### 4.2 `bootstrap/`
- INF-5: An S3 bucket named `${bucket_prefix}-${account_id}-${region}`, with versioning, SSE-S3, all public access blocked, `BucketOwnerEnforced` ownership, and `force_destroy = false`. It uses local state.
- INF-6: Output `state_bucket_name`.

### 4.3 `cluster/`
- INF-7 **VPC**: CIDR `10.40.0.0/16`, 3 AZs, a `/20` private and a `/24` public subnet per AZ, NAT gateway(s) driven by `var.single_nat_gateway`, and DNS hostnames enabled. Subnet tags: public `kubernetes.io/role/elb=1`, private `kubernetes.io/role/internal-elb=1`.
- INF-8 **EKS**: `name = var.cluster_name`, `kubernetes_version = var.cluster_version`, and `authentication_mode = "API"`.
  - Give the cluster creator admin rights (`enable_cluster_creator_admin_permissions = true`).
  - Each ARN in `var.cluster_admin_arns` gets an access entry with `AmazonEKSClusterAdminPolicy`, cluster scope.
  - Endpoint: public access on with `public_access_cidrs = var.operator_cidrs`, and private access on.
  - Control-plane logging: `api` and `audit`, with 7-day retention.
- INF-9 **Managed node group `app`**:
  - `min = max = desired = var.node_count`, `instance_types = [var.node_instance_type]`, AMI type `AL2023_x86_64_STANDARD`, private subnets in all 3 AZs.
  - `labels = {workload = "app"}`, no taints.
  - Root volume: 30 GiB gp3, encrypted.
  - Metadata options: `http_tokens = "required"`, `http_put_response_hop_limit = 1`.
  - `update_config.max_unavailable = 1`.
- INF-10 **EKS add-ons**, managed, using the most recent version compatible with `cluster_version`: `vpc-cni`, `coredns`, `kube-proxy`, `eks-pod-identity-agent`, `aws-ebs-csi-driver`. The EBS CSI driver uses an **EKS Pod Identity** association (`kube-system/ebs-csi-controller-sa`) with an IAM role carrying the AWS-managed `AmazonEBSCSIDriverPolicy`. Write the resolved add-on versions into `VERSIONS.md`.
- INF-11 **ECR**: repositories `time-api` and `node-agent`, with `image_tag_mutability = "IMMUTABLE"`, scan on push, `force_delete = true` (dev only, so teardown works with images present), and a lifecycle rule keeping the last 20 images.
- INF-12 **IAM for the AWS Load Balancer Controller**: a role using the vendored policy `policies/aws-load-balancer-controller.json`, at the **same** version as the controller chart in platform, plus a Pod Identity association for `kube-system/aws-load-balancer-controller`.
- INF-13 **Outputs** exactly as in contracts §9. `kubeconfig_command` is `aws eks update-kubeconfig --name <name> --region <region> --alias <name>`.

### 4.4 `platform/`
- INF-14: Read the cluster outputs with `terraform_remote_state` (S3, `bucket = var.state_bucket`, key `cluster/terraform.tfstate`). Configure the `kubernetes` and `helm` providers using `exec` auth (`aws eks get-token`), so no token is stored in state.
- INF-15 **AWS Load Balancer Controller**: a `helm_release` of chart `eks/aws-load-balancer-controller`, pinned, in `kube-system`. Values: `clusterName`, `region`, `vpcId`, `serviceAccount.name = aws-load-balancer-controller`, `replicaCount = 2`. Wait for the release to be ready.
- INF-16 **StorageClass `gp3`**:
  - provisioner `ebs.csi.aws.com`, parameters `type=gp3` and `encrypted=true`
  - `volumeBindingMode = WaitForFirstConsumer`, `reclaimPolicy = Delete`, `allowVolumeExpansion = true`
  - annotated as the default class
  - remove the default annotation from `gp2`, using `kubernetes_annotations` with `force = true`
- INF-17 **Namespaces** `time-api` and `monitoring`, labelled `app.kubernetes.io/part-of=time-telemetry`.
- INF-18: Output `storage_class_name`.

### 4.5 Quality gates
- INF-19: `make -C infra/terraform fmt-check validate lint` passes. `validate` runs `terraform init -backend=false && terraform validate` in each stack.
- INF-20: `trivy config infra/terraform` (or `checkov`) shows no HIGH or CRITICAL findings, except those you accept. Each accepted finding gets an inline skip comment with a reason, for example the public EKS endpoint, which is restricted by CIDR.

## 5. Interfaces you provide
- The stack paths, the backend layout, the variables and the outputs in contracts §9. WS-50's Makefile depends on them.
- After `platform` applies, the cluster guarantees:
  - `kubectl get nodes -l workload=app` shows 3 Ready nodes in 3 AZs
  - `kubectl get sc` lists `gp3 (default)`
  - `kube-system/aws-load-balancer-controller` is Available
  - the namespaces `time-api` and `monitoring` exist

## 6. Standalone testing (no other workstream needed)
1. **Without AWS credentials** (required): run `make -C infra/terraform fmt-check validate lint` and the trivy scan.
2. **With credentials**, only if your launch prompt explicitly says you may apply. The default is **not** to apply: `terraform plan` is the furthest you may go.
   ```bash
   cd infra/terraform/bootstrap && terraform init && terraform apply
   printf 'bucket = "%s"\nregion = "eu-west-2"\n' "$(terraform output -raw state_bucket_name)" > ../backend.hcl
   cd ../cluster   && terraform init -backend-config=../backend.hcl && TF_VAR_operator_cidrs='["<ip>/32"]' terraform apply
   cd ../platform  && terraform init -backend-config=../backend.hcl && TF_VAR_state_bucket=<bucket> terraform apply
   ```
   Then check §5. To prove the platform works, deploy a throwaway `nginx` Deployment with a `type: LoadBalancer` Service carrying the annotations in `20-time-api.md` §4.6, and a 1 Gi PVC. Both must become ready. Then delete them, and destroy in reverse order.

## 7. Definition of Done
- [ ] All three stacks exist, with the inputs and outputs matching contracts §9 exactly.
- [ ] `fmt-check`, `validate`, `lint` and trivy are clean (or the accepted exceptions are documented).
- [ ] `VERSIONS.md` lists Terraform, the AWS/helm/kubernetes providers, both modules, `cluster_version`, every add-on version, and the LBC chart and policy version.
- [ ] `README.md` documents each stack, every variable, and the standalone commands.
- [ ] If an apply was done: the §5 guarantees are verified, the throwaway LB and PVC test passed, destroy was clean, and the output is recorded in `HANDOFF.md`.
- [ ] `HANDOFF.md` is written (contracts §11).

## 8. Out of scope
Building images, Helm charts for the app, agent or monitoring, the top-level Makefile, CI, a DNS or TLS ingress, and autoscalers.

## 9. Gotchas
- Don't configure the helm and kubernetes providers in `cluster/`, because the cluster doesn't exist yet at plan time. That is the reason `platform/` is a separate stack.
- If the LB Controller is uninstalled before the `LoadBalancer` Services are deleted, the NLBs are orphaned and the VPC won't delete. WS-50's teardown deletes the Services first. Don't add `depends_on` tricks that try to fix this inside Terraform.
- A managed node group upgrade briefly runs a 4th node, so the account needs at least 8 vCPUs of On-Demand Standard quota.
