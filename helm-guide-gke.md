# Helm 
Helm is the package manager for Kubernetes. It is the easiest way to install new applications on your Kubernetes cluster.

This website contains a complete directory of all charts available to install from the official repository. We have grouped them into categories to make browsing for new applications quick and easy.

But how do you actually install them? The first step is to install Helm on your laptop by following this guide.

Make sure you have kubectl working with a valid context pointing to an existing cluster, or local minikube installation, then simply run helm init and it will automatically configure everything required to start installing charts.

Behind the scenes running helm init will install the server side component called Tiller. This all happens magically so you shouldn’t need to do anything.

If you find an application that you’d like to install while browsing this site simply click the visit website button on the listing. You’ll generally find the install command that you can copy and paste into your terminal.

# Helm Setup Guide on Google Kubernetes Engine (GKE)

This page describes how to set up Helm to properly support the Couchbase Autonomous Operator in your Kubernetes or OpenShift environment. Setting up Helm consists of installing the Helm client (**helm**) on your computer, and installing the Helm server (Tiller) on your Kubernetes or OpenShift cluster. Once you’ve set up Helm, you can then use official Couchbase Helm charts to deploy the Operator and the Couchbase Server cluster.

## Install Helm

### Install the Helm Client 
Follow Helm’s [official steps](https://docs.helm.sh/using_helm/#installing-helm) for installing **helm** on your operating system.

It’s helpful to install **helm** on the same computer where you normally run **kubectl** or **oc**.


### Install the Helm Server (Tiller)
After you’ve installed helm, you’ll need to install Tiller — the server portion of Helm that runs inside of your Kubernetes cluster.

Follow Helm’s [official steps](https://docs.helm.sh/using_helm/#installing-tiller) for installing Tiller on your Kubernetes cluster.

#### Installing Tiller for Development

For development use-cases, the Tiller service can be given access to deploy charts into any namespace of your Kubernetes cluster. This also means that resources created by the chart, such as the custom resource definition (CRD), are allowed when Tiller is given this level of privilege.

To create RBAC rules for Tiller with cluster-wide access create the following ServiceAccount and save to **rbac-tiller.yaml**

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
    
Now install the ServiceAccount and initialize Helm

```
kubectl create -f rbac-tiller.yaml
helm init --service-account tiller
```

#### Installing Tiller for Production
For production use-cases, it’s recommended that you restrict the Tiller service to deploying charts only into namespaces that will be used by Helm. This ensures that your applications are operating in the scope that you’ve specified.

To create RBAC rules for Tiller that have restricted access, refer to the Helm documentation about deploying Tiller in a namespace such that it’s [restricted to deploying resources only in that namespace](https://docs.helm.sh/using_helm/#example-deploy-tiller-in-a-namespace-restricted-to-deploying-resources-only-in-that-namespace).

**Important:** When Tiller is restricted to a single namespace, the Operator won’t be able to automatically install the custom resource definition (CRD) that’s required to create Couchbase clusters. See the [production deployment instructions](https://docs.couchbase.com/operator/current/helm-setup-guide.html#deploy-production) for information about manually installing the CRD.

## Deploying the Operator and Couchbase Server

Two [Helm charts](https://docs.helm.sh/using_helm/#three-big-concepts) are available for deploying Couchbase. The [Couchbase Operator Chart](https://docs.couchbase.com/operator/current/helm-operator-config.html) deploys the [admission controller](https://docs.couchbase.com/operator/current/install-admission-controller.html) and the Operator itself. The [Couchbase Cluster Chart](https://docs.couchbase.com/operator/current/helm-cluster-config.html) deploys the Couchbase Server cluster.

For production deployments, you’ll only use the Operator Chart. For development environments, the Couchbase Cluster Chart is available to help you quickly set up a test cluster.

### Deploying for Development (Quick Start)
To quickly deploy the admission controller and Operator, as well as a Couchbase Server cluster for development purposes:

1. Add the chart repository to **helm**:
```
helm repo add couchbase https://couchbase-partners.github.io/helm-charts/
```
2. Install the Operator Chart:
```
helm install couchbase/couchbase-operator
```
3. Install the Couchbase Cluster Chart:
```
helm install couchbase/couchbase-cluster
```

### Deploying for Production
For production deployments, additional customization is required in order to restrict the RBAC roles of the Operator to a single namespace. Note that this restriction requires that the CRD be manually created. This is because the CRD is a cluster-wide resource and cannot be created by the Operator when it has restricted access.

The following steps show the minimum requirements for deploying the Operator Chart under the above constraints for production usage:

1. Download the appropriate Operator [package](https://www.couchbase.com/downloads) and unpack it on the same computer where you normally run **kubectl** or **oc**.

2. Included in the package is a file called **crd.yaml**. Use the following command to install it on your Kubernetes cluster:
```
kubectl create -f crd.yaml
```
3. Add the chart repository to **helm**:
```
helm repo add couchbase https://couchbase-partners.github.io/helm-charts/
```
4. Install the Operator Chart:
```
helm install --set rbac.clusterRoleAccess=false couchbase/couchbase-operator
```
The above command uses the **--set** option to override the chart’s default **rbac.clusterRoleAccess** parameter and sets it to **false**. This is the most important parameter to modify for production environments. However, you may need to override additional parameters to meet the needs of your environment. Therefore, it’s recommended that you keep all of your overrides in a YAML file, and use the the **--values** option instead of the **--set** option when you install the chart. See [Customizing Helm Charts](https://docs.couchbase.com/operator/current/helm-setup-guide.html#customize-charts) for more information.


When you install a chart, Helm autogenerates a name for the release (usually a unique set of dictionary words). Helm prepends this name to all of the objects and resources that the chart creates.

**Note:** If you plan to use Helm to install multiple instances of the Operator, you should consider giving each [release](https://docs.helm.sh/glossary/#release) a unique name to help you more easily identify the resources that are associated with each release. You can specify a name for the release during chart installation by using the **--name** flag:
```
helm install -n <unique-name> --set rbac.clusterRoleAccess=false couchbase/couchbase-operator
```

5. Deploy the Couchbase Server cluster using the traditional method of using a [CouchbaseCluster specification](https://docs.couchbase.com/operator/current/deploying-couchbase.html).

Note that you’re not deploying the Couchbase cluster using a Helm chart. Due to the complex structure of production configurations (which usually include server groups and persistent volumes), Couchbase clusters are better expressed and managed directly using a cluster spec.

## Customizing Helm Charts

The official Couchbase Helm charts are great to use "as-is" for quickly deploying Couchbase on Kubernetes platforms, but you’ll undoubtedly need to customize them for your specific development and production needs. All of the values that are exposed by the charts are available within the **values.yaml** file of the [Couchbase Operator Chart](https://docs.couchbase.com/operator/current/helm-operator-config.html) and the [Couchbase Cluster Chart](https://docs.couchbase.com/operator/current/helm-cluster-config.html).

You can customize each chart by using [overrides](https://docs.helm.sh/using_helm/#customizing-the-chart-before-installing). There are two methods to specify overrides during chart installation: **--values** and **--set**.

### --values 
The **--values** option is the preferred method because it allows you to keep your overrides in a YAML file, rather than specifying them all on the command line.

1. Create a YAML file and add your overrides to it.

Here's an example called **myvalues.yaml**:
```
couchbaseOperator:
  imagePullPolicy: Always
```
2. Specify your overrides file when you install the chart:
```
helm install --values myvalues.yaml couchbase/coucbase-operator
```
The values in your overrides file (**myvalues.yaml**) will override their counterparts in the chart’s **values.yaml** file. Any values in **values.yaml** that weren’t overridden will keep their defaults.


### --set
If you only need to make minor customizations, you can specify them on the command line by using the **--set** option.

Here’s an example:
```
helm install --set rbac.clusterRoleAccess=false couchbase/couchbase-operator
```
This would translate to the following in the **values.yaml** of the chart:
```
rbac:
  clusterRoleAccess: true
```

## Chart Versions

The **helm install** command will always the highest version of a chart. To list the versions of the chart that are available for installation, you can run the helm search command:
```
helm search --versions couchbase/couchbase-operator
```

```
NAME                        	CHART VERSION	APP VERSION	DESCRIPTION
couchbase/couchbase-operator	0.1.2 	  1.2  	A Helm chart for Kubernetes
```

Here, the **CHART VERSION** is **0.1.2**, and the **APP VERSION** (the Couchbase Operator version) is 1.2.

To install a specific version of a chart, include the **--version** argument during installation:
```
helm install --version 0.1.2 couchbase/couchbase-operator
```

If you’re having trouble finding or installing a specific version of a chart, use the **helm repo update** command to ensure that you have the latest list of charts.


---

# GKE Starting guide

## Prerequisites

* The [Google Cloud SDK](https://cloud.google.com/sdk/) is installed

---

1. Create project
![project]()

2. Make sure that Python 2.7 is installed on your system

```
$ python -V
Python 2.7.10
```

GKE SDK Init

```
$ gcloud init --console-only
Welcome! This command will take you through the configuration of gcloud.

Your current configuration has been set to: [default]

You can skip diagnostics next time by using the following flag:
  gcloud init --skip-diagnostics

Network diagnostic detects and fixes local network connection issues.
Checking network connection...done.                                                                                                            
Reachability Check passed.
Network diagnostic passed (1/1 checks passed).

You must log in to continue. Would you like to log in (Y/n)?  Y

Go to the following link in your browser:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob&prompt=select_account&response_type=code&client_id=32555940559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline


Enter verification code: 4/WAHprN3D_phCo1KwrXUsdywps331h9qe-qMVfcfGOZaxkm_YIjvZpfg
You are logged in as: [jose.molina@couchbase.com].

Pick cloud project to use: 
 [1] helmchart
 [2] helmchart-242010
 [3] Create a new project
Please enter numeric choice or text value (must exactly match list 
item):  1

Your current project has been set to: [helmchart].

Not setting default zone/region (this feature makes it easier to use
[gcloud compute] by setting an appropriate default value for the
--zone and --region flag).
See https://cloud.google.com/compute/docs/gcloud-compute section on how to set
default compute region and zone manually. If you would like [gcloud init] to be
able to do this for you the next time you run it, make sure the
Compute Engine API is enabled for your project on the
https://console.developers.google.com/apis page.

Created a default .boto configuration file at [/Users/josemolina/.boto]. See this file and
[https://cloud.google.com/storage/docs/gsutil/commands/config] for more
information about configuring Google Cloud Storage.
Your Google Cloud SDK is configured and ready to use!

* Commands that require authentication will use jose.molina@couchbase.com by default
* Commands will reference project `helmchart` by default
Run `gcloud help config` to learn how to change individual settings

This gcloud configuration is called [default]. You can create additional configurations if you work with multiple accounts and/or projects.
Run `gcloud topic configurations` to learn more.

Some things to try next:

* Run `gcloud --help` to see the Cloud Platform services you can interact with. And run `gcloud help COMMAND` to get help on any gcloud command.
* Run `gcloud topic --help` to learn about advanced features of the SDK like arg files and output formatting
```

---

When the gcloud command has been installed and added to your PATH, log in with the following command:

```
gcloud auth login
```
---
```
$ gcloud auth login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&prompt=select_account&response_type=code&client_id=32555940559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline


WARNING: `gcloud auth login` no longer writes application default credentials.
If you need to use ADC, see:
  gcloud auth application-default --help

You are now logged in as [jose.molina@couchbase.com].
Your current project is [helmchart].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID
```
---


**Tip:** The gcloud command can support multiple logins. You can select the user to run as with:
```
gcloud config set account john.doe@acme.com
```
You can also remove the login authentication locally with:
```
gcloud auth revoke john.doe@acme.com
```


The project must also be set so that resources are provisioned into it 
```
gcloud config set project my-project
```

```
$ gcloud config set project helmchart
Updated property [core/project].
```

## Network Setup

```
gcloud compute networks create helm-network
API [compute.googleapis.com] not enabled on project [949438218655]. 
Would you like to enable and retry (this will take a few minutes)? 
(y/N)?  y

Enabling service [compute.googleapis.com] on project [949438218655]...
Waiting for async operation operations/acf.4a188534-cefd-408c-85f3-4596c5be5c18 to complete...
Operation finished successfully. The following command can describe the Operation details:
 gcloud services operations describe operations/tmo-acf.4a188534-cefd-408c-85f3-4596c5be5c18
Created [https://www.googleapis.com/compute/v1/projects/helmchart/global/networks/helm-network].
NAME          SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
helm-network  AUTO         REGIONAL

Instances on this network will not be reachable until firewall rules
are created. As an example, you can allow all internal traffic between
instances as well as SSH, RDP, and ICMP by running:

$ gcloud compute firewall-rules create <FIREWALL_NAME> --network helm-network --allow tcp,udp,icmp --source-ranges <IP_RANGE>
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network helm-network --allow tcp:22,tcp:3389,icmp
```

cluster 1 network
```
$ gcloud compute networks subnets create my-subnet-us-east1 --network helm-network --region us-east1 --range 10.0.0.0/12
Created [https://www.googleapis.com/compute/v1/projects/helmchart/regions/us-east1/subnetworks/my-subnet-us-east1].
NAME                REGION    NETWORK       RANGE
my-subnet-us-east1  us-east1  helm-network  10.0.0.0/12
```
cluster 2 network
```
$ gcloud compute networks subnets create my-subnet-us-west1 --network helm-network --region us-west1 --range 10.16.0.0/12
Created [https://www.googleapis.com/compute/v1/projects/helmchart/regions/us-west1/subnetworks/my-subnet-us-west1].
NAME                REGION    NETWORK       RANGE
my-subnet-us-west1  us-west1  helm-network  10.16.0.0/12
```

firewall
```
$ gcloud compute firewall-rules create my-network-allow-all-private --network helm-network --direction INGRESS --source-ranges 10.0.0.0/8 --allow all
Creating firewall...⠛Created [https://www.googleapis.com/compute/v1/projects/helmchart/global/firewalls/my-network-allow-all-private].         
Creating firewall...done.                                                                                                                      
NAME                          NETWORK       DIRECTION  PRIORITY  ALLOW  DENY  DISABLED
my-network-allow-all-private  helm-network  INGRESS    1000      all          False
```

enable kubernetes engine api

https://console.cloud.google.com/apis/api/container.googleapis.com/overview?project=helmchart


```
$gcloud container clusters create my-cluster-us-east1 --cluster-version 1.13.6-gke.0 --region us-east1 --network helm-network --subnetwork my-subnet-us-east1
WARNING: In June 2019, node auto-upgrade will be enabled by default for newly created clusters and node pools. To disable it, use the `--no-enable-autoupgrade` flag.
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `--[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
WARNING: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s). 
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
ERROR: (gcloud.container.clusters.create) ResponseError: code=403, message=
	(1) insufficient regional quota to satisfy request: resource "CPUS": request requires '9.0' and is short '1.0'. project has a quota of '8.0' with '8.0' available. View and manage quotas at https://console.cloud.google.com/iam-admin/quotas?usage=USED&project=helmchart
	(2) insufficient regional quota to satisfy request: resource "IN_USE_ADDRESSES": request requires '9.0' and is short '1.0'. project has a quota of '8.0' with '8.0' available. View and manage quotas at https://console.cloud.google.com/iam-admin/quotas?usage=USED&project=helmchart.
```

gcloud compute machine-types list
```
$ gcloud compute machine-types list
NAME             ZONE                       CPUS  MEMORY_GB  DEPRECATED
f1-micro         us-central1-f              1     0.60
g1-small         us-central1-f              1     1.70

... etc

n1-standard-4    asia-northeast2-a          4     15.00
n1-standard-64   asia-northeast2-a          64    240.00
n1-standard-8    asia-northeast2-a          8     30.00
n1-standard-96   asia-northeast2-a          96    360.00
```



---
# day 2

restarting ...

## 1. setting my default region/zone
```
$ gcloud config set compute/region europe-west3
Updated property [compute/region].
josemolina@EMEA-MAC0144:~/couchbase/kubernetes/gke$ gcloud config set compute/zone europe-west3-a
Updated property [compute/zone].
```


## 2. login

```
$ gcloud auth login
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&prompt=select_account&response_type=code&client_id=32555940559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline


WARNING: `gcloud auth login` no longer writes application default credentials.
If you need to use ADC, see:
  gcloud auth application-default --help

You are now logged in as [jose.molina@couchbase.com].
Your current project is [helmchart].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID
  ```

## 3. Setup network  
  
```  
$ gcloud compute networks create helm-network --subnet-mode custom
Created [https://www.googleapis.com/compute/v1/projects/helmchart/global/networks/helm-network].
NAME          SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
helm-network  CUSTOM       REGIONAL

Instances on this network will not be reachable until firewall rules
are created. As an example, you can allow all internal traffic between
instances as well as SSH, RDP, and ICMP by running:

$ gcloud compute firewall-rules create <FIREWALL_NAME> --network helm-network --allow tcp,udp,icmp --source-ranges <IP_RANGE>
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network helm-network --allow tcp:22,tcp:3389,icmp
```

subnet 1
```
$ gcloud compute networks subnets create my-subnet-europe-west3 --network helm-network --region europe-west3 --range 10.0.0.0/12
Created [https://www.googleapis.com/compute/v1/projects/helmchart/regions/europe-west3/subnetworks/my-subnet-europe-west3].
NAME                    REGION        NETWORK       RANGE
my-subnet-europe-west3  europe-west3  helm-network  10.0.0.0/12
```

subnet2

```
$ gcloud compute networks subnets create my-subnet-europe-west1 --network helm-network --region europe-west1 --range 10.16.0.0/12
Created [https://www.googleapis.com/compute/v1/projects/helmchart/regions/europe-west1/subnetworks/my-subnet-europe-west1].
NAME                    REGION        NETWORK       RANGE
my-subnet-europe-west1  europe-west1  helm-network  10.16.0.0/12
```

Firewall rules

```
$ gcloud compute firewall-rules create my-network-allow-all-private --network helm-network --direction INGRESS --source-ranges 10.0.0.0/8 --allow all
Creating firewall...⠧Created [https://www.googleapis.com/compute/v1/projects/helmchart/global/firewalls/my-network-allow-all-private].                                                                          
Creating firewall...done.                                                                                                                                                                                       
NAME                          NETWORK       DIRECTION  PRIORITY  ALLOW  DENY  DISABLED
my-network-allow-all-private  helm-network  INGRESS    1000      all          False
```

## kubernetes cluster setup

1. Check cluster-version

```
e$ gcloud container get-server-config --zone europe-west3-a
Fetching server config for europe-west3-a
defaultClusterVersion: 1.12.7-gke.10
defaultImageType: COS
validImageTypes:
- COS_CONTAINERD
- COS
- UBUNTU
validMasterVersions:
- 1.13.6-gke.0
- 1.13.5-gke.10
- 1.12.7-gke.17
- 1.12.7-gke.10
- 1.12.6-gke.11
- 1.11.9-gke.13
- 1.11.9-gke.8
- 1.11.8-gke.6
validNodeVersions:
- 1.13.6-gke.0
- 1.13.5-gke.10
- 1.12.7-gke.17
- 1.12.7-gke.10
- 1.12.7-gke.7
...
- 1.7.12-gke.2
- 1.6.13-gke.1
```

create cluster 1

```
$ gcloud container clusters create my-cluster-europe-west3-a --machine-type n1-standard-2 --cluster-version 1.13.6-gke.0 --zone europe-west3-a --network helm-network --subnetwork my-subnet-europe-west3 --num-nodes 3
WARNING: In June 2019, node auto-upgrade will be enabled by default for newly created clusters and node pools. To disable it, use the `--no-enable-autoupgrade` flag.
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `--[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
WARNING: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s). 
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
Creating cluster my-cluster-europe-west3-a in europe-west3-a... Cluster is being health-checked (master is healthy)...done.                                                                                     
Created [https://container.googleapis.com/v1/projects/helmchart/zones/europe-west3-a/clusters/my-cluster-europe-west3-a].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west3-a/my-cluster-europe-west3-a?project=helmchart
kubeconfig entry generated for my-cluster-europe-west3-a.
NAME                       LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
my-cluster-europe-west3-a  europe-west3-a  1.13.6-gke.0    35.234.69.209  n1-standard-2  1.13.6-gke.0  3          RUNNING
```

create cluster 2

```
$ gcloud container clusters create my-cluster-europe-west1-b --machine-type n1-standard-2 --cluster-version 1.13.6-gke.0 --zone europe-west1-b --network helm-network --subnetwork my-subnet-europe-west1 --num-nodes 3
WARNING: In June 2019, node auto-upgrade will be enabled by default for newly created clusters and node pools. To disable it, use the `--no-enable-autoupgrade` flag.
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `--[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
WARNING: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s). 
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
Creating cluster my-cluster-europe-west1-b in europe-west1-b... Cluster is being health-checked (master is healthy)...done.                                                                                     
Created [https://container.googleapis.com/v1/projects/helmchart/zones/europe-west1-b/clusters/my-cluster-europe-west1-b].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-b/my-cluster-europe-west1-b?project=helmchart
kubeconfig entry generated for my-cluster-europe-west1-b.
NAME                       LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
my-cluster-europe-west1-b  europe-west1-b  1.13.6-gke.0    35.187.73.153  n1-standard-2  1.13.6-gke.0  3          RUNNING
```


list clusters
```
$ gcloud container clusters list
NAME                       LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
my-cluster-europe-west1-b  europe-west1-b  1.13.6-gke.0    35.187.73.153  n1-standard-2  1.13.6-gke.0  3          RUNNING
my-cluster-europe-west3-a  europe-west3-a  1.13.6-gke.0    35.234.69.209  n1-standard-2  1.13.6-gke.0  3          RUNNING
```

get credentials
cluster1

```
$ gcloud container clusters get-credentials my-cluster-europe-west3-a --zone europe-west3-a --project helmchart
Fetching cluster endpoint and auth data.
kubeconfig entry generated for my-cluster-europe-west3-a.
```

cluster2
```
$ gcloud container clusters get-credentials my-cluster-europe-west1-b --zone europe-west1-b --project helmchart
Fetching cluster endpoint and auth data.
kubeconfig entry generated for my-cluster-europe-west1-b.
```

Kubernetes Environment Setup

```
$ kubectl create clusterrolebinding jose-molina-admin-binding --clusterrole cluster-admin --user $(gcloud config get-value account)
clusterrolebinding.rbac.authorization.k8s.io/jose-molina-admin-binding created
```

## Helm Setup

### Init
```
$ nano rbac-tiller-development.yaml
```

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Now install the ServiceAccount and initialize Helm
```
$ kubectl create -f rbac-tiller-development.yaml 
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

```
$ helm init --service-account tiller
$HELM_HOME has been configured at /Users/josemolina/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```

### Deploying the operator and Couchbase Server ( Development )

1. Add the chart repository to helm:
```
helm repo add couchbase https://couchbase-partners.github.io/helm-charts/
```

2. Install the operator chart

```
$ helm install couchbase/couchbase-operator
NAME:   wizened-wolverine
LAST DEPLOYED: Wed May 29 11:00:22 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRoleBinding
NAME                                              AGE
wizened-wolverine-couchbase-admission-controller  1s

==> v1/Deployment
NAME                                              READY  UP-TO-DATE  AVAILABLE  AGE
wizened-wolverine-couchbase-admission-controller  0/1    1           0          1s
wizened-wolverine-couchbase-operator              0/1    1           0          1s

==> v1/Pod(related)
NAME                                                             READY  STATUS             RESTARTS  AGE
wizened-wolverine-couchbase-admission-controller-7f9b8d4d8b2btt  0/1    ContainerCreating  0         1s
wizened-wolverine-couchbase-operator-698554dbb-hfrkm             0/1    ContainerCreating  0         1s

==> v1/Secret
NAME                                              TYPE    DATA  AGE
wizened-wolverine-couchbase-admission-controller  Opaque  2     1s

==> v1/Service
NAME                                              TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
wizened-wolverine-couchbase-admission-controller  ClusterIP  10.59.247.202  <none>       443/TCP  1s

==> v1/ServiceAccount
NAME                                              SECRETS  AGE
wizened-wolverine-couchbase-admission-controller  1        1s
wizened-wolverine-couchbase-operator              1        1s

==> v1beta1/ClusterRole
NAME                                              AGE
wizened-wolverine-couchbase-admission-controller  1s
wizened-wolverine-couchbase-operator              1s

==> v1beta1/ClusterRoleBinding
NAME                                  AGE
wizened-wolverine-couchbase-operator  1s

==> v1beta1/MutatingWebhookConfiguration
NAME                                              AGE
wizened-wolverine-couchbase-admission-controller  1s

==> v1beta1/ValidatingWebhookConfiguration
NAME                                              AGE
wizened-wolverine-couchbase-admission-controller  1s


NOTES:
1. couchbase-operator deployed. Check the couchbase-operator logs
  kubectl logs -f deployment/wizened-wolverine-couchbase-operator  --namespace default

2. admission-controller deployed. Check the admission-controller logs
  kubectl logs -f deployment/wizened-wolverine-couchbase-admission-controller --namespace default
```

3. Install the Couchbase Cluster Chart:

```
$ helm install couchbase/couchbase-cluster
NAME:   intentional-shrimp
LAST DEPLOYED: Wed May 29 11:02:36 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/CouchbaseCluster
NAME                                  AGE
intentional-shrimp-couchbase-cluster  0s

==> v1/Secret
NAME                                  TYPE    DATA  AGE
intentional-shrimp-couchbase-cluster  Opaque  2     0s


NOTES:
1. Get couchbase cluster status
  kubectl describe --namespace default couchbasecluster intentional-shrimp-couchbase-cluster

2. Connect to Admin console
  kubectl port-forward --namespace default intentional-shrimp-couchbase-cluster-0000 8091:8091
  # open http://localhost:8091

```


## Custom Charts

```
$ helm install --values myvalues-clustercfg.yaml --set couchbaseCluster.name=my-cluster-west1b couchbase/couchbase-cluster
NAME:   falling-bird
LAST DEPLOYED: Wed May 29 11:50:51 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/CouchbaseCluster
NAME               AGE
my-cluster-west1b  1s

==> v1/Secret
NAME                            TYPE    DATA  AGE
falling-bird-my-cluster-west1b  Opaque  2     1s


NOTES:
1. Get couchbase cluster status
  kubectl describe --namespace default couchbasecluster my-cluster-west1b

2. Connect to Admin console
  kubectl port-forward --namespace default my-cluster-west1b-0000 8091:8091
  # open http://localhost:8091

```

update helm chart

```
$ helm upgrade --install falling-bird --values myvalues-clustercfg.yaml couchbase/couchbase-cluster
Release "falling-bird" has been upgraded.
LAST DEPLOYED: Wed May 29 12:42:13 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/CouchbaseCluster
NAME                            AGE
falling-bird-couchbase-cluster  0s

==> v1/Secret
NAME                            TYPE    DATA  AGE
falling-bird-couchbase-cluster  Opaque  2     1s


NOTES:
1. Get couchbase cluster status
  kubectl describe --namespace default couchbasecluster falling-bird-couchbase-cluster

2. Connect to Admin console
  kubectl port-forward --namespace default falling-bird-couchbase-cluster-0000 8091:8091
  # open http://localhost:8091

```


## config XDCR

falling-bird-couchbase-cluster-0000.falling-bird-couchbase-cluster.default.svc
```
$ kubectl get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
intentional-shrimp-couchbase-cluster-0000                         1/1     Running   0          115m
intentional-shrimp-couchbase-cluster-0001                         1/1     Running   0          114m
intentional-shrimp-couchbase-cluster-0002                         1/1     Running   0          114m
wizened-wolverine-couchbase-admission-controller-7f9b8d4d8b2btt   1/1     Running   0          117m
wizened-wolverine-couchbase-operator-698554dbb-hfrkm              1/1     Running   0          117m
```

```
$ kubectl get pod intentional-shrimp-couchbase-cluster-0000 -o yaml | grep hostIP
  hostIP: 10.0.0.7
```

```
$ kubectl get service intentional-shrimp-couchbase-cluster-0000-exposed-ports
NAME                                                      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                         AGE
intentional-shrimp-couchbase-cluster-0000-exposed-ports   NodePort   10.59.250.69   <none>        8091:30768/TCP,18091:31941/TCP,8092:31558/TCP,18092:30769/TCP,11210:30014/TCP,11207:30862/TCP   115m
```


