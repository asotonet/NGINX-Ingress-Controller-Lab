# Advanced Layer 7 routing

This use case shows how to perform advanced layer 7 routing based on HTTP Cookies and HTTP method for an application with
four services: `tea-post-svc`, `tea-svc`, `coffee-v1-svc` and `coffee-v2-svc`

- POST requests for `/tea` are routed `tea-post-svc`
- non-POST requests for `tea` are routed to `tea-svc`
- Requests for `/coffee` that include the cookie `version` set to `v2` are routed to `coffee-v2-svc`
- Requests for `/coffee` with no `version` coookie are routed to `coffee-v1-svc`


Get NGINX Ingress Controller Node IP, HTTP and HTTPS NodePorts
```code
export NIC_IP=`kubectl get pod -l app.kubernetes.io/instance=nic -n nginx-ingress -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc nic-nginx-ingress-controller -n nginx-ingress -o jsonpath='{.spec.ports[0].nodePort}'`
export HTTPS_PORT=`kubectl get svc nic-nginx-ingress-controller -n nginx-ingress -o jsonpath='{.spec.ports[1].nodePort}'`
```

Check NGINX Ingress Controller IP address, HTTP and HTTPS ports
```code
echo -e "NIC address: $NIC_IP\nHTTP port  : $HTTP_PORT\nHTTPS port : $HTTPS_PORT"
```

`cd` into the lab directory
```code
cd ~/NGINX-Ingress-Controller-Lab/labs/2.advanced-routing
```

Deploy two sample web applications
```code
kubectl apply -f 0.cafe.yaml
```

Check all application pods deployed
```code
kubectl get all
```

Output should be similar to
```
NAME                             READY   STATUS    RESTARTS   AGE
pod/coffee-v1-c48b96b65-pkvlw    1/1     Running   0          33s
pod/coffee-v2-685fd9bb65-m6zgv   1/1     Running   0          33s
pod/tea-596697966f-26swq         1/1     Running   0          33s
pod/tea-post-5647b8d885-9zq6f    1/1     Running   0          33s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/coffee-v1-svc   ClusterIP   172.20.122.65    <none>        80/TCP    33s
service/coffee-v2-svc   ClusterIP   172.20.195.88    <none>        80/TCP    33s
service/kubernetes      ClusterIP   172.20.0.1       <none>        443/TCP   22h
service/tea-post-svc    ClusterIP   172.20.194.126   <none>        80/TCP    33s
service/tea-svc         ClusterIP   172.20.188.11    <none>        80/TCP    33s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee-v1   1/1     1            1           33s
deployment.apps/coffee-v2   1/1     1            1           33s
deployment.apps/tea         1/1     1            1           33s
deployment.apps/tea-post    1/1     1            1           33s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-v1-c48b96b65    1         1         1       33s
replicaset.apps/coffee-v2-685fd9bb65   1         1         1       33s
replicaset.apps/tea-596697966f         1         1         1       33s
replicaset.apps/tea-post-5647b8d885    1         1         1       33s
```

Create TLS certificate and key to be used for TLS offload
```code
kubectl apply -f 1.cafe-secret.yaml
```

Publish `coffee` and `tea` through NGINX Ingress Controller using the `VirtualServer` Custom Resource Definition
```code
kubectl apply -f 2.advanced-routing.yaml
```

Check the newly created `VirtualServer` resource
```code
kubectl get vs -o wide
```

Output should be similar to
```code
NAME   STATE   HOST               IP    EXTERNALHOSTNAME   PORTS   AGE
cafe   Valid   cafe.example.com                                    3s
```

# Test application access

Access the `tea` service using a `GET` request
```code
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/tea
```

The `tea` service replies
```
Server address: 192.168.36.91:8080
Server name: tea-596697966f-7rvb7
Date: 03/Apr/2025:17:58:12 +0000
URI: /tea
Request ID: cf2cb6b008113c356eb76c6cf6d35a79
```

Access the `tea-post` service using a `POST` request
```code
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/tea -X POST
```

The `tea-post` service replies
```code
Server address: 192.168.169.157:8080
Server name: tea-post-5647b8d885-qthpp
Date: 03/Apr/2025:17:58:40 +0000
URI: /tea
Request ID: f40e6d860b0714412e18976ecd702c38
```

Access the `coffee` service sending a request with the cookie `version=v2`
```code
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/coffee --cookie "version=v2"
```

The `coffee-v2` service replies
```
Server address: 192.168.36.90:8080
Server name: coffee-v2-685fd9bb65-wrkvt
Date: 03/Apr/2025:17:59:14 +0000
URI: /coffee
Request ID: 03009ab419bb3dd24a5e23698f6c5f76
```

Access the `coffee` service sending a request without the cookie
```code
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/coffee
```

The `coffee-v1` service replies
```
Server address: 192.168.36.93:8080
Server name: coffee-v1-c48b96b65-tnj4g
Date: 03/Apr/2025:17:59:42 +0000
URI: /coffee
Request ID: c58a357c6633990bc3a64b76a2a11dc7
```

Delete the lab

```code
kubectl delete -f .
```
