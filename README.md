# Deploying Flask applications using kube-downscaler
In this tutorial we will show how to deploy Kube-downscaler and test with sample
[flask application](https://github.com/OpenGov/opendata-ops/tree/master/docs/tutorial/tutorial/k8s/flask).

## What is Kube-downscaler?
Please check  [kube-downscaler](https://github.com/hjacobs/kube-downscaler) for a detailed explanation.  

## Production Status
The current version of downscaler is 0.5.

## Architecture
The diagram below depicts how an Downscaler agent control applications.
![Alt text](/images/architecture.png?raw=true "Kube Downscaler diagram")

## Quick Start
Below are instructions to quickly install and configure Downscaler.  

### Prerequisites
Make sure you have a Kubernetes cluster up and running. 
For local installation [minikube](https://github.com/kubernetes/minikube) can be used as well. 
Make 
We are going to deploy application using [Helm chart](https://docs.helm.sh) and [Kubectl client](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Installing Downscaler

1. Make sure in using right cluster:
```
kubectl config current-context
```

2. Deploy Helm into cluster:
```
helm init
```

3. Clone Sandbox repository:
```
git clone https://github.com/sakomws/sandbox.git |  cd sandbox
```

4. Deploy Downscaler:
```
helm install ./downscaler
```

5. Deploy Flask applications:
```
kubectl apply -f flaskapp/flask_1.yaml,flaskapp/flask_2.yaml
```

5. Ensure the following Kubernetes pods are up and running: kube-downscaler-* , flask-v1-tutorial-* , flask-v1-tutorial-*
```
kubectl get pods -n istio-system    
```
```
NAME                                 READY     STATUS    RESTARTS   AGE
flask-v1-tutorial-6b59556b55-kd2tv   1/1       Running   0          1m
flask-v2-tutorial-575fd64689-rkf55   1/1       Running   0          1m
kube-downscaler-55b9f8ffd8-5k9q4     1/1       Running   0          8h
```

6. Automatic sidecar:
To set up sidecar injection, please run following script which will install Istio webhook with nginMesh customization.
```
nginmesh-0.7.2/install/kubernetes/install-sidecar.sh
```

7. Verify that istio-injection label is not labeled for the default namespace :
```
kubectl get namespace -L istio-injection
```
```
NAME           STATUS        AGE       ISTIO-INJECTION
default        Active        1h        
istio-system   Active        1h        
kube-public    Active        1h        
kube-system    Active        1h
```

### Kafka deployment using Helm

1. Install the binary release of the Helm client depending on your OS.  Please follow [Setup guide](https://docs.helm.sh/using_helm/#quickstart) for detailed instructions.

2. Install Helm server(Tiller) in Kubernetes cluster:
```
helm init
```
3. Make sure Tiller is up and running:

```
kubectl get pods --namespace kube-system
```
```
NAME                                                 READY     STATUS    RESTARTS   AGE
tiller-deploy-f44659b6c-p48hf                        1/1       Running   0          51m
```

4. Run the following script to setup Kafka. It will be installed in 'kafka' namespace.  It is also possible to use existing kafka installation.

```
nginmesh-0.7.2/install/kafka/install.sh
```
Note: In GKE environment you may need to grant permission to default serviceaccount for cluster-wide access before install:

```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller","automountServiceAccountToken": true}}}}'
```

5. Wait for a while and make sure all kafka and zookeeper pods are up and runnning:

```
kubectl get pods -n kafka
```
```
NAME                   READY     STATUS    RESTARTS   AGE
my-kafka-kafka-0       1/1       Running   1          15m
my-kafka-kafka-1       1/1       Running   0          13m
my-kafka-kafka-2       1/1       Running   0          12m
my-kafka-zookeeper-0   1/1       Running   0          15m
my-kafka-zookeeper-1   1/1       Running   0          14m
my-kafka-zookeeper-2   1/1       Running   0          14m
testclient             1/1       Running   0          15m
```
6. Check the deployed release status:

```
helm list
```
```
NAME    	         REVISION	         UPDATED                       	STATUS  	    CHART      	  NAMESPACE
my-kafka	           1           Tue Mar 27 18:45:18 2018	          DEPLOYED  	 kafka-0.4.7  	   kafka
```

7. Set up  topic named "nginmesh" by running below script:

```
nginmesh-0.7.2/tools/kafka-add-topics.sh nginmesh
```
8. View created topic by running below script:

```
nginmesh-0.7.2/tools/kafka-list-topics.sh
```
```
nginmesh
```

### Deploy a Sample Application
In this section we deploy the Bookinfo application, which is taken from the Istio samples. Please see [Bookinfo](https://istio.io/docs/guides/bookinfo.html)  for more details.

1. Label the default namespace with istio-injection=enabled:

```
kubectl label namespace default istio-injection=enabled
```

2. Deploy the application:

```
kubectl apply -f  nginmesh-0.7.2/samples/bookinfo/kube/bookinfo.yaml
```

3. Confirm that all application services are deployed: productpage, details, reviews, ratings:

```
kubectl get services
```
```
NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)              AGE
details                    10.0.0.31    <none>        9080/TCP             6m
kubernetes                 10.0.0.1     <none>        443/TCP              7d
productpage                10.0.0.120   <none>        9080/TCP             6m
ratings                    10.0.0.15    <none>        9080/TCP             6m
reviews                    10.0.0.170   <none>        9080/TCP             6m
```

4. Confirm that all application pods are running --details-v1-* , productpage-v1-* , ratings-v1-* , reviews-v1-* , reviews-v2-* and reviews-v3-* :
```
kubectl get pods
```
```
NAME                                        READY     STATUS    RESTARTS   AGE
details-v1-1520924117-48z17                 2/2       Running   0          6m
productpage-v1-560495357-jk1lz              2/2       Running   0          6m
ratings-v1-734492171-rnr5l                  2/2       Running   0          6m
reviews-v1-874083890-f0qf0                  2/2       Running   0          6m
reviews-v2-1343845940-b34q5                 2/2       Running   0          6m
reviews-v3-1813607990-8ch52                 2/2       Running   0          6m
```

5. Get the public IP of the Istio Ingress controller. If the cluster is running in an environment that supports external load balancers:

```
kubectl get svc -n istio-system | grep -E 'EXTERNAL-IP|istio-ingress'
```

OR

```
kubectl get ingress -o wide       
```

6. Open the Bookinfo application in a browser using the following link:
```
http://<Public-IP-of-the-Ingress-Controller>/productpage
```

Note: For E2E routing rules and performace testing you could refer to [E2E Test](istio/tests/README.md).

### Demo nginMesh streaming using Graylog
1. [Demo Graylog](istio/release/demo/graylog/README.md) Please, refer for Graylog integration with nginMesh.
2. [Demo KSQL](istio/release/demo/ksql/README.md) Please, refer for KSQL integration with nginMesh.

### Uninstalling the Application
1. To uninstall application, run:

```
./nginmesh-0.7.2/samples/bookinfo/kube/cleanup.sh
```


### Uninstalling Istio
1. To uninstall the Istio core components:

```
kubectl delete -f istio-0.7.1/install/kubernetes/istio.yaml
```


2. To uninstall the initializer, run:

```
nginmesh-0.7.2/install/kubernetes/delete-sidecar.sh
```

### Uninstalling Kafka

1. Uninstall Kafka:

```
nginmesh-0.7.2/install/kafka/uninstall.sh
``` 

2. Delete Tiller deployment:

```
kubectl delete deployment tiller-deploy --namespace kube-system
```

 


## Limitations
nginMesh has the following limitations:
* TCP and gRPC traffic is not supported.
* Quota Check is not supported.
* Only Kubernetes is supported.

All sidecar-related limitations and supported traffic management rules are described [here](istio/agent).
