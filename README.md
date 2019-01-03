[![Build Status](https://dev.azure.com/pjawesome/winops-london/_apis/build/status/pierluigi.winops-london?branchName=master)](https://dev.azure.com/pjawesome/winops-london/_build/latest?definitionId=2?branchName=master)

# winops-london

Repository to hold demo files for [Alessandro's talk](https://www.winops.org/london/agenda/bluegreen.php) at WinOps London 2018.


## Blue/Green and Canary Deployments with Azure DevOps, Istio and AKS

Room: CTRL

Talk Time: 2:45 - 3:30pm

In this demo-driven talk I will show how you can implement advanced DevOps concepts like blue/green and canary deployment using Azure Pipelines targeting a polyglot application deployed to an Azure Kubernetes Cluster using Helm. Istio is used to shape traffic to different versions of the same microservice giving full control on what your users see and controlling the flow of releases throughout the pipeline.

### Deploy AKS and install Helm

```bash
az group create --location centralus --name aks
az aks create -g aks -n mesh -k 1.11.4
az aks get-credentials -g aks -n mesh

#deploy helm with muy fancy one-liner
kubectl create serviceaccount -n kube-system tiller; kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller; helm init --service-account tiller
```

### Install istio

```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.3
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
kubectl apply -f install/kubernetes/istio-demo-auth.yaml

# Enable automatic sidecar injector
kubectl label namespace default istio-injection=enabled

# Our pods should be in the Running/Completed state
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

### Deploy all pods at once

```bash
cd manual_deploy
kubectl apply -f .
```
### Verify web app is running

```bash
kubectl get svc -n istio-system
# Find the row corresponding to this:
# NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
# {...}
# istio-ingressgateway     LoadBalancer   10.0.134.104   168.61.161.70   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:30482/TCP,8060:30740/TCP,853:31204/TCP,15030:31704/TCP,15031:31097/TCP   38m
# {...}
# and paste the EXTERNAL_IP in your browser to load the web app's dashboard. 
# By default it shows a 50/50 Blue/green deployment

```

### Modify blue/green traffic routing

```yaml
# Modify the weights at the end of smackapi-vs.yaml
# ...
  http:
  - route:
    - destination:
        host: smackapi
        subset: blue
      weight: 100 # max out blue
    - destination:
        host: smackapi
        subset: green
      weight: 0 # zero out green
```

Now apply the updated template:

```bash
kubectl apply -f smackapi-vs.yaml
```

Reload the web app to see all blue nodes.

### Build an Azure DevOps Build definition

Create a project and a build pipeline connected to Github and point it to `azure-pipelines.yml`


## Optional

### Install istioctl on mac

```bash
brew tap ams0/istioctl
brew install istioctl
```

### Add a DNS entry for `istio-ingressgateway`

```bash
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Returns your IP
az network dns record-set a add-record -g dns -z <YOUR_FQDN> -n *.mesh --value <IP>
```

### Build and push the web frontend

```bash
cd app/smackweb
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o smackweb
docker build -t ams0/smackweb:0.4 --build-arg BUILD_DATE=`date -u +"%Y-%m-%dT%H%M%SZ"` --build-arg VCS_REF=`git rev-parse --short HEAD` --build-arg IMAGE_TAG_REF=0.4 .
docker push ams0/smackweb:0.4
```


### Demo flow

- [ ] Deploy AKS
- [ ] Install Helm
- [ ] Install Istio
- [ ] Deploy app
- [ ] Watch Azure DevOps
- [ ] Watch Istio with [Grafana](http://127.0.0.1:8001/api/v1/namespaces/istio-system/services/grafana:3000/proxy/d/UbsSZTDik/istio-workload-dashboard?refresh=10s&orgId=1&var-namespace=default&var-workload=smackapi&var-srcns=All&var-srcwl=All&var-dstsvc=All)



### Useful links

[Smackapp source](https://github.com/chzbrgr71/microsmackv2)
[Azure trials](aka.ms/aztrialsuk)
[Isti](http://istio.sh)o
