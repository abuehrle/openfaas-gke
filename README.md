# OpenFaaS GKE

![OpenFaaS](https://github.com/stefanprodan/openfaas-gke/blob/master/screens/banner.jpg)

OpenFaaS is a serverless framework that runs on top of Kubernetes. It comes with a built-in UI and a handy CLI that 
takes you from scaffolding new functions to deploying them on your Kubernetes cluster. What's special about OpenFaaS 
is that you can package any executable as a function, as long as it runs in a Docker container it will work on OpenFaaS. 

What follows is a step by step guide on running OpenFaaS with Kubernetes 1.8 on Google Cloud.

### Create a GCP project

Login into GCP and create a new project named openfaas. If you don't have a GCP account you can apply for 
a free trial. After creating the project, enable billing and wait for API and related services to be enabled.
Download and install the Google Cloud SDK from this [page](https://cloud.google.com/sdk/). After installing 
the SDK run `gcloud init` and set the default project to `openfaas` and the default zone to `europe-west3-a`.

Install `kubectl` using `gcloud`:

```bash
gcloud components install kubectl
```

Go to _Google Cloud Platform -> API Manager -> Credentials -> Create Credentials -> Service account key_ and 
chose JSON as key type. Rename the file to `account.json` and put it in the project root.
Add your SSH key under _Compute Engine -> Metadata -> SSH Keys_, also create a metadata entry named `sshKeys` 
with your public SSH key as value.

### Create a Kubernetes cluster

Create a three nodes cluster with each node on a different zone:

```bash
k8s_version=$(gcloud container get-server-config --format=json | jq -r '.validNodeVersions[0]')

gcloud container clusters create demo \
    --cluster-version=${k8s_version} \
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
gcloud container clusters get-credentials demo -z=europe-west3-a
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

### Deploy OpenFaaS with basic authentication

Clone `openfaas-gke` repo:

```bash
git clone https://github.com/stefanprodan/openfaas-gke
cd openfaas-gke
```

Create the `openfaas` and `openfaas-fn` namespaces:

```bash
kubectl apply -f ./namespaces.yaml
```

Deploy OpenFaaS services in the `openfaas` namespace:

```bash
kubectl apply -f ./openfaas
```

This will create the pods, deployments and services for OpenFaaS gateway, faas-netesd (K8S controller), 
Prometheus, Alert Manager, Nats and the Queue worker.

Before exposing OpenFaaS on the internet we need to setup authentication. 
First create a basic-auth secret with your username and password:

```bash
kubectl -n openfaas create secret generic basic-auth \
    --from-literal=user=admin \
    --from-literal=password=admin
```

Deploy Caddy as a reverse proxy for OpenFaaS gateway:

```bash
kubectl apply -f ./caddy
```

Wait for an external IP to be allocated and use it to access the OpenFaaS gateway UI 
with your credentials at `http://<EXTERNAL-IP>`. You can get the external IP by running `kubectl get svc`.

```bash
get_gateway_ip() {
    kubectl -n openfaas describe service caddy-lb | grep Ingress | awk '{ print $NF }'
}

until [[ "$(get_gateway_ip)" ]]
 do sleep 1;
 echo -n ".";
done
echo "."
gateway_ip=$(get_gateway_ip)
echo "OpenFaaS Gateway IP: ${gateway_ip}"
```

Install OpenFaaS CLI:

```bash
curl -sL cli.openfaas.com | sudo sh
```

Login with the CLI:

```bash
faas-cli login -u admin -p admin --gateway http://<EXTERNAL-IP>
```

If you want to avoid having your password in bash history, you could create a text file with it and use that 
along with the `--password-stdin` flag:

```bash
cat ~/faas_pass.txt | faas-cli login -u admin --password-stdin --gateway http://<EXTERNAL-IP>
```

You can logout at any time with:

```bash
faas-cli logout --gateway http://<EXTERNAL-IP>
```

### Deploy functions

Create a stack file named `stack.yml` containing two function:

```yaml
provider:
  name: faas
  gateway: http://<EXTERNAL-IP>

functions:
  nodeinfo:
    lang: nodejs
    handler: node main.js
    image: functions/nodeinfo:burner
  echo:
    lang: Dockerfile
    handler: ./echo
    image: functions/faas-echo:latest
```

Deploy `nodeinfo` and `echo` on OpenFaaS:

```bash
faas-cli deploy
```

The `deploy` command will look for a `stack.yml` file in the current directory and will deploy all functions in the 
openfaas-fn namespace.

Invoke the functions:

```bash
echo -n "test" | faas-cli invoke echo --gateway=http://<EXTERNAL-IP>
echo -n "" | faas-cli invoke nodeinfo --gateway=http://<EXTERNAL-IP>
```

Load testing:

```bash
#install hey
go get -u github.com/rakyll/hey

#do 1K requests 
hey -n 1000 -c 10 -m POST -d "test" http://admin:admin@<EXTERNAL-IP>/function/nodeinfo
```

In the Weave Cloud UI under Explore you'll see how OpenFaaS scales up the nodeinfo service:

![scaling](https://github.com/stefanprodan/openfaas-gke/blob/master/screens/scaling.png)

You can also monitor the scale up/down events with GCP Stackdrive Logs using this advanced filter:

```bash
resource.type: "container"
logName: "projects/openfaas/logs/gateway"  
labels."container.googleapis.com/namespace_name": "openfaas"
textPayload: "alerts"
```

Weave Cloud extends Prometheus by providing a distributed, multi-tenant, horizontally scalable version of Prometheus. 
It hosts the scraped Prometheus metrics for you, so that you don’t have to worry about storage or backups.

You can monitor your OpenFaaS setup by writing PromQL queries in the Weave Cloud Monitor GUI:

![cortex](https://github.com/stefanprodan/swarm-gcp/blob/master/screens/openfaas-metrics.png)


### Create functions

With OpenFaaS CLI you can chose between using a programming language template where you only need to provide a 
handler file, or a Docker that you can build yourself. There are many supported languages like 
Go, JS (node), Python, C# and Ruby.

Lets create a function with Go that will fetch the SSL/TLS certificate info for a given URL.

First create a directory for your functions under `GOPATH`:

```bash
mkdir -p $GOPATH/src/functions
```

Inside the `functions` dir use the CLI to create the `certinfo` function:

```bash
cd $GOPATH/src/functions
faas-cli new --lang go certinfo
```

Open `handler.go` in your favorite editor and add the certificate fetching code:

```go
package function

import (
	"crypto/tls"
	"fmt"
	"net"
	"time"
)

func Handle(req []byte) string {
	address := string(req) + ":443"
	ipConn, err := net.DialTimeout("tcp", address, 2*time.Second)
	if err != nil {
		return fmt.Sprintf("Dial error: %v", err)
	}
	defer ipConn.Close()
	conn := tls.Client(ipConn, &tls.Config{
		InsecureSkipVerify: true,
	})
	if err = conn.Handshake(); err != nil {
		return fmt.Sprintf("Handshake error: %v", err)
	}
	defer conn.Close()
	addr := conn.RemoteAddr()
	host, port, err := net.SplitHostPort(addr.String())
	if err != nil {
		return fmt.Sprintf("Error: %v", err)
	}
	cert := conn.ConnectionState().PeerCertificates[0]

	return fmt.Sprintf("Host %v\nPort %v\nIssuer %v\nCommonName %v\nNotBefore %v\nNotAfter %v\nSANs %v\n",
		host, port, cert.Issuer.CommonName, cert.Subject.CommonName, cert.NotBefore, cert.NotAfter, cert.DNSNames)
}
```

Next to `handler.go` create `handler_test.go` with the following content:

```go
package function

import (
	"regexp"
	"testing"
)

func TestHandleReturnsCorrectResponse(t *testing.T) {
	expected := "Google Internet Authority"
	resp := Handle([]byte("google.com"))

	r := regexp.MustCompile("(?m:" + expected + ")")
	if !r.MatchString(resp) {
		t.Fatalf("\nExpected: \n%v\nGot: \n%v", expected, resp)
	}
}
```

Modify the certinfo.yml file and add your Docker Hub username to the image name:

```yaml
provider:
  name: faas
  gateway: http://localhost:8080

functions:
  certinfo:
    lang: go
    handler: ./certinfo
    image: stefanprodan/certinfo
```

Now let's build the function into a docker image:

```bash
faas-cli build -f certinfo.yml
```

This will check your code for proper formatting with `gofmt`, run `go build & test` and pack your binary into 
an alpine image. If everything goes well, you'll have a local Docker image named `username/certinfo:latest`. 

Let's push this image to Docker Hub. First create a public repository named `certinfo`, login to Docker Hub using 
docker CLI and push the image:

```bash
docker login
faas-cli push -f certinfo.yml
```

Once the image is on Docker Hub you can deploy the function to your OpenFaaS GKE cluster:

```bash
faas-cli deploy -f certinfo.yml --gateway=http://<EXTERNAL-IP>
```

Invoke certinfo with:

```bash
$ echo -n "www.openfaas.com" | faas-cli invoke certinfo --gateway=<EXTERNAL-IP>

Host 147.75.74.69
Port 443
Issuer Let's Encrypt Authority X3
CommonName www.openfaas.com
NotBefore 2017-10-06 23:54:56 +0000 UTC
NotAfter 2018-01-04 23:54:56 +0000 UTC
SANs [www.openfaas.com]
```

### Conclusions

Like most serverless frameworks, OpenFaaS empowers developers to build, ship and run code in the cloud. 
What OpenFaaS adds to this, is portability between dev environments and production with no vendor locked-in or 
proprietary solutions. 
OpenFaaS is a promising project with over 64 contributors and a vibrant community. Even if it's a young project you 
can run it reliably in production backed by Kubernetes and Google Cloud. 


