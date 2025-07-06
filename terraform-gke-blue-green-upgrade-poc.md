# GKE Blue-Green Node Pool Upgrade with Terraform: A Complete Implementation Guide

This document provides a comprehensive Terraform implementation for performing blue-green and surge upgrades on a Google Kubernetes Engine (GKE) private cluster. This guide demonstrates how to manage node pool upgrades with zero downtime using Infrastructure as Code.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Infrastructure Setup](#infrastructure-setup)
4. [Blue-Green Upgrade Configuration](#blue-green-upgrade-configuration)
5. [Surge Upgrade Configuration](#surge-upgrade-configuration)
6. [Test Application Deployment](#test-application-deployment)
7. [Bastion Host for Private Cluster Access](#bastion-host-for-private-cluster-access)
8. [Usage Guide](#usage-guide)
9. [Validation and Monitoring](#validation-and-monitoring)

## Overview

This implementation creates:
- **Platform:** Google Kubernetes Engine (GKE)
- **Cluster Type:** Private Cluster
- **Location:** `us-west1` (regional)
- **Initial Node Count:** 3 nodes
- **Upgrade Strategies:** Both Blue-Green and Surge upgrade configurations
- **Test Workload:** Nginx deployment with PodDisruptionBudget

The Terraform configuration provides a complete infrastructure-as-code approach to manage GKE cluster upgrades, replacing manual `gcloud` commands with declarative configuration.

## Prerequisites

1. **Terraform:** Version 1.3+ installed
2. **Google Cloud SDK:** `gcloud` CLI installed and authenticated
3. **kubectl:** Kubernetes CLI installed
4. **Google Cloud Project:** With required APIs enabled:
   - Kubernetes Engine API
   - Compute Engine API
   - Cloud Resource Manager API

## Infrastructure Setup

### Provider Configuration

```hcl
# terraform/providers.tf
terraform {
  required_version = ">= 1.3"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.10"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "kubernetes" {
  host                   = "https://${module.gke.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke.ca_certificate)
}

data "google_client_config" "default" {}
```

### Variables

```hcl
# terraform/variables.tf
variable "project_id" {
  description = "The project ID to host the cluster in"
  type        = string
}

variable "region" {
  description = "The region to host the cluster in"
  type        = string
  default     = "us-west1"
}

variable "cluster_name" {
  description = "The name of the cluster"
  type        = string
  default     = "gke-upgrade-poc"
}

variable "network_name" {
  description = "The name of the VPC network"
  type        = string
  default     = "gke-upgrade-vpc"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "gke-upgrade-subnet"
}

variable "master_ipv4_cidr_block" {
  description = "The IP range in CIDR notation for the hosted master network"
  type        = string
  default     = "172.16.0.0/28"
}

variable "pods_range_name" {
  description = "The name of the secondary subnet ip range to use for pods"
  type        = string
  default     = "ip-range-pods"
}

variable "svc_range_name" {
  description = "The name of the secondary subnet range to use for services"
  type        = string
  default     = "ip-range-services"
}

variable "ip_range_pods_cidr" {
  description = "The secondary ip range to use for pods"
  type        = string
  default     = "192.168.0.0/18"
}

variable "ip_range_services_cidr" {
  description = "The secondary ip range to use for services"
  type        = string
  default     = "192.168.64.0/18"
}

variable "master_authorized_networks" {
  description = "List of master authorized networks"
  type = list(object({
    cidr_block   = string
    display_name = string
  }))
  default = [
    {
      cidr_block   = "10.0.0.0/8"
      display_name = "Internal"
    }
  ]
}

variable "kubernetes_version" {
  description = "The Kubernetes version for the cluster"
  type        = string
  default     = "1.33.1-gke.1584000"
}

variable "target_kubernetes_version" {
  description = "The target Kubernetes version for upgrades"
  type        = string
  default     = "1.33.1-gke.1744000"
}
```

### VPC and Subnet Configuration

```hcl
# terraform/network.tf

# VPC Network
resource "google_compute_network" "vpc" {
  name                    = var.network_name
  auto_create_subnetworks = false
  project                 = var.project_id
}

# Subnet with secondary ranges for pods and services
resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  ip_cidr_range = "10.0.0.0/17"
  region        = var.region
  network       = google_compute_network.vpc.self_link
  project       = var.project_id

  secondary_ip_range {
    range_name    = var.pods_range_name
    ip_cidr_range = var.ip_range_pods_cidr
  }

  secondary_ip_range {
    range_name    = var.svc_range_name
    ip_cidr_range = var.ip_range_services_cidr
  }

  # Enable private Google access for private nodes
  private_ip_google_access = true
}

# Cloud Router for NAT Gateway
resource "google_compute_router" "router" {
  name    = "${var.network_name}-router"
  region  = var.region
  network = google_compute_network.vpc.self_link
  project = var.project_id
}

# NAT Gateway for private nodes to access internet
resource "google_compute_router_nat" "nat" {
  name                               = "${var.network_name}-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
  project                            = var.project_id

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Firewall rule to allow internal communication
resource "google_compute_firewall" "allow_internal" {
  name    = "${var.network_name}-allow-internal"
  network = google_compute_network.vpc.self_link
  project = var.project_id

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "icmp"
  }

  source_ranges = [
    "10.0.0.0/17",          # Subnet range
    var.ip_range_pods_cidr, # Pods range
    var.ip_range_services_cidr, # Services range
    var.master_ipv4_cidr_block  # Master range
  ]
}
```

### IAM Service Accounts and Roles

```hcl
# terraform/iam.tf

# Service account for GKE nodes
resource "google_service_account" "gke_nodes" {
  account_id   = "${var.cluster_name}-nodes"
  display_name = "GKE Node Pool Service Account"
  project      = var.project_id
}

# IAM roles for the node service account
resource "google_project_iam_member" "gke_nodes_worker" {
  project = var.project_id
  role    = "roles/container.nodeServiceAccount"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_registry" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_artifact" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_monitoring_writer" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_logging_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}
```

### Main GKE Cluster Configuration

```hcl
# terraform/main.tf

# GKE Private Cluster
module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version = "~> 37.0"

  project_id   = var.project_id
  name         = var.cluster_name
  region       = var.region
  zones        = ["${var.region}-a", "${var.region}-b", "${var.region}-c"]

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  ip_range_pods     = var.pods_range_name
  ip_range_services = var.svc_range_name

  # Private cluster configuration
  enable_private_endpoint = false  # Allow public access to control plane
  enable_private_nodes    = true   # Nodes have private IPs only
  master_ipv4_cidr_block  = var.master_ipv4_cidr_block

  # Master authorized networks
  master_authorized_networks = var.master_authorized_networks

  # Kubernetes version
  kubernetes_version = var.kubernetes_version

  # Remove default node pool (we'll create managed node pools separately)
  remove_default_node_pool = true
  initial_node_count       = 1

  # Cluster features
  horizontal_pod_autoscaling = true
  network_policy             = true
  enable_shielded_nodes      = true
  enable_binary_authorization = false

  # Monitoring and logging
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"

  # Release channel for automatic updates
  release_channel = "REGULAR"

  # Enable workload identity
  identity_namespace = "${var.project_id}.svc.id.goog"

  # Cluster resource labels
  cluster_resource_labels = {
    purpose     = "upgrade-poc"
    environment = "test"
  }
}

# Output cluster information
output "cluster_name" {
  description = "Cluster name"
  value       = module.gke.name
}

output "cluster_location" {
  description = "Cluster location (region/zone)"
  value       = module.gke.location
}

output "cluster_endpoint" {
  description = "Cluster endpoint"
  value       = module.gke.endpoint
  sensitive   = true
}

output "cluster_ca_certificate" {
  description = "Cluster ca certificate (base64 encoded)"
  value       = module.gke.ca_certificate
  sensitive   = true
}
```

## Blue-Green Upgrade Configuration

```hcl
# terraform/blue_green_node_pool.tf

# Blue-Green Node Pool Configuration
resource "google_container_node_pool" "blue_green_pool" {
  name       = "${var.cluster_name}-blue-green-pool"
  location   = var.region
  cluster    = module.gke.name
  node_count = 1  # 1 node per zone, 3 total in regional cluster

  # Blue-Green upgrade strategy
  upgrade_settings {
    strategy = "BLUE_GREEN"
    
    blue_green_settings {
      # Soak duration before deleting blue nodes
      standard_rollout_policy {
        batch_percentage = 100  # Replace all nodes at once
        batch_soak_duration = "300s"  # 5 minutes soak time
      }
    }

    max_surge       = 3  # Allow up to 3 additional nodes during upgrade
    max_unavailable = 0  # Don't allow any nodes to be unavailable
  }

  # Auto-upgrade and auto-repair
  management {
    auto_repair  = true
    auto_upgrade = false  # Manual control for testing
  }

  # Node configuration
  node_config {
    machine_type = "e2-standard-2"
    disk_size_gb = 30
    disk_type    = "pd-ssd"
    
    # Use custom service account
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    # Security configurations
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    # Workload Identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Node labels
    labels = {
      pool-type = "blue-green"
      purpose   = "upgrade-poc"
    }

    # Node tags for firewall rules
    tags = ["gke-node", "blue-green-pool"]
  }

  # Lifecycle management
  lifecycle {
    ignore_changes = [
      node_config[0].labels,
      node_config[0].taint,
    ]
  }

  depends_on = [
    module.gke,
    google_service_account.gke_nodes
  ]
}
```

## Surge Upgrade Configuration

```hcl
# terraform/surge_node_pool.tf

# Surge Node Pool Configuration
resource "google_container_node_pool" "surge_pool" {
  name       = "${var.cluster_name}-surge-pool"
  location   = var.region
  cluster    = module.gke.name
  node_count = 1  # 1 node per zone, 3 total in regional cluster

  # Surge upgrade strategy (default)
  upgrade_settings {
    strategy = "SURGE"
    
    max_surge       = 1  # Add 1 node at a time
    max_unavailable = 0  # Don't allow any nodes to be unavailable
  }

  # Auto-upgrade and auto-repair
  management {
    auto_repair  = true
    auto_upgrade = false  # Manual control for testing
  }

  # Node configuration
  node_config {
    machine_type = "e2-standard-2"
    disk_size_gb = 30
    disk_type    = "pd-ssd"
    
    # Use custom service account
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    # Security configurations
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    # Workload Identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Node labels
    labels = {
      pool-type = "surge"
      purpose   = "upgrade-poc"
    }

    # Node tags for firewall rules
    tags = ["gke-node", "surge-pool"]
  }

  # Lifecycle management
  lifecycle {
    ignore_changes = [
      node_config[0].labels,
      node_config[0].taint,
    ]
  }

  depends_on = [
    module.gke,
    google_service_account.gke_nodes
  ]
}
```

## Test Application Deployment

```hcl
# terraform/test_application.tf

# Nginx test deployment
resource "kubernetes_deployment" "nginx_test" {
  metadata {
    name = "nginx-test"
    labels = {
      app = "nginx-test"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "nginx-test"
      }
    }

    template {
      metadata {
        labels = {
          app = "nginx-test"
        }
      }

      spec {
        container {
          name  = "nginx"
          image = "nginx:latest"

          port {
            container_port = 80
          }

          liveness_probe {
            http_get {
              path = "/"
              port = 80
            }
            initial_delay_seconds = 30
            period_seconds        = 10
          }

          readiness_probe {
            http_get {
              path = "/"
              port = 80
            }
            initial_delay_seconds = 5
            period_seconds        = 5
          }

          resources {
            requests = {
              cpu    = "100m"
              memory = "128Mi"
            }
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
          }
        }
      }
    }
  }

  depends_on = [module.gke]
}

# Service for nginx test deployment
resource "kubernetes_service" "nginx_test_service" {
  metadata {
    name = "nginx-test-service"
  }

  spec {
    selector = {
      app = "nginx-test"
    }

    port {
      protocol    = "TCP"
      port        = 80
      target_port = 80
    }

    type = "LoadBalancer"
  }

  depends_on = [kubernetes_deployment.nginx_test]
}

# PodDisruptionBudget for nginx test
resource "kubernetes_pod_disruption_budget_v1" "nginx_test_pdb" {
  metadata {
    name = "nginx-test-pdb"
  }

  spec {
    min_available = 2

    selector {
      match_labels = {
        app = "nginx-test"
      }
    }
  }

  depends_on = [kubernetes_deployment.nginx_test]
}

# Output service information
output "nginx_service_ip" {
  description = "Nginx test service external IP"
  value       = kubernetes_service.nginx_test_service.status.0.load_balancer.0.ingress.0.ip
}

output "nginx_service_cluster_ip" {
  description = "Nginx test service cluster IP"
  value       = kubernetes_service.nginx_test_service.spec.0.cluster_ip
}
```

## Bastion Host for Private Cluster Access

For secure access to the private GKE cluster, especially when the control plane endpoint is private or when accessing from outside the VPC, a bastion host with IAP tunnel provides a secure and straightforward solution.

### Bastion Host Configuration

```hcl
# terraform/bastion.tf

# Variables for bastion host
variable "bastion_machine_type" {
  description = "Machine type for the bastion host"
  type        = string
  default     = "e2-micro"
}

variable "bastion_image" {
  description = "Image for the bastion host"
  type        = string
  default     = "debian-cloud/debian-12"
}

# Service account for bastion host
resource "google_service_account" "bastion" {
  account_id   = "${var.cluster_name}-bastion"
  display_name = "Bastion Host Service Account"
  project      = var.project_id
}

# IAM roles for bastion host
resource "google_project_iam_member" "bastion_compute_viewer" {
  project = var.project_id
  role    = "roles/compute.viewer"
  member  = "serviceAccount:${google_service_account.bastion.email}"
}

resource "google_project_iam_member" "bastion_container_developer" {
  project = var.project_id
  role    = "roles/container.developer"
  member  = "serviceAccount:${google_service_account.bastion.email}"
}

resource "google_project_iam_member" "bastion_iap_user" {
  project = var.project_id
  role    = "roles/iap.tunnelResourceAccessor"
  member  = "serviceAccount:${google_service_account.bastion.email}"
}

# Startup script for bastion host
locals {
  startup_script = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y curl wget gnupg2 software-properties-common apt-transport-https ca-certificates
    
    # Install Google Cloud SDK
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
    apt-get update
    apt-get install -y google-cloud-cli google-cloud-cli-gke-gcloud-auth-plugin
    
    # Install kubectl
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubectl
    
    # Install helm (optional but useful)
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    apt-get update
    apt-get install -y helm
    
    # Create kubectl config directory for all users
    mkdir -p /home/debian/.kube
    chown debian:debian /home/debian/.kube
    
    # Install additional utilities
    apt-get install -y vim nano htop git curl jq
  EOF
}

# Debian bastion host
resource "google_compute_instance" "bastion" {
  name         = "${var.cluster_name}-bastion"
  machine_type = var.bastion_machine_type
  zone         = "${var.region}-a"
  project      = var.project_id

  boot_disk {
    initialize_params {
      image = var.bastion_image
      size  = 20
      type  = "pd-standard"
    }
  }

  network_interface {
    network    = google_compute_network.vpc.self_link
    subnetwork = google_compute_subnetwork.subnet.self_link
    # No external IP - access via IAP only
  }

  # Enable IP forwarding for potential proxy usage
  can_ip_forward = false

  # Service account
  service_account {
    email  = google_service_account.bastion.email
    scopes = ["cloud-platform"]
  }

  # Startup script
  metadata_startup_script = local.startup_script

  # Metadata
  metadata = {
    enable-oslogin = "TRUE"
    # Optionally add SSH keys for specific users
    # ssh-keys = "username:ssh-rsa AAAAB3NzaC1yc2E... user@example.com"
  }

  # Labels
  labels = {
    purpose     = "bastion"
    environment = "gke-upgrade-poc"
  }

  # Tags for firewall rules
  tags = ["bastion-host", "iap-ssh"]

  depends_on = [
    google_compute_network.vpc,
    google_compute_subnetwork.subnet,
    google_service_account.bastion
  ]
}

# Firewall rule to allow IAP SSH access
resource "google_compute_firewall" "allow_iap_ssh" {
  name    = "${var.network_name}-allow-iap-ssh"
  network = google_compute_network.vpc.self_link
  project = var.project_id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  # IAP's source IP ranges
  source_ranges = ["35.235.240.0/20"]
  target_tags   = ["iap-ssh"]

  description = "Allow IAP SSH access to bastion host"
}

# Update master authorized networks to include bastion
variable "master_authorized_networks_with_bastion" {
  description = "List of master authorized networks including bastion"
  type = list(object({
    cidr_block   = string
    display_name = string
  }))
  default = [
    {
      cidr_block   = "10.0.0.0/8"
      display_name = "Internal"
    },
    {
      cidr_block   = "10.0.0.0/17"
      display_name = "GKE Subnet"
    }
  ]
}

# Output bastion information
output "bastion_name" {
  description = "Bastion host instance name"
  value       = google_compute_instance.bastion.name
}

output "bastion_zone" {
  description = "Bastion host zone"
  value       = google_compute_instance.bastion.zone
}

output "bastion_internal_ip" {
  description = "Bastion host internal IP"
  value       = google_compute_instance.bastion.network_interface.0.network_ip
}
```

### Private Cluster Configuration Options

For enhanced security, you can configure the GKE cluster with a private endpoint. Update the main cluster configuration:

```hcl
# terraform/main.tf - Private Endpoint Option

module "gke" {
  # ... existing configuration ...

  # Private cluster configuration - Enhanced Security Option
  enable_private_endpoint = true   # Private control plane endpoint
  enable_private_nodes    = true   # Nodes have private IPs only
  master_ipv4_cidr_block  = var.master_ipv4_cidr_block

  # Master authorized networks (include bastion subnet)
  master_authorized_networks = var.master_authorized_networks_with_bastion
  
  # ... rest of configuration ...
}
```

**Note**: When `enable_private_endpoint = true`, the Kubernetes API server is only accessible from within the VPC or through authorized networks. The bastion host becomes essential for cluster access.

## Usage Guide

### 1. Initial Setup

```bash
# Clone and setup
git clone <your-repo>
cd terraform/

# Initialize Terraform
terraform init

# Create terraform.tfvars
cat > terraform.tfvars <<EOF
project_id = "your-project-id"
region     = "us-west1"
cluster_name = "gke-upgrade-poc"
EOF

# Plan and apply
terraform plan
terraform apply
```

### 2. Configure kubectl

#### Option A: Direct Access (Public Endpoint)

For clusters with public endpoints (`enable_private_endpoint = false`):

```bash
# Get cluster credentials
gcloud container clusters get-credentials gke-upgrade-poc \
  --region us-west1 \
  --project your-project-id

# Verify cluster access
kubectl get nodes
kubectl get pods -l app=nginx-test
```

#### Option B: Via IAP Tunnel and Bastion Host (Recommended for Private Clusters)

For enhanced security or private endpoints (`enable_private_endpoint = true`):

**Step 1: Enable required APIs and configure IAP access**

```bash
# Enable required APIs
gcloud services enable iap.googleapis.com
gcloud services enable compute.googleapis.com

# Add your user account to IAP SSH access
gcloud projects add-iam-policy-binding your-project-id \
  --member="user:your-email@domain.com" \
  --role="roles/iap.tunnelResourceAccessor"

gcloud projects add-iam-policy-binding your-project-id \
  --member="user:your-email@domain.com" \
  --role="roles/compute.instanceAdmin.v1"
```

**Step 2: Connect to bastion host via IAP tunnel**

```bash
# Get bastion information
BASTION_NAME=$(terraform output -raw bastion_name)
BASTION_ZONE=$(terraform output -raw bastion_zone)

# Connect to bastion via IAP tunnel
gcloud compute ssh debian@${BASTION_NAME} \
  --zone=${BASTION_ZONE} \
  --tunnel-through-iap \
  --project=your-project-id
```

**Step 3: Configure kubectl on bastion host**

Once connected to the bastion host:

```bash
# Authenticate with Google Cloud
gcloud auth login

# Set project
gcloud config set project your-project-id

# Get cluster credentials
gcloud container clusters get-credentials gke-upgrade-poc \
  --region us-west1 \
  --project your-project-id

# Verify cluster access
kubectl get nodes
kubectl get pods -l app=nginx-test
```

**Step 4: Set up kubectl proxy for local access (Optional)**

To use kubectl from your local machine through the bastion:

```bash
# In one terminal: Create IAP tunnel to bastion
gcloud compute start-iap-tunnel ${BASTION_NAME} 22 \
  --local-host-port=localhost:2222 \
  --zone=${BASTION_ZONE} \
  --project=your-project-id

# In another terminal: Create SSH tunnel for kubectl proxy
ssh -L 8001:localhost:8001 \
  -o ProxyCommand="gcloud compute ssh debian@${BASTION_NAME} --zone=${BASTION_ZONE} --tunnel-through-iap --command='nc %h %p'" \
  debian@localhost

# On bastion host: Start kubectl proxy
kubectl proxy --accept-hosts='.*' --address='0.0.0.0'

# On local machine: Configure kubectl to use proxy
kubectl --server=http://localhost:8001 get nodes
```

### 3. Blue-Green Upgrade Process

```hcl
# Update the blue-green node pool version
# terraform/terraform.tfvars
target_kubernetes_version = "1.33.1-gke.1744000"
```

```bash
# Update the node pool version in Terraform configuration
# Edit blue_green_node_pool.tf and add:
resource "google_container_node_pool" "blue_green_pool" {
  # ... existing configuration ...
  
  version = var.target_kubernetes_version  # Add this line
  
  # ... rest of configuration ...
}

# Apply the upgrade
terraform plan -target=google_container_node_pool.blue_green_pool
terraform apply -target=google_container_node_pool.blue_green_pool
```

### 4. Surge Upgrade Process

```hcl
# Update the surge node pool version
resource "google_container_node_pool" "surge_pool" {
  # ... existing configuration ...
  
  version = var.target_kubernetes_version  # Add this line
  
  # ... rest of configuration ...
}
```

```bash
# Apply the surge upgrade
terraform plan -target=google_container_node_pool.surge_pool
terraform apply -target=google_container_node_pool.surge_pool
```

### 5. Monitoring Commands

```bash
# Monitor nodes during upgrade
watch -n 10 'kubectl get nodes -o wide'

# Monitor pods during upgrade
watch -n 5 'kubectl get pods -l app=nginx-test -o wide'

# Check upgrade operations
gcloud container operations list \
  --filter="operationType:UPGRADE_NODES" \
  --format="table(name,status,statusMessage,startTime)"

# Test application availability
CLUSTER_IP=$(kubectl get svc nginx-test-service -o jsonpath='{.spec.clusterIP}')
POD_NAME=$(kubectl get pods -l app=nginx-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- curl http://$CLUSTER_IP:80
```

## Validation and Monitoring

### Infrastructure Validation

```bash
# Verify cluster configuration
terraform output cluster_name
terraform output cluster_location

# Check node pools
kubectl get nodes --show-labels
gcloud container node-pools list --cluster=gke-upgrade-poc --region=us-west1

# Verify private cluster setup
gcloud container clusters describe gke-upgrade-poc \
  --region=us-west1 \
  --format="value(privateClusterConfig.enablePrivateNodes)"
```

### Application Health Checks

```bash
# Check deployment status
kubectl get deployment nginx-test
kubectl describe deployment nginx-test

# Check service status
kubectl get service nginx-test-service
kubectl describe service nginx-test-service

# Check PodDisruptionBudget
kubectl get pdb nginx-test-pdb
kubectl describe pdb nginx-test-pdb

# Continuous availability test
while true; do
  kubectl exec $(kubectl get pods -l app=nginx-test -o jsonpath='{.items[0].metadata.name}') \
    -- curl -s http://$(kubectl get svc nginx-test-service -o jsonpath='{.spec.clusterIP}'):80 \
    | grep -q "Welcome to nginx" && echo "$(date): OK" || echo "$(date): FAILED"
  sleep 5
done
```

### Cleanup

```bash
# Destroy infrastructure
terraform destroy

# Or destroy specific components
terraform destroy -target=kubernetes_deployment.nginx_test
terraform destroy -target=google_container_node_pool.blue_green_pool
terraform destroy -target=google_container_node_pool.surge_pool
```

## Best Practices

1. **Version Management**: Always specify exact Kubernetes versions
2. **Resource Limits**: Set appropriate resource requests and limits
3. **Health Checks**: Configure liveness and readiness probes
4. **Monitoring**: Use proper monitoring and alerting
5. **Backup**: Regular backups of persistent data
6. **Testing**: Test upgrades in non-production environments first
7. **Documentation**: Keep upgrade procedures documented and updated

## Troubleshooting

### Common Issues

1. **Node Pool Upgrade Stuck**: Check master version compatibility
2. **Pod Eviction Failures**: Verify PodDisruptionBudget settings
3. **Network Connectivity**: Ensure NAT Gateway and firewall rules
4. **Permission Errors**: Verify IAM roles and service accounts
5. **Resource Quotas**: Check project quotas and limits
6. **IAP Tunnel Connection Failed**: Check IAP access permissions and firewall rules
7. **Bastion Host Access Denied**: Verify IAM roles and OS Login configuration
8. **Private Cluster Unreachable**: Check master authorized networks and bastion IP
9. **kubectl Commands Timeout**: Ensure bastion has access to cluster endpoint

### Debugging Commands

#### General Cluster Debugging

```bash
# Check cluster events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check node pool operations
gcloud container operations list --region=us-west1

# Check cluster details
gcloud container clusters describe gke-upgrade-poc --region=us-west1

# Check logs
kubectl logs -l app=nginx-test
gcloud logging read "resource.type=gke_cluster"
```

#### IAP and Bastion Debugging

```bash
# Check bastion host status
gcloud compute instances describe ${BASTION_NAME} --zone=${BASTION_ZONE}

# Check IAP firewall rules
gcloud compute firewall-rules list --filter="name~iap"

# Test IAP connectivity
gcloud compute start-iap-tunnel ${BASTION_NAME} 22 \
  --local-host-port=localhost:2222 \
  --zone=${BASTION_ZONE} \
  --project=your-project-id

# Check bastion host logs
gcloud compute instances get-serial-port-output ${BASTION_NAME} --zone=${BASTION_ZONE}

# Verify IAM permissions
gcloud projects get-iam-policy your-project-id \
  --filter="bindings.members:user:your-email@domain.com"

# Test bastion connectivity to GKE cluster
gcloud compute ssh debian@${BASTION_NAME} \
  --zone=${BASTION_ZONE} \
  --tunnel-through-iap \
  --command="kubectl cluster-info"

# Check master authorized networks
gcloud container clusters describe gke-upgrade-poc \
  --region=us-west1 \
  --format="value(masterAuthorizedNetworksConfig.cidrBlocks[].cidrBlock)"

# Verify bastion can reach cluster endpoint
CLUSTER_ENDPOINT=$(gcloud container clusters describe gke-upgrade-poc \
  --region=us-west1 \
  --format="value(endpoint)")

gcloud compute ssh debian@${BASTION_NAME} \
  --zone=${BASTION_ZONE} \
  --tunnel-through-iap \
  --command="curl -k https://${CLUSTER_ENDPOINT}/api/v1 -I"
```

#### Quick IAP Access Test

```bash
# Simple connectivity test
gcloud compute ssh debian@${BASTION_NAME} \
  --zone=${BASTION_ZONE} \
  --tunnel-through-iap \
  --command="echo 'Bastion is reachable via IAP'"

# Test kubectl from bastion
gcloud compute ssh debian@${BASTION_NAME} \
  --zone=${BASTION_ZONE} \
  --tunnel-through-iap \
  --command="kubectl get nodes --kubeconfig=/home/debian/.kube/config"
```

This comprehensive Terraform implementation provides a complete infrastructure-as-code solution for managing GKE cluster upgrades with both blue-green and surge strategies, replacing manual processes with automated, repeatable deployments.
