# Access Control

This use case applies access control policies to deny and allow traffic from a specific subnet

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
cd ~/NGINX-Ingress-Controller-Lab/labs/5.access-control
```

Deploy the sample web applications
```code
kubectl apply -f 0.webapp.yaml
```

Deploy an access control policy that denies requests from clients with an IP that belongs to the subnet 10.0.0.0/8
```code
kubectl apply -f 1.access-control-policy-deny.yaml
```

Publish the application through NGINX Ingress Controller applying the access control policy
```code
kubectl apply -f 2.virtual-server.yaml
```

Check the newly created `VirtualServer` resource
```code
kubectl get vs -o wide
```

Output should be similar to
```code
NAME     STATE   HOST                 IP    EXTERNALHOSTNAME   PORTS   AGE
webapp   Valid   webapp.example.com                                    4s
```

Describe the `webapp` virtualserver
```code
kubectl describe vs webapp
```

Output should be similar to
```
Name:         webapp
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.nginx.org/v1
Kind:         VirtualServer
Metadata:
  Creation Timestamp:  2025-04-03T20:44:26Z
  Generation:          1
  Resource Version:    248472
  UID:                 06882d7b-5ec7-4fe8-b272-7052868aa9d6
Spec:
  Host:  webapp.example.com
  Policies:
    Name:  webapp-policy
  Routes:
    Action:
      Pass:  webapp
    Path:    /
  Upstreams:
    Name:     webapp
    Port:     80
    Service:  webapp-svc
Status:
  Message:  Configuration for default/webapp was added or updated 
  Reason:   AddedOrUpdated
  State:    Valid
Events:
  Type    Reason          Age   From                      Message
  ----    ------          ----  ----                      -------
  Normal  AddedOrUpdated  23s   nginx-ingress-controller  Configuration for default/webapp was added or updated
```

Access the application
```code
curl -i -H "Host: webapp.example.com" http://$NIC_IP:$HTTP_PORT
```

NGINX Ingress controller blocks the request if the client IP belongs to subnet `10.0.0.0/8`
```
HTTP/1.1 403 Forbidden
Server: nginx/1.27.2
Date: Thu, 03 Apr 2025 20:45:33 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.27.2</center>
</body>
</html>
```

Update the access control policy
```code
kubectl apply -f 3.access-control-policy-allow.yaml
```

Access the application
```code
curl -i -H "Host: webapp.example.com" http://$NIC_IP:$HTTP_PORT
```

NGINX Ingress controller allows traffic from subnet `10.0.0.0/8`
```
HTTP/1.1 200 OK
Server: nginx/1.27.2
Date: Thu, 03 Apr 2025 20:46:29 GMT
Content-Type: text/plain
Content-Length: 157
Connection: keep-alive
Expires: Thu, 03 Apr 2025 20:46:28 GMT
Cache-Control: no-cache

Server address: 192.168.36.99:8080
Server name: webapp-6db59b8dcc-nchgr
Date: 03/Apr/2025:20:46:29 +0000
URI: /
Request ID: db6e3caf0b45c7e364f01961b40a3dd9
```

Delete the lab

```code
kubectl delete -f .
```
