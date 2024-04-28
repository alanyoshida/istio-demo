# Istio demo

Demonstration for the talk "What service mesh has to do with security ?"

## Create Kind Cluster

```bash
# Create cluster
kind create cluster --name istio-demo

# Show clusters
kind get clusters
```

## Install metallb

You will need the docker network cidr, to configure metallb. This is needed to create the istio ingress service load balancer


```bash
# Get docker cidr
docker network inspect -f '{{.IPAM.Config}}' kind
```

Edit charts/metallb/values.yaml
```yaml
  addresses:
  - 172.19.255.200-172.19.255.250 # <---- CHANGE THIS IP POOL ADDRESS
```

## Kubernetes Dashboard (optional)

```bash
token=$(kubectl -n kubernetes-dashboard create token admin-user)
kubectl proxy
```

Access The dashboard in and use the token you obtained from previous commands:

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


## Install istioctl

Download the istioctl
`curl -L https://istio.io/downloadIstio | sh -`


```bash
# Install istio on cluster
istioctl install --set profile=demo -y

# Enable sidecar injection on default namespace
kubectl label namespace default istio-injection=enabled

# Test a request from pod ratings to productpage
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# Show more information from istio
istioctl analyze
```

## Observability

```bash
istioctl dashboard kiali

istioctl dashboard grafana

istioctl dashboard jaeger
```

## Deploy Apps

Tilt will provision almost everything you need in the cluster with this simple command

```bash
tilt up
```

## Debug

```bash
# Show routes
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system

# Individual route
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system --name http.8080 -o json

istioctl proxy-config routes deploy/istio-ingressgateway.istio-system --name https.443.https.bookinfo-gateway.default -o json

```

## Accessing from outside the cluster

Get the external-ip from the service

```bash
$ kubectl -n istio-system get svc istio-ingressgateway
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.96.185.168   172.18.255.200   15021:31481/TCP,80:32382/TCP,443:31965/TCP,31400:31904/TCP,15443:31016/TCP   7d4h
```

Configure the following domains with the external ip in your /etc/hosts

```
172.18.255.200 grafana.istio.local
172.18.255.200 kiali.istio.local
172.18.255.200 jaeger.istio.local
172.18.255.200 site.bookinfo.local
172.18.255.200 api.httpbin.local
```

## Secure inbound traffic with SSL

```bash
#Interactive
openssl req -x509 -newkey rsa:4096 -keyout istio.key -out istio.crt -sha256 -days 365

# With Params
openssl req -x509 -newkey rsa:4096 -keyout istio.key -out istio.crt -sha256 -days 3650 -nodes -subj "/C=BR/ST=SP/L=Sao Paulo/O=alanyoshida/OU=security/CN=*.istio.local"

# With less params
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout istio.key -out istio.crt -subj "/CN=*.istio.local"

# Create tls secret
kubectl create -n istio-system secret tls istio-cert --key istio.key --cert istio.crt

# Test istio.local
curl --cacert ./istio.crt https://grafana.istio.local -v

# For bookinfo
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout bookinfo.key -out bookinfo.crt -subj "/CN=*.bookinfo.local"

# Create tls secret
kubectl create secret tls bookinfo-cert --key bookinfo.key --cert bookinfo.crt

# Test bookinfo.local
curl --cacert ./bookinfo.crt https://site.bookinfo.local/productpage -v

```

## Securing Communication Within Istio

```bash
kubectl get peerauthentication --all-namespaces
```


## Enable strict mTLS

You can lock down the secure access to all services in your Istio service mesh to require mTLS using a peer authentication policy.

```bash
kubectl apply -n istio-system -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

```bash
kubectl exec deploy/sleep -n default -- curl http://api.httpbin.local/

kubectl exec deploy/sleep -n no-istio -- curl http://site.bookinfo.local/productpage -v

# Certs used by istio for productpage
istioctl proxy-config secret deploy/productpage-v1

# The issuer
istioctl proxy-config secret deploy/productpage-v1 -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Issuer' -A 1

# Check if is valid
istioctl proxy-config secret deploy/productpage-v1 -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Validity' -A 2

# The identity of the client certificate
istioctl proxy-config secret deploy/productpage-v1 -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Subject Alternative Name' -A 1

# The trust domain come from istio
kubectl get cm istio -n istio-system -o yaml | grep trustDomain -m 1
```


## Authorization Policy

With the following policy:
```yaml
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/default/sa/sleep"]
   to:
   - operation:
       methods: ["GET"]
```

```bash
# Get namespace labels
kubectl get ns --show-labels
# istio-injection=enabled

# Should fail, because does not have istio sidecar, and mtls is STRICT
kubectl exec deploy/sleep -n no-istio -- curl http://httpbin.default.svc.cluster.local:8000 -v | head

# Should work
kubectl exec deploy/sleep -n default -- curl http://httpbin.default.svc.cluster.local:8000 -v | head

# Shoud not work, because you dont have Authorization Policy
kubectl exec deploy/sleep -n no-permission -- curl http://httpbin.default.svc.cluster.local:8000 -v | head
# RBAC: access denied
```
