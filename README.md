# metaphactory deployments with kubernetes

**Prerequisites:**

* kubectl installed and Kubernetes cluster setup (client and server version >= 1.9 , check with `kubectl version`)
* outgoing HTTP/HTTPS traffic for the Kubernetes cluster, allowing to access external docker registries (e.g. docker hub)

## metaphactory Deployment and Maintenance

**Prerequisites:**

To perform any deployments or updates, you will first need to log into metaphact's Docker hub repository. If you do not yet have access, please register for a trial on [https://metaphacts.com/get-started](https://metaphacts.com/get-started) and follow the steps to get started with a Docker-based deployment. 
After registration you will receive an email containing a user name and token to access the image from Docker Hub which can be used to log in using `docker login`. For use with Kubernetes the Docker Hub credentials need to be configured as image pull credentials. See below for details.

**IMPORTANT:** 
It is also possible to use `StatefulSets` for the deployment of metaphactory. An example configuration can be found in `metaphactory/statefulset/`, which includes the pod and volume claim definitions in one file and allows easy update and scale-out configurations. Below guides will use the simple pod definition instead.

The configuration for the AWS LoadBalancer (ALB) with SSL termination is included in `metaphactory/metaphactory-service.yaml`, feel free to remove `#` commenting to utilize that configuration when running metaphactory on AWS EKS.

### Initial Deployment
To create a new deployment from scratch follow these steps:

1. Clone this GIT repository with `git clone https://github.com/metaphacts/metaphactory-kubernetes.git`
2. Create a secret in your cluster to pull images from a private registry and note down the name of that secret (default name assumed in provided configuration files is `regcred`): 
  `kubectl create secret docker-registry regcred --docker-username=metaphactscustomers --docker-password=<your-token-from-registration>`
2. Change into the `metaphactory-kubernetes` folder 
3. Ensure that the correct name for the secret name is set in `metaphactory/pod/metaphactory-pod.yaml` for `imagePullSecrets: - name: `
4. Ensure that the intended storage class is set in `metaphactory/pod/metaphactory_runtime_data-persistentvolumeclaim.yaml` for `storageClassName: `. The setting is commented out in the provided configuration, which will default to the `default` storage class
5. The configuration assumes the usage of a load balancer.  Please change `type: LoadBalancer` to `type: NodePort` in `metaphactory/metaphactory-service.yaml`, if you do not want to use a load balancer.
6. Next start the metaphactory service with `kubectl apply -f ./metaphactory` (on Windows run `kubectl apply -f .\metaphactory`)
7. Verify that the service for `metaphactory` is up and running with `kubectl get service metaphactory`. `metaphactory` service should show an external IP, please note down this IP or hostname
8. Finally start the metaphactory pod and persistent volume claim with `kubectl apply -f ./metaphactory/pod` (on Windows run `kubectl apply -f .\metaphactory\pod`)
9. Verify that the service is running by connecting to `http://<external IP>` with the external IP as retrieved during step 7.
10. Login with credentials `admin` and password `admin`

**Please note:** Once the platform started and on initial login, you will be asked to configure the connection to your default repository via the [Repository Manager](https://help.metaphacts.com/resource/Help:RepositoryManager). 
See also: [How to connect to Stardog](https://help.metaphacts.com/resource/Help:HowToConnectToStardog)

To remove the complete setup run following commands (**Note: This will remove all persistent volumes and data as well**)

1. Ensure you are in the folder for `metaphactory-kubernetes`
2. First remove the public service with `kubectl delete -f ./metaphactory` (on Windows run `kubectl delete -f .\metaphactory`)
3. Next remove the metaphactory pod and volume with `kubectl delete -f ./metaphactory/pod` (on Windows run `kubectl delete -f .\metaphactory\pod`) - **Note: This will destroy all metaphactory runtime data and configuration**

#### Troubleshooting

Please follow below steps before contacting our support on `support@metaphacts.com`

1. Run `kubectl describe service metaphactory` to verify the status of the service.
2. Run `kubectl descirbe pod metaphactory` to get details on the metaphactory pod status. The init container should be in state terminated and the actual container in state running.
3. Pull logs on the metaphactory pod with `kubectl logs metaphactory`
4. You can access the container for advanced troubleshooting through `kubectl exec -it metaphactory -- sh`
5. When using `statefulsets`, please note that the persisted volumes might not be deleted with the StatefulSet. Please use `kubectl delete pvc -l app=metaphactory` to delete the persistent volumes **Note: This will remove all persistent volumes and data as well**
6. Additional environment parameters for metaphactory can be supplied in a ConfigMap with `name: metaphactory` and `key: extra_env`. This can for example be used to specify `-Dlog=log4j2-trace2` for trace logging. Please make sure that `$(EXTRA_ENV)` in `metaphactory/pod/metaphactory-pod.yaml` is no longer commented out for this to work.
