# jenkins-eks-terraform
Jenkins pipeline that runs on EKS, deploys/upgrades/destroys an EKS cluster (v1.30) with specific addons using Terraform, but critically, it should use pre-existing VPC, subnets, and security groups. This is a common pattern for managing EKS within an established network infrastructure.
