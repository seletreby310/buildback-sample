# Automated Code to URL on Kubernetes using Cloud Native Buildpacks, Knative and ArgoCD
Technologies like Docker and Kubernetes simplify the process of building, running and maintaining cloud native applications. At the same time taking source code, building it into a container image and turning it into a deployed application on Kubernetes can be a time consuming process. A large part of the process can be automated with Continuous Integration Systems (CI) and Continuous Deployment(CD) Systems. However there are stages in the build and deployment phases that still need to be defined manually. Most CI systems aren’t application aware. They cannot build automated Docker images from source code unless explicitly defined in a spec file like Dockerfiles or a config file that a CI system can understand. Due to which, Apart from writing application code, you also need to manually define and test Dockerfiles or config files to convert source code into an executable container image. When deploying the image onto Kubernetes, you need to then define various Kubernetes constructs needed to run the application. Like Deployments, StatefulSets, Services, Ingress etc. This process can add errors, security gaps and overhead.


- [Cloud Native Buildpacks](https://buildpacks.io/) to automate Container Image build process. Cloud Native Buildpacks is a specification that defines how OCI compliant containers can be build, removing the need to specify or build Dockerfiles. They can automatically detect the language an application is written in and determine the best and most secure way to package applications in a container image. Cloud Native Buildpacks can also be used to update container images easily for any changes. For this guide we will use an implementation of Cloud Native Buildpacks called [kpack](https://github.com/pivotal/kpack). kpack lets you use deploy Cloud Native Buildpacks on a Kubernetes Cluster. (See [What are Cloud Native Buildpacks?](https://tanzu.vmware.com/developer/guides/cnb-what-is/) for more on Cloud Native Buildpacks, and kpack.)

- [Knative](Knative) to automatically generate an Ingress Service with URL and other Kubernetes Resources for the container image that was built using Cloud Native Buildpacks. Knative Serving automates the process of creating Kubernetes objects needed for an application like Deployment, Replicasets, Services etc., eliminating the need to write complex Kubernetes YAML files.

- [ArgoCD](https://argoproj.github.io/) to automate deployment pipeline pushing container images on to Kubernetes using Knative. ArgoCD helps deploy application continuously using GitOps methodology. It can take specifications like Kubernetes resources, Knative, Kustomize etc. to deploy application on Kubernetes.

A sample application called [Petclinic](https://github.com/Boskey/spring-petclinic) that is based on [Spring](https://spring.io/)

- [Kind](https://kind.sigs.k8s.io/) as a local Kubernetes Cluster

In summary, our overall workflow will be to take the sample application in Spring, use Cloud Native Buildpacks/kpack to convert source code into a container image, use Knative Serving to create a deployment using ArgoCD. This process will eliminate the need to write Dockerfiles or any Kubernetes resource YAML files.

## Assumptions and prerequisites
There are a few things you will need before getting started

 * You have [kubectl](https://kubernetes.io/docs/tasks/tools/), a tool to interact with Kubernetes Cluster installed.

 * [Docker Desktop](https://www.docker.com/products/docker-desktop) is installed on your laptop/machine with at least 4 GB of memory and 4 CPU’s allocated to Docker Resources.

 * You have access to [Docker Hub Repository](https://hub.docker.com/) to store container images.

 * You have an account in [Github](https://github.com/) to clone the app Petclinic

### 1. Prepare a Kubernetes Cluster and clone Sample Application

We will deploy a Kind cluster using Docker Desktop and install [Contour](https://projectcontour.io/) on it to help provide Ingress management. Contour along with [Envoy](https://www.envoyproxy.io/) Proxy will help create service and URL management for Knative.

```
brew install kind
```
Create a Kubernetes Cluster called tdp-guide and set Kubectl context to tdp-guide
```
kind create cluster --name tdp-guide
kubectl cluster-info --context kind-tdp-guide
```
Log onto Github and Fork the repository for our sample app [Petclinic](https://github.com/Boskey/spring-petclinic)

### 2. Install Knative Serving
```
kubectl apply -f https://github.com/knative/serving/releases/download/v0.22.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/v0.22.0/serving-core.yaml
```
### 3. Install Contour Ingress Controller
```
kubectl apply -f https://github.com/knative/net-contour/releases/download/v0.22.0/contour.yaml
kubectl apply -f https://github.com/knative/net-contour/releases/download/v0.22.0/net-contour.yaml
```
Change Knative Serving config to use Contour Ingress
```
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress.class":"contour.ingress.networking.knative.dev"}}'
  ```
  
