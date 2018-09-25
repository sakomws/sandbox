# Deploying Flask applications using kube-downscaler
In this tutorial we will show how to deploy Kube-downscaler and test with sample application.

## What is Kube-downscaler?
Please check  [kube-downscaler](https://github.com/hjacobs/kube-downscaler) for a detailed explanation.  

## Production Status
The current version of downscaler is 0.5.

## Architecture
The diagram below depicts how a Downscaler agent control applications.
![Alt text](/images/architecture.png?raw=true "Kube Downscaler diagram")

## Quick Start
Below are instructions to quickly install and configure Downscaler.  

### Prerequisites
Make sure you have a Kubernetes cluster up and running. For local installation [minikube](https://github.com/kubernetes/minikube) can be used as well. 

Install the binary releases of the Helm and Kubectl clients depending on your OS.  Please follow [Helm client](https://docs.helm.sh/using_helm/#quickstart) and [Kubectl client](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for detailed instructions.


### Installing Downscaler

1. Make sure connected to right cluster:
```
kubectl config current-context
```

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

4. Clone Sandbox repository:
```
git clone https://github.com/sakomws/sandbox.git |  cd sandbox
```

5. Before deploy make sure to update [values.yaml](https://github.com/sakomws/sandbox/blob/master/downscaler/values.yaml) in Downscaler chart depending on your cluster support for RBAC:
```
rbac:
  create: true
```

6. Deploy Downscaler:
```
helm install ./downscaler
```
7. Check the deployed release status:
```
helm list
```
```
NAME            	REVISION	UPDATED                 	STATUS  	CHART                	APP VERSION	NAMESPACE
bold-guppy      	1       	Tue Sep 25 02:07:58 2018	DEPLOYED	kube-downscaler-0.5.0	0.5.0      	default
```

### Deploy Sample Applications
In this section we will deploy the Flask applications. Please see [tutorial](https://github.com/OpenGov/opendata-ops/tree/master/docs/tutorial/tutorial/k8s/flask) in opendata-ops repository under OpenGov organization for more details.

1. Deploy Flask applications:
```
kubectl apply -f https://github.com/sakomws/sandbox/blob/master/flaskapp/flask_1.yaml?raw=true
kubectl apply -f https://github.com/sakomws/sandbox/blob/master/flaskapp/flask_2.yaml?raw=true
```

2. Ensure the following Kubernetes pods are up and running: flask-v1-tutorial-* , flask-v2-tutorial-* :
```
kubectl get pods -n default  
```
```
NAME                                 READY     STATUS    RESTARTS   AGE
flask-v1-tutorial-6b59556b55-kd2tv   1/1       Running   0          1m
flask-v2-tutorial-575fd64689-rkf55   1/1       Running   0          1m
```

Note: Deployments have grace period, which means Downscaler will wait 15min to take any actions after pods get started. 

3. Check downscaler pod logs:
```
kubectl logs -f kube-downscaler-55b9f8ffd8-5k9q4  
```
```
2018-09-25 18:13:56,253 INFO: Deployment default/flask-v1-tutorial within grace period (900s), not scaling down (yet)
2018-09-25 18:13:56,253 INFO: Deployment default/flask-v2-tutorial within grace period (900s), not scaling down (yet)
2018-09-25 18:14:01,310 INFO: Scaling down Deployment default/flask-v1-tutorial from 1 to 0 replicas (uptime: Mon-FRI 07:00-19:00 US/Eastern, downtime: never)
2018-09-25 18:14:01,327 INFO: Scaling down Deployment default/flask-v2-tutorial from 1 to 0 replicas (uptime: Thu-Fri 07:00-19:00 US/Pacific, downtime: never)
```

### Uninstalling Sample Applications
1. To uninstall applications, run:
```
kubectl delete -f https://github.com/sakomws/sandbox/blob/master/flaskapp/flask_1.yaml?raw=true
kubectl delete -f https://github.com/sakomws/sandbox/blob/master/flaskapp/flask_2.yaml?raw=true
```


### Uninstalling Downscaler
1. Check Downscaler release name:
```
helm  list
```
```
NAME            	REVISION	UPDATED                 	STATUS  	CHART                	APP VERSION	NAMESPACE
bold-guppy      	1       	Tue Sep 25 02:07:58 2018	DEPLOYED	kube-downscaler-0.5.0	0.5.0      	default
```

2. Delete Downscaler release from cluster:
```
helm delete bold-guppy
```

3. Delete Tiller deployment:
```
kubectl delete deployment tiller-deploy --namespace kube-system
```

## Limitations
Still in testing phase, will be updated later.

## Authors

[Opengov](https://opengov.com) Devops team.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

* Thanks to [Kube-downscaler](https://github.com/hjacobs/kube-downscaler) project authored by [Henning Jacobs](https://github.com/hjacobs)
