# install glooctl and set version to 1.11.x
```
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
glooctl upgrade --release=v1.11.7
```

# Install Gloo OSS 1.11.7 with Helm
```
helm repo add gloo https://storage.googleapis.com/solo-public-helm
helm repo update
helm upgrade --install gloo gloo/gloo --namespace gloo-system --create-namespace --version 1.11.7 --values=values-oss.yaml
```

# watch gloo-system
```
% kubectl get pods -n gloo-system   
NAME                             READY   STATUS    RESTARTS   AGE
svclb-gateway-proxy-cq745        2/2     Running   0          82s
gateway-proxy-7d4bbc86f8-6s5m9   1/1     Running   0          82s
gateway-proxy-7d4bbc86f8-rw44h   1/1     Running   0          82s
gloo-7c55cf77bf-qmv6q            1/1     Running   0          82s
gloo-7c55cf77bf-zgz77            1/1     Running   0          82s
gateway-59577d686-2hhnq          1/1     Running   0          82s
gateway-59577d686-jg8dx          1/1     Running   0          82s
```

# deploy httpbin backend
```
kubectl create namespace httpbin
kubectl apply -f 1-httpbin.yaml
kubectl apply -f 2-httpbin-upstream.yaml
kubectl apply -f 3a-httpbin-vs-delegate-root.yaml
kubectl apply -f 3b-httpbin-vs-delegate1.yaml
```

# check
```
% glooctl get vs 
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
| VIRTUAL SERVICE | DISPLAY NAME |          DOMAINS           | SSL  |  STATUS  | LISTENERPLUGINS | ROUTES |
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
| httpbin         |              | httpbin-local.glootest.com | none | Accepted |                 | / ->   |
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
```

From here I should be able to access my application at `http://httpbin-local.glootest.com` through the browser

or with curl
```
% curl -H "Host: httpbin-local.glootest.com" httpbin-local.glootest.com/get
{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin-local.glootest.com", 
    "User-Agent": "curl/7.64.1", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000"
  }, 
  "origin": "10.42.0.12", 
  "url": "http://httpbin-local.glootest.com/get"
}
```

# uninstall gloo edge OSS
```
helm uninstall gloo --namespace gloo-system
```

# check
```
% k get pods -n gloo-system
No resources found in gloo-system namespace.
```

# Apply Latest Gloo Edge EE CRDs
```
helm pull glooe/gloo-ee --version 1.11.6 --untar
kubectl apply -f gloo-ee/charts/gloo/crds
kubectl apply -f gloo-ee/charts/gloo-fed/crds
```

# install Gloo Edge Enterprise 1.11.6 (Uses OSS v1.11.7)
```
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
helm upgrade --install gloo glooe/gloo-ee --namespace gloo-system --create-namespace --version 1.11.6 --set license_key=$LICENSE_KEY --values=values-ee.yaml
```

# watch gloo-system
You should see additional Gloo Edge Enterprise pods in the `gloo-system` namespace
```
% k get pods -n gloo-system                
NAME                                                   READY   STATUS    RESTARTS        AGE
svclb-gateway-proxy-c669g                              2/2     Running   0               10m
observability-6f75b8d489-ls5x5                         1/1     Running   0               10m
gateway-proxy-6f5c6bdfc8-t6r9j                         1/1     Running   0               10m
gateway-proxy-6f5c6bdfc8-s6chj                         1/1     Running   0               10m
gloo-8699b94dc6-vc5b5                                  1/1     Running   0               10m
gloo-8699b94dc6-kjt4w                                  1/1     Running   0               10m
redis-7874d8f495-2m6cp                                 1/1     Running   0               10m
glooe-prometheus-kube-state-metrics-6bf64ccc84-4n79f   1/1     Running   0               10m
extauth-58c45fcbfc-rxlqv                               1/1     Running   0               10m
gloo-fed-console-7cd6cc68fc-867bn                      3/3     Running   0               10m
glooe-grafana-778ff75795-chdtv                         1/1     Running   0               10m
gateway-59577d686-dl69s                                1/1     Running   0               10m
gateway-59577d686-42t5h                                1/1     Running   0               10m
glooe-prometheus-server-84789c4f55-wsrvc               2/2     Running   0               10m
rate-limit-5df9b7c8d8-57jvj                            1/1     Running   2 (8m54s ago)   10m
```

# check
The VirtualService should still be there
```
% glooctl get vs 
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
| VIRTUAL SERVICE | DISPLAY NAME |          DOMAINS           | SSL  |  STATUS  | LISTENERPLUGINS | ROUTES |
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
| httpbin         |              | httpbin-local.glootest.com | none | Accepted |                 | / ->   |
+-----------------+--------------+----------------------------+------+----------+-----------------+--------+
```

From here I should be able to again access my application at `http://httpbin-local.glootest.com` or again with curl

Run `glooctl check`:
```
% glooctl version
Client: {"version":"1.11.7"}
Server: {"type":"Gateway","enterprise":true,"kubernetes":{"containers":[{"Tag":"1.11.6","Name":"observability-ee","Registry":"quay.io/solo-io"},{"Tag":"1.11.6","Name":"gloo-ee-envoy-wrapper","Registry":"quay.io/solo-io"},{"Tag":"1.11.6","Name":"gloo-ee","Registry":"quay.io/solo-io"},{"Tag":"6.2.4","Name":"redis","Registry":"docker.io"},{"Tag":"1.11.6","Name":"extauth-ee","Registry":"quay.io/solo-io"},{"Tag":"1.11.7","Name":"gateway","Registry":"quay.io/solo-io"},{"Tag":"1.11.6","Name":"rate-limit-ee","Registry":"quay.io/solo-io"}],"namespace":"gloo-system"}}
```