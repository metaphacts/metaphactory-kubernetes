# metaphactory Deployment on Kubernetes


## metaphactory Deployment and Maintenance

### Prerequisites

* kubectl installed and Kubernetes cluster setup (client and server version >= 1.21 , check with `kubectl version`)
* outgoing HTTPS traffic for the Kubernetes cluster allowing access to Docker Hub (unless metaphactory is pulled from an internal Docker registry)
* login credentials for metaphactory Docker Hub repository

### Overview
The provided configuration deploys metaphactory as a `StatefulSet` and includes the metaphactory pod, corresponding persistent volume claim definitions as well as a service definition.

For running this in an AWS-based deployment the configuration for the AWS LoadBalancer (ALB) with SSL termination is included in `metaphactory/metaphactory-service.yaml`. Simply remove the `#` comments to utilize that configuration when running metaphactory on AWS EKS.

The `ConfigMap` defined in `metaphactory/metaphactory-statefulset.yaml` specifies important configuration values such as using SSO (the example uses OIDC with Azure AD) and a database configuration for an externally running GraphDB database. See section `Configuration` below for instructions.

#### Pulling the metaphactory image from Docker Hub
To perform deployments or updates, log into metaphact's Docker hub repository. If you do not yet have access, please register for a trial on [https://metaphacts.com/get-started](https://metaphacts.com/get-started) and follow the steps to get started with a Docker-based deployment. 
After registration you will receive an email containing a user name and token to access the image from Docker Hub which can be used to log in using `docker login`. For use with Kubernetes the Docker Hub credentials need to be configured as image pull credentials (see below for details). Alternatively, the metaphactory container image can be imported into a local container registry and used without external references.

### Initial Deployment
To create a new deployment from scratch follow these steps:

1. Clone this GIT repository with `git clone https://github.com/metaphacts/metaphactory-kubernetes.git`
2. When pulling images from Docker Hub, create a secret in your cluster for storing the registry credentials and note down the name of that secret (default name assumed in provided configuration files is `regcred`): 
  `kubectl create secret docker-registry regcred --docker-username=metaphactscustomers --docker-password=<your-token-from-registration>`
2. Change into the `metaphactory-kubernetes` folder 
3. Ensure that the name of the secret created above is set in `metaphactory/metaphactory-statefulset.yaml` for `imagePullSecrets: - name: `
4. Without further configuration any persistent volumes will be created using the `default` storage class. In order to use a custom storage class instead, adjust the corresponding setting `storageClassName:` in `metaphactory/metaphactory-statefulset.yaml` and remove the comment character `#` from the start of the line to activate this setting.
5. The configuration assumes the usage of a load balancer.  Please change `type: LoadBalancer` to `type: NodePort` in `metaphactory/metaphactory-service.yaml`, if you do not want to use a load balancer.
6. Adjust the SSO and database configuration in `metaphactory/metaphactory-statefulset.yaml` to work with the target environment (see below)
7. Next start the metaphactory service with `kubectl apply -f ./metaphactory/metaphactory-service.yaml` (on Windows run `kubectl apply -f .\metaphactory\metaphactory-service.yaml`)
8. Verify that the service for `metaphactory` is up and running with `kubectl get service metaphactory`. `metaphactory` service should show an external IP, please note down this IP or hostname
9. Finally start the metaphactory pod and persistent volume claim with `kubectl apply -f ./metaphactory/metaphactory-statefulset.yaml` (on Windows run `kubectl apply -f .\metaphactory\metaphactory-statefulset.yaml`)
10. Verify that the service is running by connecting to `http://<external IP>` with the external IP as retrieved during step 8.
11. Login with your SSO user. When local users are enabled (see `Configuration` below) user name and credentials can be provided in the login form available at the `/login` endpoint. The default credentials are user `admin` with password `admin`.


### Configuration

The `ConfigMap` defined in `metaphactory/metaphactory-statefulset.yaml` provides an example configuration using OIDC with Azure AD for Single-Sign On (SSO) and a database configuration for an externally running GraphDB database.

#### Authentication and Single-Sign On (SSO)

The OIDC configuration is defined in `shiro-sso-oidc-params.ini`, the example shows how to use OIDC with Azure AD for Single-Sign On (SSO). When using Azure, the values for `discoveryURI.value` (replace `customer-tenant-id` with your organization's ID), `callbackUrl.value` (externally reachable URL of your metaphactory installation), `clientId.value` (ID of application registered for metaphactory in Azure AD), and `clientSecret.value` (corresponding client password) need to be adjusted.

Local users can be enabled by uncommenting the parameter `enableLocalUsers` in `environment.prop`. The users are defined in the `shiro.ini` file also provided in the config map.

**Please note:** as this file is projected into the container as a read-only file, users and passwords cannot be managed using metaphactory's User Administration page. Instead, they can be created using the [Shiro Command Line Hasher
](https://shiro.apache.org/command-line-hasher.html) tool and stored in the ConfigMap.

#### Database configuration

The database configuration is provided with the `repository-config` key in the `ConfigMap`. See [Repository Manager](https://help.metaphacts.com/resource/Help:RepositoryManager) as well as [How to connect to GraphDB](https://help.metaphacts.com/resource/Help:HowToConnectToGraphDB) or [How to connect to Stardog](https://help.metaphacts.com/resource/Help:HowToConnectToStardog) for details on database configuration.

**Please note:** as this file is projected into the container as a read-only file, it cannot be changed using metaphactory's Repository Administration page.

### Deleting the deployment

To remove the complete setup run following commands (**Note: This will remove all persistent volumes and data as well**)

1. Ensure you are in the folder for `metaphactory-kubernetes`
2. First remove the public service with `kubectl delete -f ./metaphactory/metaphactory-service.yaml` (on Windows run `kubectl delete -f .\metaphactory\metaphactory-service.yaml`)
3. Next remove the metaphactory pod and volume with `kubectl delete -f ./metaphactory/metaphactory-statefulset.yaml` (on Windows run `kubectl delete -f .\metaphactory\metaphactory-statefulset.yaml`) - **Note: This will destroy the volumes containing the metaphactory runtime data and configuration!**

## Troubleshooting

Please follow below steps before contacting our support on `support@metaphacts.com`

1. Run `kubectl describe service metaphactory` to verify the status of the service.
2. Run `kubectl describe pod metaphactory` to get details on the metaphactory pod status. The init container should be in state `terminated` and the actual container in state `running`.
3. Run `kubectl get all,pv,pvc,secret` to get an overview over all components created by this setup
3. Pull logs on the metaphactory pod with `kubectl logs metaphactory`
4. You can access the container for advanced troubleshooting through `kubectl exec -it metaphactory -- sh`
5. When using `statefulsets`, please note that the persisted volumes might not be deleted with the StatefulSet. Please use `kubectl delete pvc -l app=metaphactory` to delete the persistent volumes **Note: This will remove all persistent volumes and data as well**

