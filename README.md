# proxmox-cluster-manifest

A repository for configuring the deployment manifests of my local proxmox cluster and detailing information about its setup. In here you will find useful examples of configuring an on-premise Kubernetes cluster that is running in a basic Proxmox environment for home testing. These manifests should not be considered production ready (unless otherwise stated).

## Load Balancing and Ingress Control

Load Balancing and Ingress Control provide crucial functionality in Kubernetes that allow services and deployments to be exposed outside of the cluster. At a top level, they each do the following:

- Load Balancer => Manages the distribution of traffic across cluster nodes and assigns external IPs to services running in the cluster so that they can be exposed to external clients.
- Ingress Controller => Manages access and routing of traffic to specific services in the cluster, can also enforce certain security rules.

#### Metallb Load Balancer

To provide Load Balancing functionality in the proxmox cluster I am using the OSS project Metallb. Installation instructions for Metallb can be found here:

https://metallb.universe.tf/installation/

Once deployed, Metallb must be configured before it will function. The configuration options are described in the following documentation:

https://metallb.universe.tf/configuration/

Additionally, the following sections provide additional notes about the configuration.

*IP Address Pool:*

In order to allow the load balancer to assign an external IP address, we need to give it a pool of available IPs. This is compeleted by providing an IP Address pool using a CIDR.
CIDR calculators can be useful for this task:

https://mxtoolbox.com/subnetcalculator.aspx

Enusre that the pool you provide does not have IPs that are already in use, as this could create potential conflicts.

*Layer 2 Configuration:*

The most basic way of configuring Metallb is to use a layer two network configuration. Layer two listens to ARP requests on your local network and returns the MAC address of the worker node so that the traffic can be routed to the correct location. In this way, no special configuration is required, other than telling Metallb that is needs to use a Layer Two configuration. The Layer Two configuration will automatically use the IP Address Pool configured previously when assigning external IP addresses for services.

#### Ingress NGINX

Ingress Nginx is an officially supported ingress controller for Kubernetes that is maintained by the Kubernetes project. It provides an Nginx configuration that allows internal routing to services to be configured for a single external IP. This limits the number of external IPs required by hosting different services under URL paths on that IP.

Installation instructions for Ingress Nginx can be found here:

https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

As this service is preconfigured as type LoadBalancer, it will automatically be assigned an external IP address by Metallb. All access to the cluster can then be configured through this single IP address.

In addition to the above documentation, the following guide shows the combined used of ingress nginx and metallb:

https://www.adaltas.com/en/2022/09/08/kubernetes-metallb-nginx/

## Argo CD

Argo CD is a tool for managing the deployment process of cluster configurations on a cluster. It allows you to perform the management of deployments in a git repository and will automatically sync changes to your cluster based on updates in the git repository. Installation of Argo CD can be achieved using the instructions at the following link:

https://argo-cd.readthedocs.io/en/stable/getting_started/

In addition to performing these installation instructions, there are some additional steps for configuration which are outlined in the following sections.

*Service Modification:*

**The following instructions should only be followed if you do not want to expose Argo CD through an Ingress Controller (Ingress NGINX, etc.).**

In order to expose the Argo CD server service externally, we must modify it to use type LoadBalancer. Apply the file argocd-load-balancer-service.yaml from the **argocd** folder and the service will be updated to add the LoadBalancer type.

From here, inspecting the service using K9S or Kubectl will show the external IP that has been assigned to Argo CD.

*Ingress Control:*

To run Argo CD behind the nginx ingress controller, run the **argocd-ingress.yaml** file using Kubectl. In addition, some setup and configuration of the Argo CD server configmap is required when running with no SSL behind the nginx ingress:

https://stackoverflow.com/questions/76624577/argocd-ingress-kubernetes-too-many-redirects-even-with-insecure-nginx

Additionally, if you want to run Argo CD behind a non-root ingress path, then you need to reconfigure the Argo CD ingress and argocd-server deployment to run behind the non root path. The files in the repository are currently configured to run Argo CD behind the path "/argocd". More information on requirements to do this can be found here (baring in mind that nginx is configured through the annotations in the ingress configuration):

https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#argocd-server-and-ui-root-path-v153

#### Configuring Argo CD

Once we have installed Argo CD and configured into run behind the Load Balancer and Ingress Controller, we can start to deploy things using it. There are a few base configurations we must perform to make this easier. These configurations are outlined in the following sections.

*App of Apps Configuration:*

The most common way of managing an Argo CD installation is by connecting it to a git repository that holds all of the Kubernetes manifests you want to deploy. To make these easier to manage and allow folder structures in the git repository, an "App of Apps" configuration is used as an initial (manual) deployment to Argo CD. This means all Application deployments can be managed in the git repository, and only one application needs to be manually applied on the cluster.

More information on App of Apps can be found here:

https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/

To create the initial App of Apps installation, apply the **app-of-apps-configuration.yaml** file from the **argocd** folder against the cluster.

```bash
kubectl apply -f app-of-apps-configuration.yaml
```

## Kustomize

Kustomize is a tool natively built into Kubernetes and kubectl that allows configuration of kubernetes manifests programatically. It does this through a series of "overlays" that dynamically reconfigure values for a particular deployment. This is particularly useful when deploying to different environments (DEV, STAGING, PRODUCTION).

More information on Kustomize can be found here:

https://kubectl.docs.kubernetes.io/references/kustomize/

In addition, a helpful contact at VM Ware has created a sample Kustomization repository that demonstrates its use well:

https://github.com/markusrt/kustomize-sample

## Kubernetes Metrics Server

The Kubernetes Metrics Server is a set of services provided by the Kubernetes SIG that provide basic metrics about Pods for the purposes of Horizontal Pod Autoscaling. Further inforamtion about the Metrics Server and its deployment can be found here:

https://github.com/kubernetes-sigs/metrics-server

The metrics server is a requirement for using HPAs and fully leveraging the declaritive power of Kubernetes.

*Deployment Manifests when using HPAs:*

When using a HPA, do not define the required number of replicas in the deployment file! The HPAs **minReplicas** and **maxReplicas** manifest parameters will control this for you.

## Strimzi

Strimzi is an operator and controller that manages the deployment of Kafka within a Kubernetes cluster. It also adds a set of Custom Resource Definitions for Kubernetes that allow Topics and Kafka Clusters to be configured declaritively.

Strimzi can be deployed using a Helm chart (and is configured to do so via Argo CD in this repository), example values for the helm chart can be found here:

https://github.com/strimzi/strimzi-kafka-operator/blob/main/helm-charts/helm3/strimzi-kafka-operator/values.yaml

More information on deploying Strimzi with Helm can be found here:

https://strimzi.io/docs/operators/latest/deploying#deploying-cluster-operator-helm-chart-str

Additionally, the top level Strimzi docs can be found here:

https://strimzi.io/documentation/

#### Kafka Topic

The KafkaTopic resource can be configured in a Kubernetes manifest YAML to create Kafka topics declaritively in the cluster.
When doing this, bare in mind the following configuration options:

```yaml
metadata:
    labels:
        strimzi.io/cluster:     # The name of the KAFKA cluster the topic belongs to
```

#### UI for Apache Kafka

The UI for Apache Kafka application provided by Provectus gives a very nice method of visualising Kafka Clusters, Brokers, Topics and Messages.
More information about the application can be found here:

https://docs.kafka-ui.provectus.io/

More information about configuration of the application can be found here:

https://docs.kafka-ui.provectus.io/configuration/configuration-wizard

Additionally, in this repository an application manifest for Argo CD has been created to deploy this UI on the cluster.
