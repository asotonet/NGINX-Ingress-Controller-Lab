# Lab deployment

See the [prerequisites](/README.md#getting-started)

## Installing

1. Create NGINX Ingress Controller namespace

```code
kubectl create namespace nginx-ingress
```

2. Create Kubernetes secret to pull images from NGINX private registry

```code
kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username=`cat <nginx-one-eval.jwt>` --docker-password=none -n nginx-ingress
```

Note: `<nginx-one-eval.jwt>` is the path and filename of your `nginx-one-eval.jwt` file

3. Create Kubernetes secret holding the NGINX Plus license

```code
kubectl create secret generic license-token --from-file=license.jwt=<nginx-one-eval.jwt> --type=nginx.com/license -n nginx-ingress
```

Note: `<nginx-one-eval.jwt>` is the path and filename of your `nginx-one-eval.jwt` file

4. List available NGINX Ingress Controller docker images that include NGINX App Protect WAF

```code
curl -s https://private-registry.nginx.com/v2/nginx-ic-nap/nginx-plus-ingress/tags/list --key <nginx-one-eval.key> --cert <nginx-one-eval.crt> | jq
```

Note: `<nginx-one-eval.key>` and `<nginx-one-eval.key>` are the path and filename of your `nginx-one-eval.crt` and `nginx-one-eval.crt` files respectively

Pick the latest `5.x` version (`5.0.0` at the time of writing)

5. Apply NGINX Ingress Controller custom resources (make sure the URI below references the latest available `4.x` NGINX Ingress Controller version)

```code
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.0.0/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.0.0/deploy/crds-nap-waf.yaml
```

6. Install NGINX Ingress Controller with NGINX App Protect through its Helm chart (set `nginx.image.tag` to the latest `4.x` available NGINX Ingress Controller version)

```code
helm install nic oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.1.0 \
  --set controller.image.repository=private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress \
  --set controller.image.tag=5.0.0 \
  --set controller.nginxplus=true \
  --set controller.appprotect.enable=true \
  --set controller.serviceAccount.imagePullSecretName=regcred \
  --set controller.mgmt.licenseTokenSecretName=license-token \
  --set controller.service.type=NodePort \
  -n nginx-ingress
```

7. Check NGINX Ingress Controller pod status

```code
kubectl get pods -n nginx-ingress
```

Pod should be in the `Running` state

```code
NAME                                            READY   STATUS    RESTARTS   AGE
nic-nginx-ingress-controller-77955c859b-4sdtg   1/1     Running   0          31s
```

8. Check NGINX Ingress Controller logs

```code
kubectl logs -l app.kubernetes.io/instance=nic -n nginx-ingress -c nginx-ingress
```

Output should be similar to

```code
2025/04/03 21:01:38 [notice] 21#21: signal 29 (SIGIO) received
2025/04/03 21:01:38 [notice] 21#21: signal 17 (SIGCHLD) received from 24
2025/04/03 21:01:38 [notice] 21#21: worker process 24 exited with code 0
2025/04/03 21:01:38 [notice] 21#21: signal 29 (SIGIO) received
2025/04/03 21:01:38 [notice] 21#21: signal 17 (SIGCHLD) received from 30
2025/04/03 21:01:38 [notice] 21#21: worker process 30 exited with code 0
2025/04/03 21:01:38 [notice] 21#21: signal 29 (SIGIO) received
BD_MISC|NOTICE|Apr 03 21:01:39.620|0042|/builds/t1_j2fGn9/14/waf/waf-general/secore/bd/bd/temp_func.c:2737|UMU: 0 0 || 0 0 0 0 0 0 0 0 0 0 0 0 || 0 0 0 0 0 0 
BD_MISC|NOTICE|Apr 03 21:01:39.620|0042|/builds/t1_j2fGn9/14/waf/waf-general/secore/bd/bd/temp_func.c:2738|UMU: total     0 (  0Kb) VM (1475M) RSS (244M) SWAP (  0M) trans     0
BD_MISC|NOTICE|Apr 03 21:01:41.602|0040|/builds/t1_j2fGn9/14/waf/waf-general/secore/common/bd_tbls/LoggingAccount.cpp:4582|configuration changed for remote logger crc 1492155653, state 1, thread type 0
```

9. Check Kubernetes service status

```code
kubectl get svc -n nginx-ingress
```

NGINX Ingress Controller should be listening on TCP ports 80 and 443

```code
NAME                           TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
nic-nginx-ingress-controller   NodePort   10.101.129.241   <none>        80:31282/TCP,443:30332/TCP   42s
```

10. Check the `ingressclass`

```code
kubectl get ingressclass
```

The `nginx` ingressclass should be available

```code
NAME    CONTROLLER                     PARAMETERS   AGE
nginx   nginx.org/ingress-controller   <none>       70s
```

## Uninstalling

* Uninstall NGINX Ingress Controller through its Helm chart

```code
helm uninstall nic -n nginx-ingress
```

* Delete the namespace

```code
kubectl delete namespace nginx-ingress
```
