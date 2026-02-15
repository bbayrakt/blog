---
date:
  created: 2026-02-15
---

# Chicken or the Egg?

When it comes to bootstrapping and managing Kubernetes clusters on public cloud platforms such as AWS and Azure, there seems to be a few schools of thought, especially when it comes to using Infrastructure as Code. While it's possible to deploy and manage everything manually through their respective consoles, teams will most likely want instead to use IaC and have version controlled code that can deploy and upgrade their clusters. 

## The Vendor Lock-in Dilemma

AWS, for example, provides [eksctl](https://docs.aws.amazon.com/eks/latest/eksctl/what-is-eksctl.html) for this purpose. While it might save you some time in terms of writing your own [Terraform](https://developer.hashicorp.com/terraform)/[OpenTofu](https://opentofu.org/) code in the beginning, you'll have to learn a tool that only works for AWS EKS, which is a situation that I always try to avoid myself. Why purposefully invest my time into a tool that will inevitably bring us one step closer to vendor-locking us into one cloud platform? If you are already working with public cloud platforms, you are already most likely working with Terraform, so I personally don't see the need to add *yet another* tool on top.

I have seen some people prefer to stay as Kubernetes-native as possible, and use tools such as [Crossplane](https://www.crossplane.io/), [ACK](https://aws-controllers-k8s.github.io/docs/intro) or [ASO](https://azure.github.io/azure-service-operator/) to deploy and manage multiple clusters - all of which already require you to *already* have a functioning Kubernetes cluster to begin with. I still think that these are wonderful tools that I use, but I wouldn't use them to manage Kubernetes clusters, but to rather build components for developers to use which require the creation of public cloud resources outside of the cluster instead. You could also say that I have an unshakable fear of not seeing what will get changed when I apply resources using Crossplane, unlike Terraform, on such crucial pieces of infrastructure in production.

## The Helm CRD Upgrade Problem

And after you have bootstrapped your cluster, you will most likely want to install [Helm](https://helm.sh/) charts - this could be anything from [Karpenter](https://karpenter.sh/), to provide smart auto-scaling for your cluster, to [GitHub ARC](https://docs.github.com/en/actions/concepts/runners/actions-runner-controller), which provides you with self-hosted scale set runners for GitHub Actions. You can install your Helm charts using the [Terraform Helm provider](https://registry.terraform.io/providers/hashicorp/helm/latest/docs), but after the initial bootstrapping of your cluster, when it comes to upgrading the Helm charts in your cluster, you will painfully learn the important lesson that Helm does *not* upgrade CRDs for you, besides [installing CRDs on an initial installation](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/).

In addition, when it comes to bootstrapping a cluster from scratch *and* installing Helm charts or Kubernetes manifests to the cluster, you will quickly realize that it is not possible to do it all in one Terraform project. For example, for the Terraform Kubernetes provider and the Helm provider, you'll need to point the provider to your cluster credentials, which do not exist yet. Or, if you are installing a resource that expects a CRD to already exist, then Terraform will fail in the planning phase. This is a common challange with declarative IaC tools, and not just purely a Terraform issue.

To give a more concrete example, suppose that you want to install [cert-manager](https://cert-manager.io/) and also install your cert-manager manifests, such as `ClusterIssuer` and `Certificate` in one go. If you put it all in one Terraform project, it will fail, because when you try to create a `ClusterIssuer` without cert-manager already being installed, Terraform will fail, saying that there is no such resource type in the cluster.

It seems to me that working as a Kubernetes cluster administrator is an almost unending question of "Which came first - the chicken, or the egg?". 

In my time of researching for how people have tried to solve these issues, I have not yet found one universal, one-size-fits-all solution that I can also recommend in a production capacity. 

## A Hybrid Approach using Terragrunt and Helm

When it comes to the Terraform problem of having to split your project into multiple Terraform projects, I have found that the best solution is to indeed split it up into multiple projects - however, instead of keeping purely Terraform, I prefer to use [Terragrunt](https://terragrunt.gruntwork.io/). It is a very powerful wrapper tool for Terraform/OpenTofu that adds the ability to deploy multiple Terraform projects that depend one another using a Directed Acrylic Graph(DAG)[^2], which ensures that Terraform modules are installed in the correct order. One small downside of this type of deployment is that for subsequent Terraform modules after the initial one, you cannot get a proper Terraform plan output, for which using mock input values becomes necessary.

What about when it comes to managing your CRDs for your Helm charts? While there are numerous methods available, there seems to be no single one solution that works for all charts. Karpenter, for example, provides a separate chart for managing the CRDs, which you can then upgrade before upgrading your main Karpenter chart. Some charts recommend that you manually upgrade the new CRDs before upgrading your chart, if there are breaking changes. Some charts, such as [Kyverno](https://kyverno.io/), use Helm hooks for post-installation that upgrade your CRDs for you. GitHub ARC, on the other hand, recommends that you[^3]:

- Uninstall all existing scale sets
- Uninstall ARC
- Delete all CRDs, if the CRDs need to be upgraded (this is often the case)
- Reinstall ARC
- Reinstall your scale sets

This process is error-prone and disruptive, especially in production environments with numerous scale sets. As you can see, the situation can become an absolute mess to manage long-term, especially when you are installing dozens of these charts into your cluster. Every cluster upgrade can become a nightmare of going through all the release notes of your charts, noting breaking changes, preparing, testing on a staging cluster, and crossing your fingers when the time comes to upgrade your production cluster.

When it comes to the Terraform solution, I have yet again seen that there is no one agreed upon solution online. Some publicly available, and very popular, Terraform modules simply seem to ignore this problem. They just install the Helm chart using the `helm_release` resource type in Terraform, which will work great on a first time installation. But what happens when you use the same module to upgrade your pre-existing installation? Some Terraform modules that have put more thought into this, use the `helm_template` data block for Terraform to render out the Kubernetes manifests, which would also update the CRDs. But what about Helm charts that also use pre/post-installation hooks and similar, which would then not be a part of the `helm template`? The same problem exists for the very popular [ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/), which can also install Helm charts, but installs them using `helm template` and not installing them using native Helm. 

While I still think that there is still no universal solution to upgrading Helm charts, I think the closest to it would be to:

- Use `helm_template` to render out the chart, but only the CRDs
- Apply those CRDs to the cluster using Terraform
- Use `helm_release` to install the chart while skipping CRD installation

### An example

```hcl
# Render and install CRDs
data "helm_template" "chart_crds" {
  name       = "my-chart"
  repository = "https://charts.example.com"
  chart      = "my-chart"
  version    = "1.2.3"
  include_crds = true
  skip_templates = true
  values = [
    file("${path.module}/values.yaml")
  ]
}

locals {
  crds = {
    for k, v in data.helm_template.chart_crds.manifests :
    k => v
    if strcontains(v, "kind: CustomResourceDefinition")
  }
}

resource "kubernetes_manifest" "crds" {
  for_each = local.crds
  manifest = yamldecode(each.value)
}

resource "kubectl_manifest" "chart_crds" {
  for_each  = { for crd in data.helm_template.chart_crds.crds : crd.metadata.name => crd }
  yaml_body = each.value
}

# Install the Helm chart
resource "helm_release" "my_chart" {
  name       = "my-chart"
  repository = "https://charts.example.com"
  chart      = "my-chart"
  version    = "1.2.3"
  namespace  = "default"
  skip_crds  = true

  values = [
    file("${path.module}/values.yaml")
  ]
  
  depends_on = [kubectl_manifest.chart_crds]
}

```

## Nelm as an Alternative to Helm

In addition, there is [nelm](https://github.com/werf/nelm), from the makers of [werf](https://werf.io/), which promises to be a faster moving and improved Helm alternative, which also provides improved CRD management, among other improvements.[^4] However, I'm still a little wary of bringing it into production as our workflows are so heavily Terraform dependent, and there is no official provider to use it within Terraform. Nelm is a promising alternative, but as of early 2026, it does not seem to be widely adopted in production environments. 


[^1]: [Helm - Custom Resource Definitions](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)
[^2]: [Terragrunt - Run Queue](https://terragrunt.gruntwork.io/docs/features/run-queue/)
[^3]: [GitHub Actions - Upgrading ARC](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/deploy-runner-scale-sets#upgrading-arc)
[^4]: [Nelm - Improved CRD management](https://github.com/werf/nelm?tab=readme-ov-file#improved-crd-management)