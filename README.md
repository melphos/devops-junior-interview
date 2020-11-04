# devops-junior-interview

# Install/configure kind

Following the link: 

`https://kind.sigs.k8s.io/docs/user/quick-start/`

# kind-config-HA.yaml

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

## Creating Kind Cluster

`kind create cluster --config=KindNodes.yaml`


# Install and Configure Vault Controller

## Get project from Gitlab - Laredoute project, using git tool

`https://gitlab.com/laredoute/infra/k8s/vault-controller`


## Build manager

On the Vault Controller project root path, you can compile the project:

### Build Go Binary (manager)

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO115MODULE=on go build -a -o manager main.go

### Build a image to load on Kind

#### Buld Image

`docker build -t interview-redoute-vault-controller:latest -f Dockerfile .`

#### Load on Kind

`kind load docker-image interview-redoute-vault-controller:latest`

## Change the image name on the deploy.yaml file

Change the name of the image on deploy.yaml file in line 424:

`vi +424 config/deploy.yaml`

Change the image name and tag, and add a new policy for pull image:

```
        image: interview-redoute-vault-controller:latest
        imagePullPolicy: Never
```

## Kubernetes Environment

Vault Controller need a namespace to working and share the roles and policies

## Creating namespace vault-controller-system

`kubectl create namespace vault-controller-system`

## Label namespace

`kubectl label namespace vault-controller-system redoute.io/vault-controller-system-communication="true"`


# Pre Requirements to Deploy

## Apply pre-deploy

That files will create a requeriments to deploy any app on the Kubernetes, being them:

- Service account
- Role
- RoleBinding
- 

```
kubectl apply --dry-run -f config/pre-deploy.yaml -o yaml | kubectl apply -f -
kubectl apply --dry-run -f config/vault-controller-config.yaml -o yaml | kubectl apply -f -
kubectl apply --dry-run -f config/deploy.yaml -o yaml | kubectl apply -f -
```


