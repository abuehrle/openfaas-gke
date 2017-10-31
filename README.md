# OpenFaaS GKE

A step by step guide on running OpenFaaS with Kubernetes 1.8.1 on Google Cloud.

### Create a GCP project

Login into GCP and create a new project named openfaas. If you don't have a GCP account you can apply for 
a free trial. After creating the project, enable billing and wait for API and related services to be enabled.
Download and install the Google Cloud SDK from this [page](https://cloud.google.com/sdk/). After installing 
the SDK run `gcloud init` and set the default project to `openfaas` and the default zone to `europe-west3-a`.

Install `kubectl` using `gcloud`:

```bash
gcloud components install kubectl
```

Clone `openfaas-gke` repo:

```bash
git clone https://github.com/stefanprodan/openfaas-gke
cd openfaas-gke
```

Go to _Google Cloud Platform -> API Manager -> Credentials -> Create Credentials -> Service account key_ and 
chose JSON as key type. Rename the file to `account.json` and put it in the project root.
Add your SSH key under _Compute Engine -> Metadata -> SSH Keys_, also create a metadata entry named `sshKeys` 
with your public SSH key as value.

### Create a Kubernetes cluster

Create a three nodes cluster with each node on a different zone:

```bash
gcloud container clusters create demo \
    --cluster-version=1.8.1-gke.0 \
    --zone=europe-west3-a \
    --additional-zones=europe-west3-b,europe-west3-c \
    --num-nodes=1 \
    --machine-type=n1-standard-1 \
    --scopes=default,storage-rw
```

You can delete the cluster at any time with:

```bash
gcloud container clusters delete demo -z=europe-west3-a 
```

Setup credentials for `kubectl`:

```bash
gcloud container clusters get-credentials demo
```

Create a cluster admin user:

```bash
kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
    --clusterrole=cluster-admin \
    --user="$(gcloud config get-value core/account)"
```

Grant admin privileges to kubernetes-dashboard:

```bash
kubectl create clusterrolebinding kube-system-cluster-admin \
    --clusterrole cluster-admin \
    --user system:serviceaccount:kube-system:default
```

You can access the kubernetes-dashboard at `http://localhost:9099/ui` using kubectl reverse proxy:

```bash
kubectl proxy --port=9099 &
```

### Create a Weave Cloud project

Now that you have a Kubernetes cluster up and running you can start monitoring it with Weave Cloud. 
You'll need a Weave Could service token, if you don't have a Weave token go 
to [Weave Cloud](https://cloud.weave.works/) and sign up for a trial account. 

Deploy Weave Cloud agents:

```bash
kubectl apply -n kube-system -f \
"https://cloud.weave.works/k8s.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')&t=<WEAVE-TOKEN>"
```

Navigate to Weave Cloud Explore to inspect your K8S cluster:

![nodes](https://github.com/stefanprodan/openfaas-gke/blob/master/screens/nodes.png)

### Deploy OpenFaaS with  basic authentication

Deploy OpenFaaS services in the default namespace:

```bash
kubectl apply -f ./faas.yml
```

Create a basic-auth secret with your username and password:

```bash
kubectl create secret generic basic-auth \
    --from-literal=user=admin \
    --from-literal=password=admin
```

Deploy Caddy as a reverse proxy for OpenFaaS gateway:

```bash
kubectl apply -f caddy.yml
```

Expose Caddy on the internet:

```bash
kubectl expose deployment caddy --type=LoadBalancer --name=caddy-lb
```

Wait for an external IP to be allocated and use it to access the OpenFaaS gateway UI 
with your credentials at `http://<EXTERNAL-IP>`.

Install OpenFaaS CLI:

```bash
curl -sL cli.openfaas.com | sudo sh
```

Login with the CLI:

```bash
faas-cli login -u admin -p admin --gateway http://<EXTERNAL-IP>
```

### Deploy functions

Deploy nodeinfo function:

```bash
faas-cli deploy --name=nodeinfo \
    --image=functions/nodeinfo:latest \
    --fprocess="node main.js" \
    --network=default \
    --gateway=http://<EXTERNAL-IP>:8080 
```

Invoke nodeinfo function:

```bash
echo -n "" | faas-cli invoke nodeinfo --gateway http://<EXTERNAL-IP>:8080
```

Load testing:

```bash
#install hey
go get -u github.com/rakyll/hey

#do 10K requests 
hey -n 1000 -c 10 -m POST -d "test" http://admin:admin@<EXTERNAL-IP>/function/nodeinfo
```

Monitor the auto-scaling with Weave Cloud Explore:

![scaling](https://github.com/stefanprodan/openfaas-gke/blob/master/screens/scaling.png)
