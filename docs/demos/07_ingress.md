### What is ingress?

Typically, services and pods have IPs only routable by the cluster network. All traffic that ends up at an edge router is either dropped or forwarded elsewhere. Conceptually, this might look like:
```
    internet
        |
  ------------
  [ Services ]
```
An Ingress is a collection of rules that allow inbound connections to reach the cluster services.
```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

----

It can be configured:
* to give services externally-reachable urls
* load balance traffic
* terminate SSL
* offer name based virtual hosting 

An Ingress controller is responsible for fulfilling the Ingress, usually with a loadbalancer, though it may also configure your edge router or additional frontends to help handle the traffic in an HA manner.

----

### Ingress controller

In order for the Ingress resource to work, the cluster must have an Ingress controller running

An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the ApiServer's /ingresses endpoint for updates to the Ingress resource. Its job is to satisfy requests for ingress.

Workflow:
* Poll until apiserver reports a new Ingress
* Write the LB config file based on a go text/template
* Reload LB config

----

### Example
Ingress resource
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  name: frontend-ingress
spec: 
  rules: 
    - 
      host: frontend.example.com
      http: 
        paths: 
          - 
            backend: 
              serviceName: front-end
              servicePort: 80
            path: /
```
*POSTing this to the API server will have no effect if you have not configured an Ingress controller.*

----

This section will focus on the nginx-ingress-controller. 

Specialities of the NGINX ingress controller
* The NGINX ingress controller does not uses Services to route traffic to the pods. 
* It uses the Endpoints API in order to bypass kube-proxy to allow NGINX features like:
  * session affinity 
  * custom load balancing algorithms
* It also removes some overhead, such as conntrack entries for iptables DNAT

----

### Setup

For the controller, the first thing we need to do is setup a default backend service for nginx.

The default backend is the default fall-back service if the controller cannot route a request to a service. The default backend needs to satisfy the following two requirements :
* serves a 404 page at /
* serves 200 on a /healthz

Infos about the default backend can be found [here:](https://github.com/kubernetes/contrib/tree/master/404-server)

----

### Create the default backend

Let’s use the example default backend of the official kubernetes nginx ingress project:

```
./ingress/ingress-1.sh

```

----

### Deploy the loadbalancer

```
kubectl create -f ingress/ingress-daemonset.yaml
```

This will create a nginx-ingress-controller on each available node

----

### Deploy some application

First we need to deploy some application to publishs. To keep this simple we will use the echoheaders app that just returns information about the http request as output
```
kubectl run echoheaders --image=gcr.io/google_containers/echoserver:1.4 \
  --replicas=1 --port=8080
```
Now we expose the same application in two different services (so we can create different Ingress rules)
```
kubectl expose deployment echoheaders --port=80 --target-port=8080 \
  --name=echoheaders-x
kubectl expose deployment echoheaders --port=80 --target-port=8080 \
  --name=echoheaders-y
```

----

### Create ingress rules

Next we create a couple of Ingress rules

```
kubectl create -f ingress/ingress.yaml

cat ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  name: echomap
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec: 
  rules: 
    - host: foo.bar.com
      http: 
        paths: 
          - path: /foo
            backend: 
              serviceName: echoheaders-x
              servicePort: 80           
    - host: bar.baz.com
      http: 
        paths: 
          - path: /bar
            backend: 
              serviceName: echoheaders-y
              servicePort: 80
          - path: /foo
            backend: 
              serviceName: echoheaders-x
              servicePort: 80
```

----

### Accessing the application

We can use curl or a browser, but need to send a Host header. So either edit `/etc/hosts` or send it manually

Here we'll use `curl`

```
kubectl describe nodes ingress1 | grep Address
curl -H "Host: foo.bar.com" http://<INGRESS_IP>/bar
curl -H "Host: bar.baz.com" http://<INGRESS_IP>/bar
curl -H "Host: bar.baz.com" http://<INGRESS_IP>/foo
```

----

### Enabling SSL

If you want to have SSL for our services enabled, created the needed certificates for `foo.bar.com`:

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key \
  -out tls.crt -subj "/CN=foo.bar.com"
openssl dhparam -out dhparam.pem 4096
```
No openssl available? No problem!
```
docker run --rm -v $PWD:/work -it nginx openssl req -x509 -nodes \
  -days 365 -newkey rsa:2048 -keyout /work/tls.key -out /work/tls.crt \
  -subj "/CN=foo.bar.com"
docker run --rm -v $PWD:/work -it nginx openssl dhparam \
  -out /work/dhparam.pem 4096
```

----

### Create secrets for the SSL certificates

In order to pass the cert and key to the controller we'll create secrets as follow, where tsl.key is the key name and tsl.crt is your certificate and server.pem is the pem file.
```
kubectl create secret tls foo-secret --key tls.key --cert tls.crt
kubectl create secret generic tls-dhparam --from-file=dhparam.pem 
```

----

### Create an ingress using SSL

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: foo-ssl
  namespace: default
spec:
  tls:
  - hosts:
    - foo.bar.com
    secretName: foo-secret
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: echoheaders-x
          servicePort: 80
        path: /ssl
```

```
kubectl create -f ingress/ingress-ssl.yaml
curl -H "Host: foo.bar.com" https://<INGRESS_IP>/ssl --insecure
```

----

### Whitelist

If you are using Ingress on your Kubernetes cluster it is possible to restrict access to your application based on dedicated IP addresses. 

This can be done by specifying the allowed client IP source ranges through the `ingress.kubernetes.io/whitelist-source-range` annotation. The value is a comma separated list of CIDR block, e.g. 10.0.0.0/24,1.1.1.1/32.

If you want to set a default global set of IPs this needs to be set in the config of the ingress-controller. 

----

### The configuration:
Find out your public ip: `curl ifconfig.co` and edit the ingress/
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: whitelist
annotations:
  ingress.kubernetes.io/whitelist-source-range: "<YOUR_PUBLICIP>/32"
spec:
  rules:
  - host: whitelist.test.net
  http:
    paths:
    - path: /
    backend:
      serviceName: webserver
      servicePort: 80
```

----

### Create the ingress

```
kubectl create -f ingress/ingress-whitelist.yaml
```

----

### Testing with the annotation set:

```
curl -H "Host: whitelist.test.net" http://<INGRESS_IP>/
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.11.3</center>
</body>
</html>

```
Using this simple annotation, you’re able to restrict who can access the applications in your kubernetes cluster by its IPs.

----

### Path rewrites

Sometimes, there is a need to rewrite the path of a request to match up with the backend service. 
This can be done using the `rewrite-target` annotation

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/rewrite-target: /
  name: rewrite
  namespace: default
spec:
  rules:
  - host: rewrite.bar.com
    http:
      paths:
      - backend:
          serviceName: echoheaders-y
          servicePort: 80
        path: /something
```

----

### Create the ingress config

```
kubectl create -f ingress/ingress-rewrite.yaml
```

----

### Validate the rule

```
curl -H "Host: rewrite.bar.com" http://<INGRESS_IP>/something
CLIENT VALUES:
client_address=172.16.202.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://rewrite.bar.com:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
connection=close
host=rewrite.bar.com
user-agent=curl/7.47.0
x-forwarded-for=139.59.130.207
x-forwarded-host=rewrite.bar.com
x-forwarded-port=80
x-forwarded-proto=http
x-real-ip=139.59.130.207
BODY:
```

----

### Cleanup

```
kubectl delete -f ingress/
```

----

In this section you learned how external LoadBalancing using Ingress can be done and different options of the Nginx ingress-controller.