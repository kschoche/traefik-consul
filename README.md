# traefik-consul

This is a basic installation of Consul and Traefik which uses Consul to transparently proxy traffic from the Ingress GW to the backend services using mTLS. It uses the simple `whoami` demo from traefik as an example backend service that we will curl through the ingress gw.

~NOTE:  Until we GA 1.10 bits, use main branch instead

* Install Consul: `helm install consul hashicorp/consul --version=0.26.0 consul-values.yaml`

This is a basic consul installation pointed at 1.10-beta4, and should be updated to GA version of 1.10 and consul-k8s when its available.
```yaml
global:
  imageK8S: kschoche/consul-k8s-dev
  image: hashicorp/consul:1.10.0-rc2 
  tls:
    enabled: true
  acls:
    manageSystemACLs: true
  metrics:
    enabled: true
controller:
  enabled: true
server:
  replicas: 1
connectInject:
  enabled: true
ui:
  enabled: true
```

* Once consul is online: `helm install traefik traefik/traefik traefik-values.yaml`

There are just a few modifications required to base traefik values.yaml file, primarily setting the serviceAccount name for the traefik deployment so that it matches the service name, and then setting connect exclude ports and connect inject:
```yaml
# Whether Role Based Access Control objects like roles and rolebindings should be created
rbac:
  enabled: true
  namespaced: false
  serviceAccount:
    name: "traefik"

# Annotations to add to the deployed services
deployment:
  podAnnotations:
    consul.hashicorp.com/connect-inject: "true"
    consul.hashicorp.com/transparent-proxy-exclude-inbound-ports: "8000,80,443,9000,8443" 
    consul.hashicorp.com/transparent-proxy-exclude-outbound-cidrs: "10.108.0.1/32"
```
The `consul.hashicorp.com/transparent-proxy-exclude-outbound-cidrs` annotation must match your kube api svc because Traefik needs to make kube API calls in order to manage services and endpoints:
```
kubectl get svc kubernetes
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.108.0.1   <none>        443/TCP   5d19h
```

* Once traefik is deployed and online, install the whoami-deployment: `kubectl apply -f whoami-deployment.yaml`

* Apply the ingress resource: `kubectl apply -f ingress-web.yaml` 

* Add a service default entry that enables DialedDirectly to the whoami app:
```yaml
kubectl apply -f whoami-crd.yaml
```

At this point if you try to curl the backend whoami service through the ingress gw you will be rejected due to lack of intentions:
```
% curl ${INGRESS_GW_IP}:80/ -H "Host: whoami"
Bad Gatewayx%
```
* Finally, apply the intention to allow traefik->whoami traffic to pass: `kubectl apply -f intention.yaml`
* And curl the ingress gateway with the service name passed as a Host header:

```
% curl ${INGRESS_GW_IP}80/ -H "Host: whoami"
Hostname: whoami-xxx-bbb64
IP: 127.0.0.1
IP: ::1
IP: 10.x.x.x
IP: fe80::xxxx:xxxx:xxxx:xxxx
RemoteAddr: 127.0.0.1:54952
GET / HTTP/1.1
Host: whoami
User-Agent: curl/7.64.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.x.2.1
X-Forwarded-Host: whoami
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-xxx-ts9df
X-Real-Ip: 10.x.x.1
```



