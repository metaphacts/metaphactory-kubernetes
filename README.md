# metaphactory deployments with kubernetes

**Prerequisites:**

* kubectl installed and kubernetes cluster setup (client and server version >= 1.9 , check with `kubectl version`)
* outgoing HTTP/HTTPS traffic for the kubernetes cluster, allowing to access external docker registries (e.g. docker hub)

## metaphactory Deployment and Maintenance

**Prerequisites:**

To perform any deployments or updates, you will first need to login to the metaphact's docker hub repositroy with your docker hub account, i.e. run `docker login`.
Please request your account to be added via **support@metaphacts.com** if you have not yet done so.

**IMPORTANT:** 
It is also possible to use `stateful sets` for the deployment of metaphactory. An example configuration can be found in `/metaphactory/statefulset/`, which includes the pod and volume claim definitions in one file and allows easy update and scale-out configurations. Below guides will use the simple pod definition instead.

The configuration for AWS Loadbalancer with SSL termination is included in `/metaphactory/metaphactory-service.yaml`, feel free to remove `#` commenting to utilize that configuration when running metaphactory on AWS EKS.

### Initial Deployment
To create a new deployment from scratch chose from these two options:

#### Metaphactory with blazegraph triplestore included (recommended for initial tests)

1. Clone this GIT repository with `git clone https://github.com/metaphacts/metaphactory-kubernetes.git`
2. Create a secret in your cluster to pull images from a private registry and note down the name of that secret (default name assumed in provided configuration files is `regcred`): 
  `kubectl create secret docker-registry regcred --docker-username=metaphactscustomers --docker-password=<your-token-from-registration>`
2. Change into the `/metaphactory-kubernetes` folder 
3. Ensure that the correct name for the secret name is set in `/metaphactory/pod/metaphactory-pod.yaml`for `imagePullSecrets: - name: `
4. Remove `#` characters in `/metaphactory/pod/metaphactory-pod.yaml` for `- name: BLAZEGRAPH_ENDPOINT`, `value: -Dconfig.environment.sparqlEndpoint=http:// ...` and `$(BLAZEGRAPH_ENDPOINT)`
5. Ensure that the intended storage class is set in `/metaphactory/pod/metaphactory_runtime_data-persistentvolumeclaim.yaml` and `/metaphactory-blazegraph/metaphactory_blazegraph_data-persistentvolumeclaim.yaml` for `storageClassName: `. The setting is commented out in the provided configuration, which will default to the `default` storage class
6. The configuration assumes the usage of a loadbalancer.  Please change `type: LoadBalancer` to `type: NodePort` in `/metaphactory/metaphactory-service.yaml`, if you do not want to use a loadbalancer.
7. The blazegraph image is configured with a 1GB (1Gi) volume, which should be sufficient for 10M triples. Please increase the size of the volume in `/metaphactory-blazegraph/metaphactory_blazegraph_data-persistentvolumeclaim.yaml` if you plan to store more than 10M triples.
8. First start the blazegraph pod and service with `kubectl apply -f ./metaphactory-blazegraph` (on Windows run `kubectl apply -f .\metaphactory-blazegraph`)
9. Verify that the pod is up and running with `kubectl get pod metaphactory-blazegraph` where `READY` should be `1/1` and `STATUS` shows as `Running`
10. Next start the metaphacrory service with `kubectl apply -f ./metaphactory` (on Windows run `kubectl apply -f .\metaphactory`)
11. Verify that the service for `metaphactory` and `metaphactory-blazegraph` is up and running with `kubectl get services`. `metaphactory` service should show an external IP, please note down this IP or hostname
12. Finally start the metaphactory pod and persistent volume claim with `kubectl apply -f ./metaphactory/pod` (on Windows run `kubectl apply -f .\metaphactory\pod`)
13. Verify that the service is running by connecting to `http://<external IP>` with the external IP as retrieved during step 11. 
14. Login with credentials `admin` and password `admin`

To remove the complete setup run following commands (**Note: This will remove all persistent volumes and data as well**)

1. Ensure you are in the folder for `/metaphactory-kubernets`
2. First remove the public service with `kubectl delete -f ./metaphactory` (on Windows run `kubectl delete -f .\metaphactory`)
3. Next remove the metaphactory pod and volume with `kubectl delete -f ./metaphactory/pod` (on Windows run `kubectl delete -f .\metaphactory\pod`) - **Note: This will destroy all metaphactory runtime data and configuration**
4. Finally remove the metaphactory-blazegraph service, pod and volume with `kubectl delete -f ./metaphactory-blazegraph` (on Windows run `kubectl delete -f .\metaphactory-blazegraph`) **Note: This will destroy all data stored in metaphactory-blazegraph database**

#### Metaphactory for use with existing triplestores

1. Clone this GIT repository with `git clone https://github.com/metaphacts/metaphactory-kubernetes.git`
2. Create a secret in your cluster to pull images from a private registry and note down the name of that secret (default name assumed in provided configuration files is `regcred`): 
  `kubectl create secret docker-registry regcred --docker-username=metaphactscustomers --docker-password=<your-token-from-registration>`
2. Change into the `/metaphactory-kubernetes` folder 
3. Ensure that the correct name for the secret name is set in `/metaphactory/pod/metaphactory-pod.yaml` for `imagePullSecrets: - name: `
4. Ensure that the intended storage class is set in `/metaphactory/pod/metaphactory_runtime_data-persistentvolumeclaim.yaml` for `storageClassName: `. The setting is commented out in the provided configuration, which will default to the `default` storage class
5. The configuration assumes the usage of a loadbalancer.  Please change `type: LoadBalancer` to `type: NodePort` in `/metaphactory/metaphactory-service.yaml`, if you do not want to use a loadbalancer.
6. Next start the metaphacrory service with `kubectl apply -f ./metaphactory` (on Windows run `kubectl apply -f .\metaphactory`)
7. Verify that the service for `metaphactory` is up and running with `kubectl get service metaphactory`. `metaphactory` service should show an external IP, please note down this IP or hostname
8. Finally start the metaphactory pod and persistent volume claim with `kubectl apply -f ./metaphactory/pod` (on Windows run `kubectl apply -f .\metaphactory\pod`)
9. Verify that the service is running by connecting to `http://<external IP>` with the external IP as retrieved during step 7.
10. Login with credentials `admin` and password `admin`

**Please note:** Once the platform started and on initial login, you will be asked to configure the connection to your default repository via the [Repository Manager](https://help.metaphacts.com/resource/Help:RepositoryManager). 
See also: [How to connect to Stardog](https://help.metaphacts.com/resource/Help:HowToConnectToStardog)

To remove the complete setup run following commands (**Note: This will remove all persistent volumes and data as well**)

1. Ensure you are in the folder for `/metaphactory-kubernets`
2. First remove the public service with `kubectl delete -f ./metaphactory` (on Windows run `kubectl delete -f .\metaphactory`)
3. Next remove the metaphactory pod and volume with `kubectl delete -f ./metaphactory/pod` (on Windows run `kubectl delete -f .\metaphactory\pod`) - **Note: This will destroy all metaphactory runtime data and configuration**

#### Troubleshooting

Please follow below steps before contacting our support on `support@metaphacts.com`

1. Run `kubectl describe service metaphactory` to verify the status of the service.
2. Run `kubectl descirbe pod metaphactory` to get details on the metaphactory pod status. The init container should be in state terminated and the actual container in state running.
3. Pull logs on the metaphactory pod with `kubectl logs metaphactory`
4. You can access the container for advanced troubleshooting through `kubectl exec -it metaphactory -- sh`
5. If using `statefulsets`, please note that the persisten volumes might not be deleted with the statefulset. Please use `kubectl delete pvc -l app=metaphactory` to delete the persistent volumes **Note: This will remove all persistent volumes and data as well**
6. Additional environment parameters for metaphactory can be supplied in a configmap with `name: metaphactory` and `key: extra_env`. This can for example be used to specify `-Dlog=log4j2-trace2` for trace logging. Please make sure that `$(EXTRA_ENV)` in `/metaphactory/pod/metaphactory-pod.yaml` is no longer commented out for this to work.
