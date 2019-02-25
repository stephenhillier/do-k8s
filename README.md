# Kubernetes / Istio / cert-manager on DigitalOcean

This walkthrough documents the steps to setting up a sandbox environment on the DigitalOcean Kubernetes service with a few additional services like Istio and cert-manager.

## Other resources
* If you just want to try Kubernetes, check out https://katacoda.com/courses/kubernetes
* kubectl cheat sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
* kubectl practice exercises: https://github.com/dgkanatsios/CKAD-exercises
* Istio service mesh tutorial (using Google Cloud): https://github.com/stefanprodan/istio-gke.  *if you have Google Cloud credits remaining or haven't claimed them yet, check out this tutorial instead*. Some of the steps here are adapted from this guide, modified to work with DigitalOcean.

## Prerequisites

* `kubectl`
* `helm` client (https://helm.sh/docs/using_helm/)
* a DigitalOcean account
* a domain name that uses DigitalOcean nameservers (optional) (note: nameserver can be Route53 or CloudDNS with modified configuration - see cert-manager docs)

## Create a cluster and set up kubectl

Create a cluster with a node pool having at least 4 vCPU / 4 GB RAM (e.g. 3 * `2vCPU/2GB` will leave room for some of your own applications).  Warning: you will be charged for these resources (about $1.75 per day).  You can destroy the cluster and the load balancer when you're done.

Download the `kubeconfig` file (with the "Download Config" button) and copy it to `$HOME/.kube/config`. This will set this cluster as the default context for `kubectl`.

```sh
cp ~/Downloads/mycluster-kubeconfig.yaml ~/.kube/config
```
Check that our client can connect to the cluster:
```sh
$ kubectl get nodes
NAME                   STATUS   ROLES    AGE   VERSION
inspiring-kilby-uwd5   Ready    <none>   11m   v1.13.2
inspiring-kilby-uwdp   Ready    <none>   11m   v1.13.2
inspiring-kilby-uwds   Ready    <none>   12m   v1.13.2
```

## Install Istio

Download Istio (instructions from https://istio.io/docs/setup/kubernetes/download-release/):

```sh
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.6
```

Apply custom resource definitions (CRDs) for certmanager:
```sh
kubectl apply -f install/kubernetes/helm/istio/charts/certmanager/templates/crds.yaml
```

Create Tiller service account:
```sh
kubectl apply -f ./install/kubernetes/helm/helm-service-account.yaml
```

Install Tiller:
```sh
helm init --service-account tiller
```

create an `istio.yaml` file that configures Istio with Grafana, Jaegar, Kiali, and certmanager (adapted from the guide at https://github.com/stefanprodan/istio-gke):

```yaml
global:
  nodePort: false

sidecarInjectorWebhook:
  enabled: true
  enableNamespacesByDefault: false

gateways:
  enabled: true
  istio-ingressgateway:
    replicaCount: 2
    autoscaleMin: 2
    autoscaleMax: 3
    type: LoadBalancer

pilot:
  enabled: true
  replicaCount: 1
  autoscaleMin: 1
  autoscaleMax: 1
  resources:
    requests:
      cpu: 500m
      memory: 1024Mi

grafana:
  enabled: true
  security:
    enabled: true
    adminUser: admin
    adminPassword: change_me

prometheus:
  enabled: true

kiali:
  enabled: true
  dashboard:
    username: admin
    passphrase: change_me

servicegraph:
  enabled: true

tracing:
  enabled: true
  jaeger:
    tag: 1.9

certmanager:
  enabled: true
  tag: v0.6.2
```

Save the `istio.yaml` file and install Istio:
```sh
helm upgrade --install istio ./install/kubernetes/helm/istio --namespace=istio-system -f ./istio.yaml
```

Check that the Istio pods are running (it may take a couple minutes for all pods to start):
```sh
kubectl -n istio-system get pods
```

## Configure HTTPS with cert-manager and Let's Encrypt

### HTTPS Gateway
Create an [Istio gateway](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Gateway) (again, credit to https://github.com/stefanprodan/istio-gke for this example gateway):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*"
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
```

Save this as `istio-gateway.yaml` and apply it with:

```sh
kubectl apply -f ./istio-gateway.yaml
```

### Get a wildcard domain certificate

For this example, you will need a domain name that uses [DigitalOcean for DNS](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars).  You can also use CloudDNS or Route53 if you modify these examples [using the cert-manager docs](https://docs.cert-manager.io/en/latest/).

Get an API key from the DigitalOcean dashboard (API section), and create a secret to store it in your cluster (replace $APIKEY with your token):

```sh
kubectl -n istio-system create secret generic digitalocean-dns --from-literal=access-token=$APIKEY
```

The token will be used by cert-manager to create TXT DNS entries that Let's Encrypt will use to verify ownership of your domain.

Now, create a certificate "Issuer" that has access to your token:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-issuer
  namespace: istio-system
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: youremail@example.com
    privateKeySecretRef:
      name: letsencrypt-issuer
    dns01:
      providers:
      - name: do-dns
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns
            key: access-token
```

Save as `issuer.yaml` and apply:

```sh
kubectl apply -f issuer.yaml
```

Create a Certificate resource (replace example.com with your domain name):
```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: istio-gateway
  namespace: istio-system
spec:
  secretname: istio-ingressgateway-certs
  issuerRef:
    name: letsencrypt-issuer
    kind: Issuer
  commonName: "*.example.com"
  dnsNames:
  - example.com
  acme:
    config:
    - dns01:
        provider: do-dns
      domains:
      - "*.example.com"
      - example.com
```

Save as certificate.yaml and apply:

```sh
kubectl apply -f certificate.yaml
```

cert-manager will attempt to grab a certificate from Let's Encrypt, enabling you to serve any subdomain of your domain name using HTTPS. This process may take some time. Check logs with:

```sh
kubectl -n istio-system logs deployment/certmanager -f
```

When the certificate is successfully obtained, delete the ingress pods to reload the certificate:
```sh
kubectl -n istio-system delete pods -l istio=ingressgateway
```
