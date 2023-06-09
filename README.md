# Terraform Enterprise

> :heavy_exclamation_mark: You are viewing a beta version of the official helm chart to install Terraform Enterprise. This chart is intended for the beta release of Terraform Enterprise, Flexible Deployment Options, on Kubernetes, and it is not currently meant for production use. Please contact your Customer Success Manager for details before using.

This chart is used to install Terraform Enterprise in a generic Kubernetes environment. It is minimal in its configuration and contains the basic things necessary to launch Terraform Enterprise on a Kubernetes cluster.

## Prerequisites

To use the charts here, [Helm](https://helm.sh/) must be configured for your
Kubernetes cluster. Setting up Kubernetes and Helm is outside the scope of
this README. Please refer to the Kubernetes and Helm documentation.

The versions required are:

  * **Helm 3.0+** - This is the earliest version of Helm tested. It is possible
    it works with earlier versions but this chart is untested for those versions.
  * **Kubernetes 1.25+** - This is the earliest version of Kubernetes tested.
    It is possible that this chart works with earlier versions but it is
    untested.

## Usage

### Before You Begin
Please confirm that you have an operating Kubernetes cluster and that your local kubectl client is configured to operate that cluster. Also confirm that helm is configured appropriately.

```sh
kubectl cluster-info
helm list
```

You'll need the following to continue:

1. A hostname for your TFE instance and some way to provision DNS for this hostname such that, at a minimum, Terraform Enterprise is addressable from your workstation and any pod provisioned inside your Kubernetes cluster. The former is required in order for your workstation to communicate with the Terraform Enterprise installation created here. The later is required in order for tfc-agent instances provisioned for Terraform Enterprise workloads to be able to communicate with the Terraform Enterprise services they require to operate.
1. With the exception of a small number of cases you will need a way to create a dns address for the resulting Terraform Enterprise load balancer public ip address. This DNS address does not need to be generally publicly available, but it does need to be visible to the following use cases:
    * A user interacting with Terraform Enterprise
    * The terraform-enterprise pods within your Kubernetes cluster
    * The agent pods generated for plan / apply activity within your Kubernetes cluster
    * Any additional persistent agents added to your Terraform Enterprise instance or any additional Kubernetes clusters in which agents will be provisioned.
1. A valid TLS certificate and private key provisioned and matching the hostname selected in **1.** in pem format
1. External dependencies : Terraform Enterprise must run under the `external` or `active-active` operational mode when the Kubernetes driver used. The following prerequisite services must be available and configured prior to installing terraform-enterprise:
    * A PostgreSQL server meeting the requirements outlined in [PostgreSQL Requirements for Terraform Enterprise](https://developer.hashicorp.com/terraform/enterprise/requirements/data-storage/postgres-requirements)
    * S3 compatible object storage meeting the requirements outlined in the external services mode section of [Operational Mode Data Storage Requirements](https://developer.hashicorp.com/terraform/enterprise/requirements/data-storage/operational-mode-requirements#external-services-mode).
    * If Terraform Enterprise is running in `active-active` mode then a Redis cache instance is required also meeting the guidance in the above article.
    * Please confirm that all external services have network configuration allowing access from the Kubernetes nodes that may host Terraform Enterprise pods before proceeding.

### Create Prerequisites

1. Clone this repository
1. Create a namespace for terraform-enterprise:
    ```sh
    # Here, terraform-enterprise is the name of the custom namespace.
    kubectl create namespace terraform-enterprise
    ```
1. Create an image pull secret to fetch the `terraform-enterprise` container image from the beta registry. For example, you can create this secret by running this command:
    ```sh
    # replace REGISTRY_USERNAME, REGISTRY_PASSWORD, REGISTRY_URL with appropriate values
    kubectl create secret docker-registry terraform-enterprise --docker-server=REGISTRY_URL --docker-username=REGISTRY_USERNAME --docker-password=REGISTRY_PASSWORD  -n terraform-enterprise
    ```

### Update Chart Configuration

Create a configuration file (`/tmp/overrides.yaml` for the rest of this document) to override the default configuration values in the terraform-enterprise helm chart. **Replace all of the values in this example configuration.**

```yaml
tls:
  certData: BASE_64_ENCODED_CERTIFICATE_PEM_FILE
  keyData: BASE_64_ENCODED_CERTIFICATE_PRIVATE_KEY_PEM_FILE
image:
 repository: REGISTRY_URL
 name: terraform-enterprise
 tag: cab3e8f
env:
  TFE_HOSTNAME: "terraform-enterprise.terraform-enterprise.service.cluster.local"
  TFE_OPERATIONAL_MODE: "external"

```
> :information_source:  Using this hostname exploits the fact that Kubernetes automatically assigns an internal dns address to all services in the form of `<service_name>.<namespace>.svc.cluster.local`. This obviates the need to establish a global dns address for this example.  See the [Quickstart Guide](docs/quickstart.md#dependency-free-terraform-enterprise-quickstart-guide) for more information on this.

> :information_source:  There may be additional customization required for the database credentials, s3 compatible storage configuration, Redis configuration, etc that are often cloud provider or implementation specific.  See [Implementation Examples](docs/implementations.md#implementation-examples) for more information.

> :information_source:  Base64 values for files can be generated by using the `base64` utility. For example: `base64 -i fixtures/tls/privkey.pem`

Add to the `env` entries any configuration required for database, object storage, Redis, or any additional configuration required for your environment. See [Terraform Enterprise Configuration Options](docs/configuration.md#terraform-enterprise-configuration-options) and [Implementation Examples](docs/implementations.md#implementation-examples) for more information.

### Install Terraform Enterprise

This document will assume that the copy of the terraform-enterprise-helm chart is at `./terraform-enterprise-helm`. Install terraform-enterprise-helm: 
```sh
helm install terraform-enterprise ./terraform-enterprise-helm -n terraform-enterprise  --values /tmp/overrides.yaml
```
> :information_source:  The `-n` is for setting a namespace, if no namespace is specified and you did not create a namespace as part of the steps in the prerequisites, the namespace will be automatically set to `default`.

During installation, the helm client will print useful information about which resources were created, what the state of the release is, and also whether there are additional configuration steps you can or should take.

By default, Helm does not wait until all of the resources are running before it exits. Many charts require Docker images that are over 600M in size, and may take additional time to install into the cluster. You can use the `--wait` and `--timeout` flags in helm install to force helm to wait until a minimum number of deployment replicates have passed their health-check based readiness checks before helm returns control to the shell.

To keep track of a release's state, or to re-read configuration information, you can use [helm status](https://helm.sh/docs/helm/helm_status/), IE:

```sh
  helm status terraform-enterprise -n terraform-enterprise
```
    
> :information_source:  When using helm commands, make sure to specify the namespace you created when using the `-n` flag. For this example we are use `terraform-enterprise`.

## Post install
There are a number of common helm or kubectl commands you can use to monitor the installation and the runtime of Terraform Enterprise. We list some of them here. We assume that the namespace is `terraform-enterprise`. If you have a different namespace, replace it with yours.

* To see releases:
  ```sh
  helm list -n terraform-enterprise
  ```

* To check the status of the Terraform Enterprise pod:
  ```sh
    kubectl get pod -n terraform-enterprise
  ```
  In the output, the `STATUS` should be in `Running` state and the `READY`section should show `1/1`. e.g:
  ```sh
  NAME                                   READY   STATUS    RESTARTS   AGE
  terraform-enterprise-5946d99fc-l22s9   1/1     Running   0          25m
  ```
  If this is not the case, you can use the following steps to debug:
* Check pod logs:
  ```sh
  kubectl logs terraform-enterprise-5946d99fc-l22s9
  ```
* To diagnose issues with the terraform-enterprise deployment such as image pull errors, run the following command:
  ```sh
  kubectl describe deployments -n terraform-enterprise
  ```
* Exec into the pod if possible:
  ```sh
  kubectl exec -it terraform-enterprise-5946d99fc-l22s9 -- /bin/bash
  ```
* In the Terraform Enterprise pod, run:
  ```sh
  supervisorctl status
  ```
  This should show you which service failed. From outside the pod you can also do this:
  ```sh
  kubectl exec -it terraform-enterprise-5946d99fc-l22s9 -- supervisorctl status
  ```

* All Terraform Enterprise services logs can be found in the pod here `/var/log/terraform-enterprise/`. E.g:
  ```sh
  cat /var/log/terraform-enterprise/atlas.log
  ```
  From outside the pod, this will be:
    ```sh
    kubectl exec -it terraform-enterprise-5946d99fc-l22s9 -- cat /var/log/terraform-enterprise/atlas.log
    ```

## Bootstrap Terraform Enterprise

### Establish DNS

Once Terraform Enterprise has loaded and passed all startup health checks you should take the following actions, the details of which are particular to your environment:

* Expose the terraform-enterprise load balancer service to network access from your workstation
* Establish a DNS address or a host file entry for the Terraform Enterprise load balancer public ip address and hostname
* Install the CA certificate for your instance certificate on your workstation if necessary
* Confirm the above actions by visiting the Terraform Enterprise health check endpoint at `https://<terraform_enterprise_hostname>/_health_check`

### Create an Administrative User

In order to retrieve an _initial admin creation token_ or and iact visit the `admin/retrieve-iact` url path with a browser or curl from a source ip address and time in agreement with your `TFE_IACT_SUBNETS` and `TFE_IACT_TIME_LIMIT` settings.

```shell
curl https://terraform-enterprise.terraform-enterprise.svc.cluster.local/admin/retrieve-iact
> 1b5c826a739fe1e2b91cc5932f7adda204bfefcf4bcbe006ac88831d8d208114
```

Then use this token as a url parameter for the initial admin creation workflow:

`https://terraform-enterprise.terraform-enterprise.svc.cluster.local/admin/account/new?token=1b5c826a739fe1e2b91cc5932f7adda204bfefcf4bcbe006ac88831d8d208114`

![An image of the admin account creation page using the iact token](./docs/images/account_creation.png)

Congratulations, you're ready to start using Terraform Enterprise!  Create an organization and get started.

## Additional Documentation

For more information about Terraform Enterprise and the capabilities of this helm chart please see the following additional documentation:

* [Dependency Free Terraform Enterprise Quickstart Guide](docs/quickstart.md#dependency-free-terraform-enterprise-quickstart-guide)
* [Terraform Enterprise Application Configuration Options](docs/configuration.md#terraform-enterprise-application-configuration-options)
* [Examples of Common Implementations](docs/implementations.md#implementation-examples)
* [Terraform Enterprise Common Kubernetes Configuration](docs/kubernetes_configuration.md#terraform-enterprise-common-kubernetes-configuration)