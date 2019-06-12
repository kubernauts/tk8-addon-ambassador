# TK8 addon - Ambassador

## What are TK8 addons?

- TK8 add-ons provide freedom of choice for the user to deploy tools and applications without being tied to any customized formats of deployment.
- Simplified deployment process via CLI (will also be available via TK8 web in future).
- With the TK8 add-ons platform, you can also build your own add-ons.

To get more support join us on [Slack](https://kubernauts-slack-join.herokuapp.com)

### What is Ambassador

Ambassador is an open source, Kubernetes-native [microservices API gateway](https://www.getambassador.io/about/microservices-api-gateways/) built on the [Envoy Proxy](https://www.envoyproxy.io/). Ambassador is built from the ground up to support multiple, independent teams that need to rapidly publish, monitor, and update services for end users. Ambassador can also be used to handle the functions of a Kubernetes ingress controller and load balancer.

Ambassador is:

- **Self-service.** Ambassador is designed so that developers can manage services directly. This requires a system that is not only easy for developers to use but provides safety and protection against inadvertent operational issues.
- **Operations friendly.** Ambassador has virtually no moving parts, and delegates all routing and resilience to Envoy Proxy and Kubernetes, respectively. Ambassador stores all state in Kubernetes (no database!). Multiple Ambassadors can be run in the same cluster, making upgrades easy and seamless.
- **Designed for microservices.** Ambassador integrates the features teams need for microservices, including authentication, rate limiting, observability, routing, TLS termination, and more.

## Prerequisites 

RBAC must be enabled on the Kubernetes Cluster.

## Get started

You can install Ambassador on the Kubernetes cluster via TK8 addons functionality.

What do you need:
- tk8 binary
- A Kubernetes cluster that supports Service objects of type: LoadBalancer

## Deploy Ambassador on your Kubernetes Cluster

Run

    $ tk8 addon install ambassador
    search local for ambassador
    check if provided a url
    Search addon on kubernauts space.
    Cloning into 'ambassador'...
    Install ambassador
    execute main.sh
    Creating main.yaml
    add  ./ambassador-config/ambassador-rbac.yaml
    add  ./ambassador-config/ambassador-svc.yaml
    apply ambassador/main.yml
    service/ambassador-admin created
    clusterrole.rbac.authorization.k8s.io/ambassador created
    serviceaccount/ambassador created
    clusterrolebinding.rbac.authorization.k8s.io/ambassador created
    deployment.extensions/ambassador created
    service/ambassador created
    ambassador installation complete

This command will clone the https://github.com/kubernauts/tk8-addon-ambassador repository locally and setup Ambassador on the cluster.

## Create your first route

Create a YAML file httpbin.yaml with the below contents:

    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: httpbin
    annotations:
        getambassador.io/config: |
        ---
        apiVersion: ambassador/v1
        kind:  Mapping
        name:  httpbin_mapping
        prefix: /httpbin/
        service: httpbin.org:80
        host_rewrite: httpbin.org
    spec:
      ports:
      - name: httpbin
        port: 80
        
 A few things to notice here:

- getambassador.io/config annotation is how Ambassador identifies the service it needs to route. Once it sees this annotation, it uses the Mapping contained in it to configure the route.
- The mapping creates a route that will route traffic from the /httpbin/ endpoint to the public httpbin.org service. 

Create the service by running:

    $ kubectl create -f httpbin.yaml
    service/httpbin created

## Test the mapping

Note down the external IP address of the Ambassador by running:

    $ kubectl get svc ambassador
    NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
    ambassador   LoadBalancer   10.43.72.163   84.200.100.228   80:32295/TCP   22m
    
Now, open the browser and navigate to 84.200.100.228/httpbin/ and you should see the httpbin service page.  

## Adding a service to the deployment

We'll take a look at an example of exposing a deployment with the ambassador routes.

Create a file qotm.yaml with the following contents:

```
---
apiVersion: v1
kind: Service
metadata:
  name: qotm
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v1
      kind:  Mapping
      name:  qotm_mapping
      prefix: /qotm/
      service: qotm
spec:
  selector:
    app: qotm
  ports:
  - port: 80
    name: http-qotm
    targetPort: http-api
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: qotm
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: qotm
    spec:
      containers:
      - name: qotm
        image: datawire/qotm:1.2
        ports:
        - name: http-api
          containerPort: 5000
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 3
        resources:
          limits:
            cpu: "0.1"
            memory: 100Mi
```
Create deployment and service by running

```
$ kubectl apply -f qotm.yaml
service/qotm created
deployment.extensions/qotm created
```

Now, open the URL EXTERNAL-IP/qotm/ in your browser and you should see a message similar to this:

```
{"hostname":"qotm-7c6cccd985-7j46x","ok":true,"quote":"A principal idea is omnipresent, much like candy.","time":"2019-03-13T11:34:35.308838","version":"1.3"}
```

## Ambassador diagnostic service

Ambassador includes an integrated diagnostics service to help with troubleshooting. By default, this is not exposed to the Internet. To view it, we'll need to do port-forwarding with one of the Ambassador pods:

```
$ kubectl get pods
NAME                                                              READY   STATUS      RESTARTS   AGE
ambassador-76f644ddfb-drp85                                       1/1     Running     0          34m
ambassador-76f644ddfb-k2sn2                                       1/1     Running     0          34m
ambassador-76f644ddfb-wnlbt                                       1/1     Running     0          34m
 
$ kubectl port-forward ambassador-76f644ddfb-drp85 8877
Forwarding from 127.0.0.1:8877 -> 8877
Forwarding from [::1]:8877 -> 8877
Handling connection for 8877
Handling connection for 8877
```

Now, open http://localhost:8877/ambassador/v0/diag/ in your browser and you should see a bunch of information regarding Ambassador, routes, debug mode, etc.

## Uninstall Ambassador

For removing Ambassador from your cluster, we can use TK8 addon's **destroy** functionality. Run:
```
$ tk8 addon destroy ambassador
Search local for ambassador
Addon ambassador already exist
Found ambassador local.
Destroying ambassador
execute main.sh
Creating main.yaml
add  ./ambassador-config/ambassador-rbac.yaml
add  ./ambassador-config/ambassador-svc.yaml
delete ambassador from cluster
service "ambassador-admin" deleted
clusterrole.rbac.authorization.k8s.io "ambassador" deleted
serviceaccount "ambassador" deleted
clusterrolebinding.rbac.authorization.k8s.io "ambassador" deleted
deployment.extensions "ambassador" deleted
service "ambassador" deleted
ambassador destroy complete
```
