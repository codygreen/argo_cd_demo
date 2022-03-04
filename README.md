# Argo CD Demo

## Setup

### Install NGINX Ingress Controller
For this setup, you can use NGINX Ingress or NGINX Plus Ingress

#### Install NGINX Ingress
```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm install main nginx-stable/nginx-ingress \
  --set controller.watchIngressWithoutClass=true
```

#### Install NGINX Plus Ingress
> Note: You'll need to pull the NGINX Plus Ingress Controller from your private registry

```bash
docker login
kubectl create namespace nginx-ingress
kubectl create secret -n nginx-ingress docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=${DOCKER_USERNAME} \ 
  --docker-password=${DOCKER_PAT} \ 
  --docker-email=${DOCKER_EMAIL}
helm repo add nginx-stable https://helm.nginx.com/stable
helm install nginx-plus-ingress -n nginx-ingress nginx-stable/nginx-ingress \
  --set controller.image.repository=docker.io/codygreen/nginx-plus-ingress \
  --set controller.image.tag=latest \
  --set controller.serviceAccount.imagePullSecretName=regcred \
  --set controller.nginxplus=true \
  --set controller.nginxStatus.allowCidrs=0.0.0.0/0
```
### Install Prometheus
```bash
# Install prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus
# create prometheus-server ingress
kubectl apply -f setup/prometheus-ingress.yaml
```

### Install and configure Argo CD
#### Install Argo CD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Setup port forwarding
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
> Note: this will block the terminal, so you may want to run this command in a new terminal window.  You can also do this via the Rancher Desktop control panel.

#### Get Argo CD password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
You can now login to the Argo CD dashboard using the port forwarding information and the password.

#### Configure the podinfo application
```bash
kubectl apply -f setup/argo_cd_demo.yaml
```


## Test Application

### Check podinfo application deployment.  
You should see a podinfo pod running.
```bash
kubectl get pods
```

### Check your podinfo service
You should see a podinfo service with a type of ClusterIP
```bash
kubectl get svc
```

### Get ingress service IP address
You'll need to Address of the ingress service to test the podinfo container's ingress
```bash
export INGRESSIP=$(kubectl get ingress -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}"; echo)
```

### Check your podinfo ingress
```bash
curl -H "Host: podinfo.codygreen.local" $INGRESSIP
```

You should see output like this:
```json
{
  "hostname": "podinfo-5d76864686-jd58r",
  "version": "6.0.3",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.3",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.9",
  "num_goroutine": "6",
  "num_cpu": "2"
}%
```