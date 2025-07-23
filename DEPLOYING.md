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

Pick the latest `5.x` version (`5.1.0` at the time of writing)

5. Apply NGINX Ingress Controller custom resources (make sure the URI below references the latest available `5.x` NGINX Ingress Controller version)

```code
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.1.0/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.1.0/deploy/crds-nap-waf.yaml
```

6. Install NGINX Ingress Controller with NGINX App Protect through its Helm chart (set `nginx.image.tag` to the latest `5.x` available NGINX Ingress Controller version)

```code
helm install nic oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.2.1 \
  --set controller.image.repository=private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress \
  --set controller.image.tag=5.1.0 \
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
2025/07/23 08:26:26 [notice] 20#20: exit
2025/07/23 08:26:26 [notice] 19#19: exit
I20250723 08:26:26.931031   1 main.go:109] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"nginx-ingress", Name:"nic-nginx-ingress", UID:"6e698ced-d36e-46fa-95e7-28e20ffd5748", APIVersion:"v1", ResourceVersion:"52137240", FieldPath:""}): type: 'Normal' reason: 'Updated' ConfigMap nginx-ingress/nic-nginx-ingress updated without error
I20250723 08:26:26.931060   1 main.go:109] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"nginx-ingress", Name:"nic-nginx-ingress-mgmt", UID:"b71f04cc-ca0f-454c-a3d3-fcca4e71f2ee", APIVersion:"v1", ResourceVersion:"52137239", FieldPath:""}): type: 'Normal' reason: 'Updated' MGMT ConfigMap nginx-ingress/nic-nginx-ingress-mgmt updated without error
2025/07/23 08:26:26 [notice] 15#15: signal 17 (SIGCHLD) received from 19
2025/07/23 08:26:26 [notice] 15#15: worker process 19 exited with code 0
2025/07/23 08:26:26 [notice] 15#15: signal 29 (SIGIO) received
2025/07/23 08:26:26 [notice] 15#15: signal 17 (SIGCHLD) received from 20
2025/07/23 08:26:26 [notice] 15#15: worker process 20 exited with code 0
2025/07/23 08:26:26 [notice] 15#15: signal 29 (SIGIO) received
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
