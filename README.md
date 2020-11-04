# Devops Junior Interview

> Use more than one terminal to do the job
> You'll need 4 terminal:
>  - Forward Port from service Vault to your machine
>  - See the Vault Controller Manager logs
>  - Apply changes on kubernetes (kubectl)
> 
> 
> !
> Don't forget the commands that are already in the path of the vault-controller git repository: mean inside vault-controller directory


# Install and configure Docker

Following the link:

`https://docs.docker.com/get-docker/`

# Install and configure kind

Following the link: 

`https://kind.sigs.k8s.io/docs/user/quick-start/`

Use the config file bellow to create the cluster with HA - High Availability

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

`kind create cluster --config=kind-config-HA.yaml`

# Install and Configure Vault Controller

## Get project from Gitlab - Laredoute project, using git tool

`https://gitlab.com/laredoute/infra/k8s/vault-controller`

> Remember that all of these steps will be in the path of the vault-controller repository

`cd vault-controller`

## Build manager

On the Vault Controller project root path, you can compile the project:

### Build Go Binary (manager)

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO115MODULE=on go build -a -o manager main.go

### Build a image to load on Kind

#### Buld Image

`docker build -t interview-redoute-vault-controller:latest -f Dockerfile .`

#### Load on Kind

`kind load docker-image interview-redoute-vault-controller:latest`

**Check if the image are on the Kind**

```
kubectl get nodes 

docker exec -it kind-worker crictl images

```

Something like that will be shown to you

```
IMAGE                                                  TAG                  IMAGE ID            SIZE
docker.io/hashicorp/vault-k8s                          0.6.0                7217fb6a4d71b       19.3MB
docker.io/kindest/kindnetd                             v20200725-4d6bea59   b77790820d015       119MB
docker.io/library/interview-redoute-vault-controller   latest               25c1a9097a2bd       44.3MB
docker.io/rancher/local-path-provisioner               v0.0.14              e422121c9c5f9       42MB
k8s.gcr.io/build-image/debian-base                     v2.1.0               c7c6c86897b63       53.9MB
k8s.gcr.io/coredns                                     1.7.0                bfe3a36ebd252       45.4MB
k8s.gcr.io/etcd                                        3.4.13-0             0369cf4303ffd       255MB
k8s.gcr.io/kube-apiserver                              v1.19.1              8cba89a89aaa8       95MB
k8s.gcr.io/kube-controller-manager                     v1.19.1              7dafbafe72c90       84.1MB
k8s.gcr.io/kube-proxy                                  v1.19.1              47e289e332426       136MB
k8s.gcr.io/kube-scheduler                              v1.19.1              4d648fc900179       65.1MB
k8s.gcr.io/pause                                       3.3                  0184c1613d929       686kB

```


## Change the image name on the deploy.yaml file

Change the name of the image on deploy.yaml file in line 424:

`vi +424 config/deploy.yaml`

Change the image name and tag, and add a new policy for pull image:

```
        image: interview-redoute-vault-controller:latest
        imagePullPolicy: Never
```

## Kubernetes Environment

## Vault

We'll install the Vault first of the Vault Controller

**Create Vault namespace**

`kubectl create namespace vault`

**Add the Vault Helm Chart Repository**

`helm repo add hashicorp https://helm.releases.hashicorp.com`

`helm search repo hashicorp/vault`

**Install Vault**

`helm install vault hashicorp/vault --namespace vault`

**Check if Vault is running**

If in __ContainerCreating__ state, waint some seconds (will be depend on your network connectivity)

`kubectl -n vault get all`

```
NAME                                        READY   STATUS    RESTARTS   AGE
pod/vault-0                                 0/1     Running   0          9m25s
pod/vault-agent-injector-544f8c5f4f-dn8r8   1/1     Running   0          9m26s

NAME                               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/vault                      ClusterIP   10.96.15.182   <none>        8200/TCP,8201/TCP   9m26s
service/vault-agent-injector-svc   ClusterIP   10.96.37.56    <none>        443/TCP             9m27s
service/vault-internal             ClusterIP   None           <none>        8200/TCP,8201/TCP   9m27s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-agent-injector   1/1     1            1           9m26s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-agent-injector-544f8c5f4f   1         1         1       9m26s

NAME                     READY   AGE
statefulset.apps/vault   0/1     9m26s


```

**Inicialize Vault**

Get the Vault service name

`kubectl -n vault get svc`

```
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
vault                      ClusterIP   10.96.15.182   <none>        8200/TCP,8201/TCP   10m
vault-agent-injector-svc   ClusterIP   10.96.37.56    <none>        443/TCP             10m
vault-internal             ClusterIP   None           <none>        8200/TCP,8201/TCP   10m
```

**Forward the Vault port to your machine**

This will be forward the port 8200 of Vault to your machine on port 8210. This is needed to you inicialize the Vault and get the rootToken:

`kubectl -n vault port-forward svc/vault 8210:8200`

Go to the your browser and access the: http://localhost:8210

When you access for the first time, you'll see a init process wizard, on bellow inputs configure:

Key shares: 1
Key threshold: 1

[img/vault-init-master-keys.png]

**Anotate the root token**

Anotate the root token generated on the last step of the wizard. Ex.;

[img/vault-init-root-and-unseal-keys.png]

rootToken: s.ysdGvqWNNnOtv9ZYwsEz1QPT

## Vault controller

Change the `config/vault-controller-config.yaml` to set root Token and Vault Address:

`vi +11 config/vault-controller-config.yaml` 

To something like that:
```
  address: "http://vault.vault.svc.cluster.local:8200"
  token: "s.ysdGvqWNNnOtv9ZYwsEz1QPT"
```

Creating namespace vault-controller-system

`kubectl create namespace vault-controller-system`

Label namespace

`kubectl label namespace vault-controller-system redoute.io/vault-controller-system-communication="true"`


**Pre Requirements to Deploy**

**Apply pre-deploy**

That files will create a requirements to deploy an app on the Kubernetes, being them:

- Service account
- Role
- RoleBinding
- PodSecurityPolicy
- configMap with the Vault token

```
kubectl apply --dry-run -f config/pre-deploy.yaml -o yaml | kubectl apply -f -
kubectl apply --dry-run -f config/vault-controller-config.yaml -o yaml | kubectl apply -f -
kubectl apply --dry-run -f config/deploy.yaml -o yaml | kubectl apply -f -
```

Wait for POD's UP and Running.

See the vault-controller-manager logs:

```
kubectl --namespace vault-controller-system get pod

# anotate the pod name

kubectl --namespace vault-controller-system logs vault-controller-controller-manager-8597f547d4-d59gv -c manager -f
```

On another terminal create a policy to test the access to the Vault Controller and Vault

`k apply -f config/policies/default/policy-admin-integration.yaml`

> Remember the Vault Controller path (from repository)

You'll see something on the Vault Controller manager logs:

```
2020-11-04T16:17:01.807Z        INFO    controllers.Policy      starting reconcile loop for vault-controller-system/policy-admin-integration    {"policy": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.907Z        INFO    controllers.Policy      creating/updating policy admin-integration-operator
2020-11-04T16:17:01.932Z        INFO    controllers.Policy      add finalizer for vault-controller-system/policy-admin-integration
2020-11-04T16:17:01.945Z        INFO    controllers.Policy      completed reconcile loop for vault-controller-system/policy-admin-integration   {"policy": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.945Z        DEBUG   controller-runtime.controller   Successfully Reconciled {"controller": "policy", "request": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.945Z        INFO    controllers.Policy      starting reconcile loop for vault-controller-system/policy-admin-integration    {"policy": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.945Z        DEBUG   controller-runtime.manager.events       Normal  {"object": {"kind":"Policy","namespace":"vault-controller-system","name":"policy-admin-integration","uid":"149224ec-1a6f-4e1c-a6b4-5902b5b7411e","apiVersion":"vault.redoute.io/v1","resourceVersion":"12564"}, "reason": "added", "message": "object finalizer is added"}
2020-11-04T16:17:01.945Z        INFO    controllers.Policy      completed reconcile loop for vault-controller-system/policy-admin-integration   {"policy": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.945Z        DEBUG   controller-runtime.controller   Successfully Reconciled {"controller": "policy", "request": "vault-controller-system/policy-admin-integration"}
2020-11-04T16:17:01.945Z        DEBUG   controller-runtime.manager.events       Normal  {"object": {"kind":"Policy","namespace":"vault-controller-system","name":"policy-admin-integration","uid":"149224ec-1a6f-4e1c-a6b4-5902b5b7411e","apiVersion":"vault.redoute.io/v1","resourceVersion":"12564"}, "reason": "updated", "message": "policy is updated"}


```

Go to the Vault address (don't forgot to forward the Vault Port) on your browser:

`http://localhost:8210`

And validate if vault-controller created the policy with name: **admin-integration-operator**

