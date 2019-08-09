# ROKS (RedHat OpenShift on Kubernetes) ICD demo
Example Applications for frontend app backed by IBM Cloud Databases

##  Deploy a Multizone ROKS cluster in US-SOUTH
Visit the [OpenShift](https://cloud.ibm.com/kubernetes/clusters?platformType=openshift) page in the cloud portal to start provisioning your cluster.  
 - Click Create Cluster
 - Name your cluster
 - Pick Resource Group
 - Pick location (Geography and Region. Ex: North America, Dallas) 
 - Select VLANs
 - Select Machine size
 - Select Number of workers per zone
 - Click Create 

While the cluster is deploying, we'll go ahead and deploy a few ICD instances. In this case we'll be deploying an instance of Etcd and Elasticsearch. 

## Deploy ICD instances
 - Deploy [Etcd](https://cloud.ibm.com/catalog/services/databases-for-etcd) in the same region and Resource group as your ROKS cluster. For this example you can use the `lite` plan. If you have already provisioned a lite version of Etcd you will need to change `lite` to `standard`.

```
ibmcloud resource service-instance-create <Your-Etcd-Service-Name> databases-for-etcd lite us-south
```

 - Deploy [Elasticsearch](https://cloud.ibm.com/catalog/services/databases-for-elasticsearch) in the same region and Resource group as your ROKS cluster. For this example you can use the `lite` plan. If you have already provisioned a lite version of Elasticsearch you will need to change `lite` to `standard`.

```
ibmcloud resource service-instance-create <Your-Elasticsearch-Service-Name> databases-for-elasticsearch lite us-south
```

## Log in your your Openshift cluster
Hopefully your cluster has completed provisioning. Once it does log in to the Cluster GUI to grab your `oc` login command. It will look something like this:

```
oc login https://1.us-south.containers.cloud.ibm.com:XXXXX --token=<Generated Token>
```

You can also find your cluster master URL by running the command `ibmcloud ks cluster-get NAME_OF_CLUSTER`.

## Bind the ICD services to your OpenShift cluster
[Service binding](https://cloud.ibm.com/docs/containers?topic=containers-service-binding#bind-services) is a quick way to create service credentials for an IBM Cloud service by using its public service endpoint and storing these credentials in a Kubernetes/ROKS secret in your cluster.

```
ibmcloud ks cluster-service-bind --cluster CLUSTER-NAME --namespace default --service <Your-Etcd-Service-Name>

ibmcloud ks cluster-service-bind --cluster CLUSTER-NAME --namespace default --service <Your-Elasticsearch-Service-Name>
```

## Build and push Node.js applications
Clone the example code repository that will be used to build and deploy our node.js frontend applications. 

```
git clone https://github.com/greyhoundforty/roks-icd-demo.git
cd roks-icd-demo
```

### Build and deploy the Node.js Etcd container image
You will need to target an [IBM Cloud Container Registry](https://cloud.ibm.com/docs/services/Registry?topic=registry-getting-started#getting-started) namespace in order to build and push your container image. You can create a namepace with the `ibmcloud cr namespace-add` command and retrieve existing namespaces with `ibmcloud cr namespace-list`. 

```
cd etcd
ibmcloud cr login 
ibmcloud cr build -t <region>.icr.io/<namespace>/icdetcd .
```

Update the `etcd-deployment.yaml` file with your container image name and your Etcd ICD service-bind secret (lines 15 and 21 respectively). Once that is complete you can deploy your application to your ROKS cluster. 

```
oc create -f etcd-deployment.yaml
```

### Build and deploy the Node.js Elasticsearch container 

```
cd ../elasticsearch
ibmcloud cr build -t <region>.icr.io/<namespace>/icdes .
```

Update the `es-deployment.yaml` file with your container image name and your Elasticsearch ICD service-bind secret (lines 15 and 21 respectively). Once that is complete you can deploy your application to your ROKS cluster. 

```
oc create -f es-deployment.yaml
```

## Check status of deployments
If everything went according to plan your deployments and services have been created. You can run the `oc status` command to check on their progress.

```
oc status
In project default on server https://1.us-south.containers.cloud.ibm.com:XXXXX

svc/docker-registry - 172.21.63.116:5000
  deployment/docker-registry deploys registry.ng.bluemix.net/armada-master/iksorigin-ose-docker-registry:v3.11.129-1
    deployment #1 running for 4 hours - 2 pods

svc/node-es-service (all nodes):30091 -> 8080
  deployment/icdelasticsearch-app deploys us.icr.io/rtiffany/roks-south-icdes:1
    deployment #1 running for 3 hours - 3 pods

svc/node-etcd-service (all nodes):30524 -> 8080
  deployment/icdetcd-app deploys us.icr.io/rtiffany/rhos_icdetcd:1
    deployment #1 running for 3 hours - 3 pods

svc/router - 172.21.67.246 ports 80, 443
  deployment/router deploys registry.ng.bluemix.net/armada-master/iksorigin-ose-haproxy-router:v3.11.129-1
    deployment #2 running for 4 hours - 2 pods
    deployment #1 deployed 4 hours ago
```

## Expose our applications by creating a route
By default the services that are created are only routable from within the cluster itself. An OpenShift route exposes a service at a host name, like www.example.com, so that external clients can reach it by name. If you do not specify a route one will be generated for you in the form of `<service-name>.<namespace>.<cluster-name>.<random-string>.<ingress-subdomain>`.

```
oc expose svc/node-es-service
oc expose svc/node-etcd-service
```

## Visit your applications
Use the `get routes` command to retrieve the public URL for your deployed applications.

```
oc get routes/node-es-service
oc get routes/node-etcd-service
```
