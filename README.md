# PoC GKE Deployment

In this chapter I'm going to demonstrate how to use resources of your PC in order to deploy your own infrastructure.
The GKE cluster will be deployed in the GCP.

## Prepare Dev Container

In order to keep Operation System away from any application and packages installation we are going to use development container (so we don't need to install anything on your computer except docker-desktop).

After activating the dev container we need to authenticate our-self against out GCP account

```bash
gcloud auth application-default login
```

Now we have an auth token for our GCP account stored in the hidden ".config/gcloud/" folder, which will be used by terraform inside the github runner.

## Run deploymnet process by Github Action

In order to run our pipelines we will use self-hosted Github Action runners.

### Prepare Github Action Controller

* As a platform for running Github Action we will use Docker Desktop
* In order to avoid specifing kubernetes context all the time in the commands, let's set default context

```bash
kubectl config use-context docker-desktop
```

* Cert-manager is the mandatory component for installing the actions-runner-controller (ARC)

```bash
helm repo add jetstack "https://charts.jetstack.io"
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.8.0 \
--set installCRDs=true
```

* Install actions-runner-controller (ARC)

```bash
helm repo add actions-runner-controller "https://actions-runner-controller.github.io/actions-runner-controller"
helm repo update
helm upgrade --install actions-runner-controller actions-runner-controller/actions-runner-controller \
--namespace actions-runner-system \
--create-namespace \
--version 0.21.1 \
--wait \
--values .github/arc/values.yml --set "authSecret.github_token=YOUR_PAT"
```

* Add Runner

```bash
kubectl -n actions-runner-system apply -f .github/arc/RunnerDeployment.yml
kubectl -n actions-runner-system patch RunnerDeployment poc-gke-envs --type='json' \
-p='[{"op": "add", "path": "/spec/template/spec/volumes/0", "value":{"name":"gcp-credentials","hostPath":{"path":"/Users/'$LOGNAME'/.config/"}}}]'
```

* Check if our runnera are working

```bash
root@cte-poc-gke-envs:/automation/cloudinterplay/poc-gke-envs# kubectl -n actions-runner-system get pods
NAME                                         READY   STATUS    RESTARTS   AGE
actions-runner-controller-77f44ff945-pcfcf   2/2     Running   0          1h
poc-gke-envs-dzwvb-99vcq                     2/2     Running   0          1h
poc-gke-envs-dzwvb-mwth5                     2/2     Running   0          1h
```

* Remove Runner for our env's repository

```bash
kubectl -n actions-runner-system delete RunnerDeployment poc-gke-envs
```

## Check the GKE cluster

When deployment pipeline will be completed we can obtain credentials for the cluster and check the content of it.

```bash
gcloud auth login
gcloud config set project ${PROJECT_ID}
gcloud container clusters get-credentials poc-gke-dev --region=us-east1-b
```
