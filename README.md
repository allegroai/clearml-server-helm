# TRAINS Server for Kubernetes Clusters Using Helm

##  Auto-Magical Experiment Manager & Version Control for AI

[![GitHub license](https://img.shields.io/badge/license-SSPL-green.svg)](https://img.shields.io/badge/license-SSPL-green.svg)
[![GitHub version](https://img.shields.io/github/release-pre/allegroai/trains-server.svg)](https://img.shields.io/github/release-pre/allegroai/trains-server.svg)
[![PyPI status](https://img.shields.io/badge/status-beta-yellow.svg)](https://img.shields.io/badge/status-beta-yellow.svg)

## Introduction

The **trains-server** is the backend service infrastructure for [TRAINS](https://github.com/allegroai/trains).
It allows multiple users to collaborate and manage their experiments.
By default, **TRAINS** is set up to work with the **TRAINS** demo server, which is open to anyone and resets periodically. 
In order to host your own server, you will need to install **trains-server** and point **TRAINS** to it.

**trains-server** contains the following components:

* The **TRAINS** Web-App, a single-page UI for experiment management and browsing
* RESTful API for:
    * Documenting and logging experiment information, statistics and results
    * Querying experiments history, logs and results
* Locally-hosted file server for storing images and models making them easily accessible using the Web-App

Use this repository to add **trains-server** to your Helm and then deploy **trains_server** on Kubernetes clusters using Helm.
 
## Prerequisites

* a Kubernetes cluster
* `kubectl` is installed and configured (see [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in the Kubernetes documentation)
* `helm` installed (see [Installing Helm](https://helm.sh/docs/using_helm/#installing-helm) in the Helm documentation)
* one node labeled `app: trains`

    **Important**: TRAINS-server deployment uses node storage. If more than one node is labeled as `app: trains` and you redeploy or update later, then TRAINS-server may not locate all of your data. 
    
## Modify the default values required by Elastic in Docker configuration file (see [Notes for production use and defaults](https://www.elastic.co/guide/en/elasticsearch/reference/master/docker.html#_notes_for_production_use_and_defaults))
1 - Connect to node you labeled as `app=trains`

If your system contains a `/etc/sysconfig/docker` Docker configuration file, edit it.
Add the options in quotes to the available arguments in the `OPTIONS` section:

    OPTIONS="--default-ulimit nofile=1024:65536 --default-ulimit memlock=-1:-1"

Otherwise, edit `/etc/docker/daemon.json` (if it exists) or create it (if it does not exist).

Add or modify the `defaults-ulimits` section as shown below. Be sure the `defaults-ulimits` section contains the `nofile` and `memlock` sub-sections and values shown.

**Note**: Your configuration file may contain other sections. If so, confirm that the sections are separated by commas (valid JSON format). For more information about Docker configuration files, see [Daemon configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file), in the Docker documentation.

The trains-server required defaults values are (json):

    {
        "default-ulimits": {
            "nofile": {
                "name": "nofile",
                "hard": 65536,
                "soft": 1024
            },
            "memlock":
            {
                "name": "memlock",
                "soft": -1,
                "hard": -1
            }
        }
    }

2 - Set the Maximum Number of Memory Map Areas

Elastic requires that the vm.max_map_count kernel setting, which is the maximum number of memory map areas a process can use, is set to at least 262144.

For CentOS 7, Ubuntu 16.04, Mint 18.3, Ubuntu 18.04 and Mint 19.x, we tested the following commands to set vm.max_map_count:

    echo "vm.max_map_count=262144" > /tmp/99-trains.conf
    sudo mv /tmp/99-trains.conf /etc/sysctl.d/99-trains.conf
    sudo sysctl -w vm.max_map_count=262144

For information about setting this parameter on other systems, see the [elastic documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode).

3 - Restart docker:

    sudo service docker restart
    
## <a name="deploying-trains-server">Deploying trains-server in Kubernetes Clusters Using Helm 
 
1. Add the `trains-server` repository to your Helm:

        helm repo add allegroai https://allegroai.github.io/trains-server-helm/

1. Confirm the `trains-server` repository is now in Helm:

        helm search trains

    The helm search results must include `allegroai/trains-server-chart`.

1. By default, the `trains-server` deployment uses storage on a single node (labeled `app=trains`).
To change the type of storage used (for example NFS), see [Configuring trains-server storage for NFS](#configure-trains-server-storage-nfs).

1. If you'd like `trains-agent` support in your `trains` Kubernetes cluster, create a local `values.yaml`, as specified in [Configuring trains-agent instances in your cluster](#configure-trains-agents).    

1. Install `trains-server-chart` on your cluster:

        helm install allegroai/trains-server-chart --namespace=trains --name trains-server
        
    Alternatively, in case you've created a local `values.yaml` file, use:
    
        helm install allegroai/trains-server-chart --namespace=trains --name trains-server --values values.yaml

    A  trains `namespace` is created in your cluster and **trains-server** is deployed in it.
   
        
## <a name="updating-trains-server"></a>Updating trains-server application using Helm

1. Update using new or updated `values.yaml`
        
        helm upgrade trains-server allegroai/trains-server-chart -f new-values.yaml
        
1. If there are no breaking changes, you can update your deployment to match repository version:

        helm upgrade trains-server allegroai/trains-server-chart
   
   **Important**: 
        
    * If you previously deployed a **trains-server**, you may encounter errors. If so, you must first delete old deployment using the following command:
    
            helm delete --purge trains-server
            
        After running the `helm delete` command, you can run the `helm install` command.
        
## Port Mapping

After **trains-server** is deployed, the services expose the following node ports:

* API server on `30008`
* Web server on `30080`
* File server on `30081`

## <a name="accessing-trains-server"></a>Accessing trains-server

Access **trains-server** by creating a load balancer and domain name with records pointing to the load balancer.

Once you have a load balancer and domain name set up, follow these steps to configure access to trains-server on your k8s cluster:

1. Create domain records

   * Create 3 records to be used for Web-App, File server and API access using the following rules: 
     * `app.<your domain name>` 
     * `files.<your domain name>`
     * `api.<your domain name>`
     
     (*for example, `app.trains.mydomainname.com`, `files.trains.mydomainname.com` and `api.trains.mydomainname.com`*)
2. Point the records you created to the load balancer
3. Configure the load balancer to redirect traffic coming from the records you created:
     * `app.<your domain name>` should be redirected to k8s cluster nodes on port `30080`
     * `files.<your domain name>` should be redirected to k8s cluster nodes on port `30081`
     * `api.<your domain name>` should be redirected to k8s cluster nodes on port `30008`

## <a name="configure-trains-agents"></a>Configuring trains-agent instances in your cluster

In order to create `trains-agent` instances as part of your deployment, create or update your local `values.yaml` file.
This `values.yaml` file should be used in your `helm install` command (see [Deploying trains-server in Kubernetes Clusters Using Helm](#deploying-trains-server)) or `helm upgrade` command (see [Updating trains-server application using Helm](#updating-trains-server))  

The file must contain the following values in the `agent` section:

- `numberOfTrainsAgents`: controls the number of trains-agent pods to be deployed. Each agent pod will listen for and execute experiments from the **trains-server**
- `nvidiaGpusPerAgent`: defines the number of GPUs required by each agent pod
- `trainsApiHost`: the URL used to access the trains API server, as defined in your load-balancer (usually `https://api.<your domain name>`, see [Accessing trains-server](#accessing-trains-server))
- `trainsWebHost`: the URL used to access the trains webservice, as defined in your load-balancer (usually `https://app.<your domain name>`, see [Accessing trains-server](#accessing-trains-server))
- `trainsFilesHost`: the URL used to access the trains fileserver, as defined in your load-balancer (usually `https://files.<your domain name>`, see [Accessing trains-server](#accessing-trains-server))

Additional optional values in the `agent` section include:

- `defaultBaseDocker`: the default docker image used by the agent running in the agent pod in order to execute an experiment. Default is `nvidia/cuda`.
- `agentVersion`: determines the specific agent version to be used in the deployment, for example `"==0.13.3"`. Default is `null` (use latest version)
- `trainsGitUser` / `trainsGitPassword`: GIT credentials used by `trains-agent` running the experiment when cloning the GIT repository defined in the experiment, if defined. Default is `null` (not used)
- `awsAccessKeyId` / `awsSecretAccessKey` / `awsDefaultRegion`: AWS account info used by `trains` when uploading files to an AWS S3 buckets (not required if only using the default `trains-fileserver`). Default is `null` (not used)
- `azureStorageAccount` / `azureStorageKey`: Azure account info used by trains when uploading files to MS Azure Blob Service (not required if only using the default `trains-fileserver`). Default is `null` (not used)

For example, the following `values.yaml` file requests 4 agent instances in your deployment (see [chart-example-values.yaml](https://github.com/allegroai/trains-server-k8s/blob/master/docs/chart-example-values.yaml)):
```yaml
agent:
  numberOfTrainsAgents: 4
  nvidiaGpusPerAgent: 1
  defaultBaseDocker: "nvidia/cuda"
  trainsApiHost: "https://api.trains.mydomain.com"
  trainsWebHost: "https://app.trains.mydomain.com"
  trainsFilesHost: "https://files.trains.mydomain.com"
  trainsGitUser: null
  trainsGitPassword: null
  awsAccessKeyId: null
  awsSecretAccessKey: null
  awsDefaultRegion: null
  azureStorageAccount: null
  azureStorageKey: null
```

## <a name="configure-trains-server-storage-nfs"></a>Configuring trains-server storage for NFS

The **trains-server** deployment uses a `PersistentVolume` of type `HostPath`, 
which uses a fixed path on the node labeled `app: trains`.

The existing chart supports changing the volume type to `NFS`, 
by setting the `use_nfs` value and configuring the NFS persistent volume using additional values in your local `values.yaml` file:
```yaml
storage:
  use_nfs: true
  nfs:
    server: "<nfs-server-ip-address>"
    base_path: "/nfs/path/for/trains/data"
``` 

## Additional Configuration for trains-server

You can also configure the **trains-server** for:
 
* fixed users (users with credentials)
* non-responsive experiment watchdog settings
 
For detailed instructions, see the [Optional Configuration](https://github.com/allegroai/trains-server#optional-configuration) section in the **trains-server** repository README file.
