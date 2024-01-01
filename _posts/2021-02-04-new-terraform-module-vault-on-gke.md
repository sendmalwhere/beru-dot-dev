---
title: 'New Terraform Module: Vault on GKE'
date: 2021-02-04 00:00:00
---

#Good News, Everyone: I wrote a new Terraform module
One of the major holes in the GCP/Hashicorp ecosystem is the fact that there isn’t a batteries-included module to spin up Vault on GKE. So I decided to write one. If you’re a seasoned Terraformer, this will be pretty self-explanatory, but for the rest of us. I thought I’d walk through what this module does and why you should consider using it.

Here’s the module on the registry and the source code, if you want to follow along:

Terraform Provider Registry: [https://registry.terraform.io/modules/gatsbysghost/gke-helm-vault/google/latest](https://registry.terraform.io/modules/gatsbysghost/gke-helm-vault/google/latest)

Source Code: [https://github.com/scooberu/terraform-google-gke-helm-vault](https://github.com/scooberu/terraform-google-gke-helm-vault)

## Module Structure
This module is comprised of a set of related internal submodules. It produces no outputs (though this can be easily changed) and takes the following inputs:

### Inputs
#### GCP Auth & Zone/Region Specifiers:
`credentials_file`: Path to your GCP Service Account Creds JSON
`project_id`: Your Project ID (not Project Name)
`region`: The GCP Region to build the infra in
`cluster_zone`: The GCP Zone to build the GKE cluster in

#### Vault/Kubernetes Configuration:
`num_vault_pods`: The number of pods to create (Note: this is also the number of nodes that will be created in GKE)
`cluster_name`: The name to assign to the GKE cluster
`vault_version`: The version of Vault to install with Helm
`vault_hostname`: The FQDN of the Vault cluster

#### TLS Gubbins:
`cert_secret_name`: The name of the k8s secret to store private certs in
`cert_organization_name`: Org Name for a Private Cert
`cert_common_name`: CN for a Private Cert
`cert_country`: Country for a Private Cert
`public_cert_email_address`: Email address to associate with the ACME public TLS cert

### Submodules
Each of the internal submodules referenced by the parent module has a distinct function.

#### vault
This is the primary submodule, and is directly responsible for installing the Helm chart containing Vault. Because of the nature of Helm, a lot of the configuration for Vault is stored as text in this module. Some highlights:
The storage “raft” block defines a separate raft storage backend for each node.
An NGINX ingress is created for the whole cluster as defined by this config. The cluster will be accessible at https://[your domain name here]:8200.
The kubernetes.tf file defines a namespace in the GKE cluster for vault stuff (by default, it’s just called “vault”). That same file also defines some kubernetes secrets containing private TLS cert/ca/key files (from the tls-private submodule), and the Helm config mounts these so that Vault can use them to secure the raft backend. 

#### google-gke-cluster
This submodule create the GKE cluster itself, with as many nodes as are specified in the input variables for the number of Vault pods to run. It assigns the GCP service account generated in the google-kms submodule as a Kubernetes service account for each node. 

#### google-static-ip
This submodule simply reserves a static, external IP address for Vault to use. The output from this submodule (i.e., the address attribute) is used in google-cloud-dns and vault, to associate the A record with the listener IP address and to provision the listener IP address to NGINX (respectively).

#### google-cloud-dns
This submodule creates a DNS zone and an A record based on the provided vault_hostname input variable, and stores the output of google-static-ip in the rrdatas for that A record.

#### acme-cert
This submodule is responsible for generating a TLS cert for the publicly facing TLS listener. N.B., a terraform apply will be necessary at regular intervals in order to renew the cert, because ACME is just an interface for LetsEncrypt, which has famously short renewal intervals. In order to avoid certificate issues, run an apply every 30 days, at a minimum.

#### google-kms
This submodule creates a keyring and an encryption/decryption key. This key will be used to [automatically unseal Vault](https://learn.hashicorp.com/tutorials/vault/autounseal-gcp-kms?in=vault%2Fauto-unseal). In order to allow Vault to be automatically unsealed, this submodule also creates a new service account that Vault will use to access the keyring (along with the necessary IAM bindings to give the service account permission to do that).

#### tls-private
This submodule creates a private CA and generates a certificate, key, and ca file. It stores these files locally, and then provisions them to all nodes in the Vault cluster. This is cluster-internal transport security that allows the members of the Raft storage cluster to communicate with each other relatively securely.

## Cool, so my cluster’s up! How do I get started?
Follow [this guide](https://www.vaultproject.io/docs/platform/k8s/helm/examples/ha-with-raft) from Hashicorp! It’s that easy!
