# Tutorial: Running your 4G Network on Ubuntu

## Overview

[Magma](https://magmacore.org/) is an open-source software platform that gives network operators 
an open, flexible and extendable mobile core network solution.

[Juju](https://juju.is/) is a tool used to deploy cloud infrastructure and applications and manage their operations 
from Day 0 through Day 2.

In this tutorial, we will use Juju to deploy the Magma 4G core as well as a simulated radio
and a cellphone from the [srsRAN](https://www.srslte.com/) project to showcase that our network 
is actually working.

### Table of Contents

1. [Overview](#Overview)
2. [Getting Started](#Getting Started)
3. [Deploying Magma's network management software](#Deploying Magma's network management software)
4. [Deploying the 4G core](#Deploying the 4G core)
5. [Deploying a radio simulator](#Deploying a radio simulator)
6. [Destroying the environment](#Destroying the environment)


## Getting Started

This tutorial has been written to work on a Ubuntu 22.04 computer with at least 16GB of RAM, 8 cores
and 100GB of storage.


### Installing dependencies

First, open a terminal window and install Juju, Multipass and kubectl.

```bash
ubuntu@host:~$ sudo snap install juju
ubuntu@host:~$ sudo snap install multipass
ubuntu@host:~$ sudo snap install kubectl --classic
```

Set the multipass backend to `lxd`:

```bash
ubuntu@host:~$ multipass set local.driver=lxd
```

Create a new network:
```bash
ubuntu@host:~$ lxc network create magma --type=bridge
```
### Creating the environment

First, create 3 virtual machines using multipass.

```bash
ubuntu@host:~$ multipass launch --name magma-orchestrator --mem 8G --disk 40G --cpus 4 --network name=magma,mode=manual
ubuntu@host:~$ multipass launch --name magma-access-gateway
ubuntu@host:~$ multipass launch --name srsran
```

List the created virtual machines and their addresses:

```bash
ubuntu@host:~$ multipass list
Name                    State             IPv4             Image
magma-access-gateway    Running           10.24.157.231    Ubuntu 22.04 LTS
magma-orchestrator      Running           10.24.157.52     Ubuntu 22.04 LTS
srsran                  Running           10.24.157.67     Ubuntu 22.04 LTS
```

Note the addresses associated with each virtual machine, we will need those later. Yours will be 
different of course.

Now, connect to the virtual machine that we named `magma-orchestrator`:

```bash
ubuntu@host:~$ multipass shell magma-orchestrator
```

Then install MicroK8s and configure the network:

```bash
ubuntu@magma-orchestrator:~$ sudo snap install microk8s --classic
ubuntu@magma-orchestrator:~$ sudo ufw default allow routed
```

Add the ubuntu user to the microk8s group:

```bash
ubuntu@magma-orchestrator:~$ sudo usermod -a -G microk8s ubuntu
ubuntu@magma-orchestrator:~$ sudo chown -f -R ubuntu ~/.kube
ubuntu@magma-orchestrator:~$ newgrp microk8s
```

Enable the following microk8s add-ons:

```bash
ubuntu@magma-orchestrator:~$ microk8s enable dns storage
```

Enable the metallb add-on. Here we need to provide a range of 5 IP addresses on the same subnet
that is already used by the VM. Make sure to use a range that does not include any of the 3 addresses
provided to the VM's.

```bash
ubuntu@magma-orchestrator:~$ microk8s enable metallb:10.24.157.80-10.24.157.85
```

Output the kubernetes configuration to a file:

```bash
ubuntu@magma-orchestrator:~$ microk8s config > config
```

Now, from the host terminal, retrieve the configuration file and place it under the `~/.kube/` 
directory:

```bash
ubuntu@host:~$ multipass transfer magma-orchestrator:config .
ubuntu@host:~$ mv config ~/.kube/
```

Validate that you can run kubectl commands:

```bash
ubuntu@host:~$ kubectl get nodes
NAME                 STATUS   ROLES    AGE    VERSION
magma-orchestrator   Ready    <none>   104s   v1.25.2
```

## Deploying Magma's network management software

### Bootstrapping a Juju controller

Bootstrap a Juju controller on the Kubernetes instance we just created:

```bash
ubuntu@host:~$ juju add-k8s magma-orchestrator-k8s --client
ubuntu@host:~$ juju bootstrap magma-orchestrator-k8s
ubuntu@host:~$ juju add-model magma-orchestrator
```

### Deploying Magma Orchestrator

From your Ubuntu machine, create an `overlay.yaml` file that contains the following content:

```yaml
applications:
  orc8r-certifier:
    options:
      domain: awesome.com
  orc8r-nginx:
    options:
      domain: awesome.com
  tls-certificates-operator:
    options:
      generate-self-signed-certificates: true
      ca-common-name: rootca.awesome.com
```

Deploy Magma Orchestrator:

```bash
ubuntu@host:~$ juju deploy magma-orc8r --overlay overlay.yaml --trust --channel=edge
```

You can see the deployment's status by running

```bash
ubuntu@host:~$ juju status
```

The deployment is completed when all units are in the `Active-Idle` state:

```bash
ubuntu@host:~$ juju status
Model               Controller                          Cloud/Region                        Version  SLA          Timestamp
magma-orchestrator  magma-orchestrator-k8s-localhost  magma-orchestrator-k8s/localhost  2.9.35   unsupported  18:19:48-04:00

[...]

Unit                              Workload  Agent  Address      Ports     Message
nms-magmalte/0*                   active    idle   10.1.50.73             
nms-nginx-proxy/0*                active    idle   10.1.50.75             
orc8r-accessd/0*                  active    idle   10.1.50.76             
orc8r-alertmanager-configurer/0*  active    idle   10.1.50.81             
orc8r-alertmanager/0*             active    idle   10.1.50.77             
orc8r-analytics/0*                active    idle   10.1.50.82             
orc8r-bootstrapper/0*             active    idle   10.1.50.84             
orc8r-certifier/0*                active    idle   10.1.50.87             
orc8r-configurator/0*             active    idle   10.1.50.88             
orc8r-ctraced/0*                  active    idle   10.1.50.89             
orc8r-device/0*                   active    idle   10.1.50.90             
orc8r-directoryd/0*               active    idle   10.1.50.91             
orc8r-dispatcher/0*               active    idle   10.1.50.92             
orc8r-eventd/0*                   active    idle   10.1.50.94             
orc8r-ha/0*                       active    idle   10.1.50.95             
orc8r-lte/0*                      active    idle   10.1.50.97             
orc8r-metricsd/0*                 active    idle   10.1.50.99             
orc8r-nginx/0*                    active    idle   10.1.50.102            
orc8r-obsidian/0*                 active    idle   10.1.50.103            
orc8r-orchestrator/0*             active    idle   10.1.50.106            
orc8r-policydb/0*                 active    idle   10.1.50.107            
orc8r-prometheus-cache/0*         active    idle   10.1.50.110            
orc8r-prometheus-configurer/0*    active    idle   10.1.50.116            
orc8r-prometheus/0*               active    idle   10.1.50.72             
orc8r-service-registry/0*         active    idle   10.1.50.111            
orc8r-smsd/0*                     active    idle   10.1.50.112            
orc8r-state/0*                    active    idle   10.1.50.115            
orc8r-streamer/0*                 active    idle   10.1.50.117            
orc8r-subscriberdb-cache/0*       active    idle   10.1.50.119            
orc8r-subscriberdb/0*             active    idle   10.1.50.118            
orc8r-tenants/0*                  active    idle   10.1.50.120            
orc8r-user-grafana/0*             active    idle   10.1.50.123            
postgresql-k8s/0*                 active    idle   10.1.50.126  5432/TCP  Pod configured
tls-certificates-operator/0*      active    idle   10.1.50.121            
```

### Getting Access to Magma Orchestrator

First, retrieve the PFX package and password that contains the certificates to authenticate against 
Magma Orchestrator:

```bash
ubuntu@host:~$ juju scp --container="magma-orc8r-certifier" orc8r-certifier/0:/var/opt/magma/certs/admin_operator.pfx admin_operator.pfx
ubuntu@host:~$ juju run-action orc8r-certifier/leader get-pfx-package-password --wait
```

The pfx package was copied to your current working directory. If you are using Google Chrome, 
navigate to `chrome://settings/certificates?search=https`, click on Import, select 
the `admin_operator.pfx` package that we just copied and write in the password that you received.

> **SHOW PICTURE OF HOW THIS IS LOADED IN GOOGLE CHROME**

### Setupping DNS

Now, retrieve the list of services that need to be exposed:

```bash
ubuntu@host:~$ juju run-action orc8r-orchestrator/leader get-load-balancer-services --wait
```

In your domain registrar, create A records for the following Kubernetes services:

| Address                                | Hostname                                | 
|----------------------------------------|-----------------------------------------|
| `<orc8r-bootstrap-nginx External IP>`  | `bootstrapper-controller.<your domain>` | 
| `<orc8r-nginx-proxy External IP>`      | `api.<your domain>`                     | 
| `<orc8r-clientcert-nginx External IP>` | `controller.<your domain>`              | 
| `<nginx-proxy External IP>`            | `*.nms.<your domain>`                   | 

### Verify the deployment

Get the master organization's username and password:

```bash
juju run-action nms-magmalte/leader get-master-admin-credentials --wait
```

Confirm successful deployment by visiting `https://master.nms.<your domain>` and logging in
with the `admin-username` and `admin-password` outputted here.


## Deploying the 4G core

Bootstrap a second juju controller:

```bash
ubuntu@host:~$ juju bootstrap localhost localhost
```

Retrieve the IP address of the machine that will hold Magma's access gateway:

```bash
ubuntu@host:~$ multipass list
```

```bash
ubuntu@host:~$ juju add-machine
```

## Deploying a radio simulator

## Destroying the environment

First, destroy the 3 virtual machines that we created:

```bash
ubuntu@host:~$ multipass delete --all
```

Then, uninstall all the installed packages:

```bash
ubuntu@host:~$ sudo snap remove juju --purge
ubuntu@host:~$ sudo snap remove multipass --purge
ubuntu@host:~$ sudo snap remove kubectl --purge
```
