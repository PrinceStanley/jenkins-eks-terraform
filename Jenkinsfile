// Jenkinsfile

// Define global environment variables for the pipeline
// IMPORTANT: DO NOT hardcode sensitive values here. Use Jenkins Credentials or IRSA.
def AWS_REGION = 'us-east-1' // Example region, change as needed
def TF_STATE_BUCKET = 'n8n-sb1-bucket001' // Replace with your S3 bucket name
def TF_STATE_LOCK_TABLE = 'test-eks-upgrade-lock-table' // Replace with your DynamoDB table name
def EKS_CLUSTER_NAME = 'test-eks-upgrade-cluster' // Name of your EKS cluster
def EKS_KUBERNETES_VERSION = '1.30' // Target EKS Kubernetes version
def EKS_NODE_GROUP_NAME = 'test-eks-upgrade-cluster-ng' // Name of your node group for the cluster

def EXISTING_VPC_ID = 'vpc-0e3e0e5f71c6d2dfb' // <<-- REPLACE with your existing VPC ID
def EXISTING_PRIVATE_SUBNET_IDS = 'subnet-05c75af00f233a847,subnet-0ee22c1cfa1d6fbb2,subnet-01cb3b49d3b0228e2' // <<-- REPLACE with your existing private subnet IDs (comma-separated)
def EXISTING_CLUSTER_SECURITY_GROUP_ID = 'sg-0de38d80fa2770dd5' // <<-- REPLACE with your existing cluster security group ID (if you have one for common access, else omit or let Terraform create)

def ADDON_COREDNS_VERSION = 'v1.11.4-eksbuild.2' // Example version for EKS 1.30, check AWS docs for latest
def ADDON_KUBE_PROXY_VERSION = 'v1.32.0-eksbuild.2' // Example, check AWS docs for latest
def ADDON_VPC_CNI_VERSION = 'v1.19.2-eksbuild.1' // Example, check AWS docs for latest
def ADDON_EBS_CSI_DRIVER_VERSION = 'v1.35.0-eksbuild.2' // Example, check AWS docs for latest
def ADDON_EFS_CSI_DRIVER_VERSION = 'v2.1.4-eksbuild.1' // Example, check AWS docs for latest

def NODE_GROUP_LAUNCH_TEMPLATE_ID = 'lt-0a614edd658f028f5' // <<-- REPLACE with your existing Launch Template ID
def NODE_GROUP_LAUNCH_TEMPLATE_VERSION = '2' // <<-- REPLACE with specific version, or '$Latest', '$Default'



pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes' // Must match the name of your Kubernetes cloud config in Jenkins
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
    role: terraform-eks-builder
spec:
  # This ServiceAccount must exist in the namespace where agents are deployed.
  # It should be annotated for IRSA (IAM Roles for Service Accounts) to interact with AWS.
  serviceAccountName: default # Or a dedicated service account like 'jenkins-eks-sa'
  containers:
    - name: jnlp # Jenkins JNLP Agent
      image: jenkins/jnlp-agent:latest # Or a specific version like '4.13.2-1'
      resources:
        limits:
          cpu: "1000m" # 1 CPU core
          memory: "2Gi"
        requests:
          cpu: "500m" # 0.5 CPU core
          memory: "1Gi"
      env:
        # Replace with your Jenkins service URL that agents can reach
        - name: JENKINS_URL
          value: "http://jenkins.jenkins.svc.cluster.local:8080"
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent/workspace

    - name: terraform # Sidecar for Terraform CLI
      image: hashicorp/terraform:1.7.0 # Use the specific version you need (1.30.x not yet released, 1.7.x is current)
      # Run as non-root for security
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        allowPrivilegeEscalation: false
      resources:
        limits:
          cpu: "500m"
          memory: "512Mi"
        requests:
          cpu: "250m"
          memory: "256Mi"
      command: ["/bin/sh", "-c"] # Keep the container running
      args: ["sleep infinity"] # Keep the container running
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent/workspace
        - name: kubeconfig-volume # Mount for kubectl access (if Terraform uses it)
          mountPath: /home/jenkins/.kube

    - name: awscli-kubectl # Sidecar for AWS CLI and kubectl
      image: public.ecr.aws/aws-cli/aws-cli:latest # Or a specific version like '2.17.0'
      # Run as non-root for security
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        allowPrivilegeEscalation: false
      resources:
        limits:
          cpu: "500m"
          memory: "512Mi"
        requests:
          cpu: "250m"
          memory: "256Mi"
      command: ["/bin/sh", "-c"] # Keep the container running
      args: ["sleep infinity"] # Keep the container running
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent/workspace
        - name: kubeconfig-volume # Mount for kubectl to write and read config
          mountPath: /home/jenkins/.kube

  volumes:
    - name: workspace-volume # Shared volume for workspace (cloned repo, tfplan)
      emptyDir: {}
    - name: kubeconfig-volume # Shared volume for kubectl configuration
      emptyDir: {}
            """
        }
    }

    options {
        disableConcurrentBuilds() // Prevent multiple builds from running simultaneously
        timeout(time: 2, unit: 'HOURS') // Global pipeline timeout
    }

    parameters {
        choice(name: 'ACTION', choices: ['install', 'upgrade', 'destroy'], description: 'Select action for EKS cluster')
        string(name: 'CLUSTER_VERSION_TO_UPGRADE_TO', defaultValue: '', description: 'Specify the Kubernetes version to upgrade to (e.g., 1.29, 1.30). Required for "upgrade" action.')
        booleanParam(name: 'CONFIRM_DESTROY', defaultValue: false, description: 'Check this box to confirm cluster destruction. DANGER ZONE!')
        string(name: 'COREDNS_ADDON_VERSION', defaultValue: ADDON_COREDNS_VERSION, description: 'Specific version for CoreDNS addon.')
        string(name: 'KUBE_PROXY_ADDON_VERSION', defaultValue: ADDON_KUBE_PROXY_VERSION, description: 'Specific version for Kube-Proxy addon.')
        string(name: 'VPC_CNI_ADDON_VERSION', defaultValue: ADDON_VPC_CNI_VERSION, description: 'Specific version for VPC CNI addon.')
        string(name: 'EBS_CSI_DRIVER_ADDON_VERSION', defaultValue: ADDON_EBS_CSI_DRIVER_VERSION, description: 'Specific version for EBS CSI Driver addon.')
        string(name: 'EFS_CSI_DRIVER_VERSION', defaultValue: ADDON_EFS_CSI_DRIVER_VERSION, description: 'Specific version for EFS CSI Driver addon.')
        string(name: 'NODE_GROUP_LT_ID', defaultValue: NODE_GROUP_LAUNCH_TEMPLATE_ID, description: 'ID of the existing EC2 Launch Template for the node group.')
        string(name: 'NODE_GROUP_LT_VERSION', defaultValue: NODE_GROUP_LAUNCH_TEMPLATE_VERSION, description: 'Version of the Launch Template ($Latest, $Default, or specific version number).')
    }

    environment {
        // Terraform environment variables to pass to the Terraform run
        TF_VAR_cluster_name = "${EKS_CLUSTER_NAME}"
        TF_VAR_kubernetes_version =  "${EKS_KUBERNETES_VERSION}"
        TF_VAR_node_group_name = "${EKS_NODE_GROUP_NAME}"
        TF_VAR_aws_region = "${AWS_REGION}"
        TF_VAR_existing_vpc_id = "${EXISTING_VPC_ID}"
        TF_VAR_existing_private_subnet_ids = "${EXISTING_PRIVATE_SUBNET_IDS}"
        TF_VAR_existing_public_subnet_ids = "${EXISTING_PUBLIC_SUBNET_IDS}"
        TF_VAR_existing_cluster_security_group_id = "${EXISTING_CLUSTER_SECURITY_GROUP_ID}"
        TF_VAR_addon_coredns_version = "${params.COREDNS_ADDON_VERSION}"
        TF_VAR_addon_kube_proxy_version = "${params.KUBE_PROXY_ADDON_VERSION}"
        TF_VAR_addon_vpc_cni_version = "${params.VPC_CNI_ADDON_VERSION}"
        TF_VAR_addon_ebs_csi_driver_version = "${params.EBS_CSI_DRIVER_ADDON_VERSION}"
        TF_VAR_addon_efs_csi_driver_version = "${params.EFS_CSI_DRIVER_VERSION}"
        TF_VAR_node_group_launch_template_id = "${params.NODE_GROUP_LT_ID}"
        TF_VAR_node_group_launch_template_version = "${params.NODE_GROUP_LT_VERSION}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                container('jnlp') { // Run this step in the JNLP agent container
                    script {
                        echo "Cloning Terraform repository..."
                        // Replace 'your-git-repo-url' and 'your-branch' with your actual repository
                        // Use a Jenkins credential for Git if your repository is private
                        git branch: 'main', credentialsId: 'your-git-credential-id', url: 'https://github.com/your-org/your-eks-terraform-repo.git'
                    }
                }
            }
        }

        stage('Terraform Init') {
            steps {
                container('terraform') { // Run this step in the Terraform sidecar
                    script {
                        echo "Initializing Terraform..."
                        // IRSA (IAM Roles for Service Accounts) is assumed for AWS authentication.
                        // The 'aws-cli' container runs first in 'Apply' stage to set up kubeconfig,
                        // ensuring Terraform's AWS provider can authenticate correctly if needed for initial calls.
                        // Ensure your Jenkins agent's Service Account has an IAM role attached via IRSA with permissions.
                        sh "terraform init -backend-config=\"bucket=${TF_STATE_BUCKET}\" -backend-config=\"key=eks/${EKS_CLUSTER_NAME}/terraform.tfstate\" -backend-config=\"dynamodb_table=${TF_STATE_LOCK_TABLE}\" -backend-config=\"region=${AWS_REGION}\""
                    }
                }
            }
        }

        stage('Plan') {
            steps {
                container('terraform') { // Run this step in the Terraform sidecar
                    when { expression { params.ACTION != 'destroy' } } // Don't plan if destroying, destroy command does its own plan
                    script {
                        echo "Generating Terraform plan..."
                        sh "terraform plan -out=tfplan"
                    }
                }
            }
        }

        stage('Apply / Upgrade / Destroy') {
            steps {
                script { // Use a script block for conditional logic
                    // Install EKS Cluster
                    when { expression { params.ACTION == 'install' } }
                    input message: "Proceed with EKS cluster installation for '${EKS_CLUSTER_NAME}' (v${EKS_KUBERNETES_VERSION})?", ok: 'Install'
                    container('terraform') {
                        echo "Applying Terraform to install EKS cluster..."
                        sh "terraform apply -auto-approve tfplan"
                    }
                    container('awscli-kubectl') { // Use awscli-kubectl for AWS CLI commands
                        echo "Updating Kubeconfig for new cluster..."
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}"
                    }

                    // Upgrade EKS Cluster and Addons
                    when { expression { params.ACTION == 'upgrade' } }
                    script {
                        if (params.CLUSTER_VERSION_TO_UPGRADE_TO.trim().isEmpty()) {
                            error("CLUSTER_VERSION_TO_UPGRADE_TO parameter is required for 'upgrade' action.")
                        }
                    }
                    input message: "Proceed with EKS cluster upgrade for '${EKS_CLUSTER_NAME}' to v${params.CLUSTER_VERSION_TO_UPGRADE_TO}? This will upgrade both the control plane and node group, and addons.", ok: 'Upgrade'

                    container('terraform') {
                        echo "Upgrading EKS Control Plane to Kubernetes v${params.CLUSTER_VERSION_TO_UPGRADE_TO}..."
                        // Pass the upgrade version to Terraform environment variables explicitly
                        sh "export TF_VAR_kubernetes_version=${params.CLUSTER_VERSION_TO_UPGRADE_TO} && terraform apply -auto-approve -target=module.eks"
                    }

                    container('awscli-kubectl') {
                        echo "Updating Kubeconfig for new cluster version after control plane upgrade..."
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}"
                    }

                    container('terraform') {
                        echo "Performing Node Group Rolling Update (Blue/Green for 1 node) and upgrading addons..."
                        // Pass the upgrade version for the node group and addons
                        // The 'update_config' in Terraform for the node group ensures new node comes up first.
                        sh "export TF_VAR_kubernetes_version=${params.CLUSTER_VERSION_TO_UPGRADE_TO} && terraform apply -auto-approve"
                        echo "EKS Cluster, Node Group, and Addons upgraded successfully to v${params.CLUSTER_VERSION_TO_UPGRADE_TO}"
                    }

                    // Destroy EKS Cluster
                    when { expression { params.ACTION == 'destroy' } }
                    script {
                        if (!params.CONFIRM_DESTROY) {
                            error("Destroy action requires 'CONFIRM_DESTROY' to be checked.")
                        }
                    }
                    input message: "ARE YOU ABSOLUTELY SURE YOU WANT TO DESTROY EKS CLUSTER '${EKS_CLUSTER_NAME}'? This is irreversible!", ok: 'Destroy'
                    container('terraform') {
                        echo "Destroying EKS cluster..."
                        sh "terraform destroy -auto-approve"
                    }
                }
            }
        }

        stage('Cleanup (Optional)') {
            steps {
                container('jnlp') { // Can run in any container, JNLP is fine for basic file ops
                    echo "Cleaning up workspace..."
                    sh "rm -f tfplan" // Remove the Terraform plan file
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for EKS cluster: ${EKS_CLUSTER_NAME} with action: ${params.ACTION}"
        }
        success {
            echo "EKS operation completed successfully! Check AWS Console for confirmation."
        }
        failure {
            echo "EKS operation failed. Review the logs for errors."
            // Optionally, add notifications here (e.g., Slack, Email)
            // mail to: 'devops-team@example.com', subject: "Jenkins Pipeline Failed: EKS ${params.ACTION} for ${EKS_CLUSTER_NAME}", body: "Build URL: ${env.BUILD_URL}"
        }
    }
}