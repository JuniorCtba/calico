---
title: Calico Kubernetes Hosted Install
---

Calico can be installed on a Kubernetes cluster with a single command.  

```
kubectl apply -f calico.yaml
```

We maintain several manifests.  Which one you use depends on the specific 
requirements of your Calico installation:

#### [Standard Hosted Install](hosted)

This manifest installs Calico for use with an existing etcd cluster.  This is 
the recommended hosted approach for deploying Calico in production.

#### [Kubeadm Hosted Install](kubeadm/)

This manifest installs Calico as well as a single node etcd cluster.  This is the recommended hosted approach
for getting started quickly with Calico in conjunction with tools like kubeadm.

#### [Etcdless Hosted Install](k8s-backend/)

This manifest installs Calico in a mode where it does not require its own etcd cluster.  This is an experimental
mode in which the Kubernetes API is used by Calico as its datastore. 

## How it works

Each manifest contains all the necessary resources for installing Calico on each node in your Kubernetes cluster.

It installs the following Kubernetes resources: 

- The `calico-config` ConfigMap, which contains parameters for configuring the install.
- Installs the `calico/node` container on each host using a DaemonSet.
- Installs the Calico CNI binaries and network config on each host using a DaemonSet.
- Runs the `calico/kube-policy-controller` pod as a ReplicaSet.
- The `calico-etcd-secrets` Secret, which optionally allows for providing etcd TLS assets.

## Upgrade

There are two parts to upgrading a Calico hosted installation.

##### Upgrading the Calico policy controller

The policy controller is installed as a ReplicaSet, and thus supports standard Kubernetes [rolling updates](http://kubernetes.io/docs/user-guide/rolling-updates/).

##### Upgrading the Calico DaemonSet

Upgrading the CNI plugin or calico/node image is done through a DaemonSet, and as such does not yet support rolling updates.

To upgrade the DaemonSet:

1. Apply changes to the existing DaemonSet via kubectl apply.
2. Manually perform the upgrade by deleting pods and allowing the DaemonSet controller to recreate them 
with the new images and configuration.

## Configuration options

The ConfigMap in `calico.yaml` provides a way to configure a Calico self-hosted installation.  It exposes
the following configuration parameters:

## Etcd Configuration

By default, these manifests do not configure secure access to etcd and assume an etcd proxy is running on each host.  The following configuration
options let you specify custom etcd cluster endpoints as well as TLS.  

The following table outlines the supported ConfigMap options for etcd:
 
| Option                 | Description    | Default 
|------------------------|----------------|----------
| etcd_endpoints         | A comma separated list of etcd nodes. | http://127.0.0.1:2379
| etcd_ca                | The location of the CA mounted in the pods deployed by the DaemonSet. | None
| etcd_key               | The location of the client cert mounted in the pods deployed by the DaemonSet. | None
| etcd_cert              | The location of the client key mounted in the pods deployed by the DaemonSet. | None

To use these manifests with a TLS enabled etcd cluster you must do the following:

- Populate the `calico-etcd-secrets` Secret with the contents of the following files: 
  - `etcd-ca`
  - `etcd-key`
  - `etcd-cert`
- Populate the following options in the ConfigMap which will trigger the various services to expect the provided TLS assets: 
  - `etcd_ca: /calico-secrets/etcd-ca`
  - `etcd_key: /calico-secrets/etcd-key`
  - `etcd_cert: /calico-secrets/etcd-cert`

## Other Configuration Options

The following table outlines the remaining supported ConfigMap options: 

| Option                 | Description         | Default 
|------------------------|---------------------|----------
| calico_backend         | The backend to use. | bird 
| cni_network_config     | The CNI Network config to install on each node.  Supports templating as described below. | 

### CNI Network Config Template Support

The `cni_network_config` configuration option supports the following template fields, which will 
be filled in automatically by the `calico/cni` container:

| Field                                 | Substituted with 
|---------------------------------------|----------------------------------
| `__KUBERNETES_SERVICE_HOST__`         | The Kubernetes Service ClusterIP. e.g 10.0.0.1 
| `__KUBERNETES_SERVICE_PORT__`         | The Kubernetes Service port. e.g 443
| `__SERVICEACCOUNT_TOKEN__`            | The serviceaccount token for the namespace, if one exists.
| `__ETCD_ENDPOINTS__`                  | The etcd endpoints specified in etcd_endpoints. 
| `__KUBECONFIG_FILEPATH__`             | The path to the automatically generated kubeconfig file in the same directory as the CNI network config file.
| `__ETCD_KEY_FILE__`                   | The path to the etcd key file installed to the host, empty if no key present.
| `__ETCD_CERT_FILE__`                  | The path to the etcd cert file installed to the host, empty if no cert present.
| `__ETCD_CA_CERT_FILE__`               | The path to the etcd CA file installed to the host, empty if no CA present.