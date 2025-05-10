# GitHub Actions: Self hosted runners

## Pre-requisites

We need to install the below tools to run and test the project.

* [Docker Desktop](https://www.docker.com/products/docker-desktop/)
* [Amazon AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Kubernetes CLI (Kubectl)](https://kubernetes.io/docs/tasks/tools/)
* [eksctl CLI utility](https://eksctl.io/installation/)
* [Minikube (for creating local kubernetes cluster for testing)](https://minikube.sigs.k8s.io/)


Also, apart from the above tools we need:
* Github Account
* Github PAT Token (Personal Access Token)

## Create a kubernetes cluster in local

In this Project we'll need a Kubernetes cluster for testing in local machine. Let's create one using [minikube](https://minikube.sigs.k8s.io/) </br>

```
minikube start --profile githubactions --kubernetes-version=v1.32.0 --driver=docker --memory=4g --cpus=2
```

This would create a cluster in local by provisioning 2 core CPU and 4 GB RAM single node.

Let's test our cluster:
```
minikube status -p githubactions

githubactions
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

kubectl get nodes
NAME                          STATUS   ROLES           AGE     VERSION
githubactions-control-plane   Ready    control-plane   2m53s   v1.28.0
```

## Running the Runner in Docker 

We can simply install this directly on to virtual machines , but for this project, We have decided to run it in Kubernetes inside a docker container. </br>

### Security notes

* Running in Docker needs high priviledges.
* Would not recommend to use these on public repositories. but, we have kept our repository public for the sake of assignment.
* Would recommend to always run your CI systems in seperate Kubernetes clusters and not in the cluster where the runners are working.

### Creating a Dockerfile

* Installing Docker CLI 
For this to work we need a `dockerfile` and follow instructions to [Install Docker](https://docs.docker.com/engine/install/debian/).
We would grab the content and create statements for my `dockerfile` </br>

Now notice that we only install the `docker` CLI. </br> 
This is because we want our running pipelines to be able to run docker commands , but the actual docker server runs elsewhere </br>
This gives you flexibility to tighten security by running docker on the host itself and potentially run the container runtime in a non-root environment </br>

* Installing Github Actions Runner 

Next up we will need to install the [GitHub actions runner](https://github.com/actions/runner) in our `dockerfile`
Now to give a "behind the scenes" of how we usually build our `dockerfile`s, We first build our docker image: 

```
docker build . -t bits-github-runner:latest 
```

Next steps:

* Now we can see `docker` is installed 
* To see how a runner is installed, lets go to our repo | runner and click "New self-hosted runner"
* Try the steps mentioned in the github docs in the docker container.
* We will need few dependencies, we have added them in line 29
* We download the runner in the container



Finally lets test the runner in `docker` container 

```
docker run -it -e GITHUB_PERSONAL_TOKEN="" -e GITHUB_OWNER=spandan1460 -e GITHUB_REPOSITORY=self-hosted-runner-BITS-Assignment bits-github-runner
```


Now, that our runner works successfully in the container, we would proceed by building containers inside the local kubernetes cluster.

## Deploy to local Kubernetes for testing 

Load our github runner image so we dont need to push it to a registry:

```
minikube image load bits-github-runner:latest -p githubactions
```

Create a new namespace for our github runners
```
kubectl create ns github-runners
```


Create a kubernetes secret with our github details 

```
kubectl -n github-runners create secret generic github-secret `
  --from-literal GITHUB_OWNER=spandan1460 `
  --from-literal GITHUB_REPOSITORY=self-hosted-runner-BITS-Assignment `
  --from-literal GITHUB_PERSONAL_TOKEN=""
```

Apply the kubernetes manifest yaml files to deploy the kubernetes manifests
```
kubectl -n github-runners apply -f kubernetes.yaml
kubectl -n github-runners apply -f Pod-Autoscalers.yaml
```


## Create a EKS Cluster for our runners

Now, we will use the eksctl CLI utility to create our clusters.

- All the configurations of the clusters can be found in a YAML format in [self-hosted-runner/eksctl.yaml](https://github.com/spandan1460/self-hosted-runner-BITS-Assignment/blob/main/self-hosted-runner/eksctl.yaml)


```
eksctl create cluster -f self-hosted-runner/eksctl.yaml
```

## Create AWS ECR Container Registry for Storing Docker images

We are not ready yet, our EKS Cluster cannot store Docker images inside the cluster. So, we need an Container Registry for it. Hence, for our project we would be using AWS ECR(Elastic Container Registry) services to store our docker images, for our EKS clusters to pull the docker images and create containers out of it.

Here, is the AWS CLI command to create one ECR(Elastic Container Registry) Service inside our AWS VPC.

```
aws ecr create-repository --repository-name github-runner
```
