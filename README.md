# IOTPAAS Installation

## Overview

This is a howto to install the iot-paas ecosystem

Its covers the installation of :

- 6 replica redis ha cluster (statefulset) with persistent storage
- 3 replica kafka/zookeeper ha cluster (statefulset) with persistent storage using strimzi operator
- iotpaas-message-producer
- iotpaas-message-consumer (for redis)
- iotpaas-auth-service
- iotpaas-nginx-api gateway service (and configmap)
- iotpass-infra-pg (grafana & prometheus)

The installation of couchbase is not covered, this should be fairly straightforward using the couchabse operator

There is a iotpaas-message-consumer for couchbase that must be used with couchbase, the installation is left up to the user.


## Installation

Clone the iotpaas-templates repo and 'cd' into the repo


### Requirements

- kubectl and kubernetes cluster (left up to reader to install and setup)
- kustomize [https://kustomize.io/]

### Redis HA Cluster

Execute the following commands 

```
$ kustomize build manifests/apps/redis/base | kubectl apply -f-
$ # monitor the deployment
$ kubectl get all -n redic-cluster
$ # once deployed execute the following
$ IPS=$(kubectl -n redis-cluster get pods -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
$ kubectl exec -it redis-cluster-0 -- /bin/sh
$ redis-cli --cluster create --cluster-replicas 1 $(IPS)
  
$ # result should be in the form

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.0.32.194:6379 to 10.0.58.133:6379
Adding replica 10.0.133.5:6379 to 10.0.58.134:6379
Adding replica 10.0.58.135:6379 to 10.0.133.4:6379
M: 9d0ceae9a8f6bca18edfa54aa9b22b08fe774fb0 10.0.58.133:6379
   slots:[0-5460] (5461 slots) master
M: 64cc7a4f26c7801d6fd6c3636e6888d1b3890363 10.0.58.134:6379
   slots:[5461-10922] (5462 slots) master
M: 02a6c53473abdc9ffd0e44164a38ed47e1820a1c 10.0.133.4:6379
   slots:[10923-16383] (5461 slots) master
S: ea89a4f9bf15b2cb5d9780d3853da39e81ced407 10.0.32.194:6379
   replicates 9d0ceae9a8f6bca18edfa54aa9b22b08fe774fb0
S: 1873b9e1501fce0863eba068625a83bf06fb1916 10.0.133.5:6379
   replicates 64cc7a4f26c7801d6fd6c3636e6888d1b3890363
S: 76e2f0a80b9ffd41a96e73f9f50c2214a9e1715b 10.0.58.135:6379
   replicates 02a6c53473abdc9ffd0e44164a38ed47e1820a1c
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 10.0.58.133:6379)
M: 9d0ceae9a8f6bca18edfa54aa9b22b08fe774fb0 10.0.58.133:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 64cc7a4f26c7801d6fd6c3636e6888d1b3890363 10.0.58.134:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 76e2f0a80b9ffd41a96e73f9f50c2214a9e1715b 10.0.58.135:6379
   slots: (0 slots) slave
   replicates 02a6c53473abdc9ffd0e44164a38ed47e1820a1c
M: 02a6c53473abdc9ffd0e44164a38ed47e1820a1c 10.0.133.4:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: ea89a4f9bf15b2cb5d9780d3853da39e81ced407 10.0.32.194:6379
   slots: (0 slots) slave
   replicates 9d0ceae9a8f6bca18edfa54aa9b22b08fe774fb0
S: 1873b9e1501fce0863eba068625a83bf06fb1916 10.0.133.5:6379
   slots: (0 slots) slave
   replicates 64cc7a4f26c7801d6fd6c3636e6888d1b3890363
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
```


### Kafka HA cluster

Install the operator and custom resource definitions
```
$ kubectl create namespace kafka
$ kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
$ # monitor the kafka operator
$ kubectl get all -n kafka
$ # once the opertaor is up and running
$ kustomize build manifests/apps/kafka-ha/base | kubectl apply -f-
$ # check installation
$ kubectl get all -n kafka
```

### Install IOTPAAS application

```
$ kustomize build manifests/apps/iotpaas-message-producer/base | kubectl apply -f-
$ kustomize build manifests/apps/iotpaas-message-consumer/base | kubectl apply -f-
```

Its left up to the user to configure and deploy the other services
- iotpaas-reverse-proxy
- iotpaas-auth-service



### Install IOTPAAS Observability

The observability install is in this repo [https://github.com/luigizuccarelli/iotpaas-infra-pg]
Clone the repo and cd into the repo and execute the following

```
$ kustomize build environmenst/overlays/prod | kubectl apply -f-
$ # once installed open the grafana browser (it's up to the user to enable an ingress for the grafana service)
$ # navigate to import dashboard
$ # load the file 'IOTPaaS [Whitebox Appplication Metrics]-1628494542444.json' found in the repo directory
```


### Install IOTPAAS cicd pipeline

This assumes that 
- Tekton has been installed
- Gitea is installed and setup (webhook)
- The golang-gitwebhook-service is deployed [https://github.com/luigizuccarelli/golang-gitwebhook-service]
- The golang cicd repo gitea is installed and setup with the repos to build your golang projects

The golang cicd pipeline repo is [https://github.com/luigizuccarelli/infra-golang-cicd]

Clone and 'cd' into the repo then execute the following

The eventListener uses the schema as defined in gitea repo

```
$ kustomize build environments/overlays/cicd | kubectl apply -f-
```


