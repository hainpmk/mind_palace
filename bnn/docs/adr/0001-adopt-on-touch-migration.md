---
status: accepted
---

# Adopt existing infra into Terraform on-touch, not via migration project

Much of the current infra (VMs, EKS nodepools, MongoDB VM, GCP components) was created manually and is undocumented. We considered running a dedicated migration project to import everything into Terraform up front, but decided against it: it's high-effort, high-risk (importing live infra without full understanding of its config), and blocks new work. Instead, new components are Terraform-managed from creation, and existing components are imported only when someone needs to change them for a real reason. Discovery/documentation of existing components proceeds independently and does not by itself trigger import.
