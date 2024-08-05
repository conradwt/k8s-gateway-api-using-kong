# K8s Gateway API Using Kong

## Software Requirements

- Helm v3.15.2 or newer

- Minikube v1.33.1 or newer

- OrbStack v1.6.2 or newer

Note: This tutorial was updated on macOS 14.5. The below steps doesn't work with Docker Desktop v4.31.1
because it doesn't expose Linux VM IP addresses to the host OS (i.e. macOS).

## Tutorial Installation

1.  clone github repository

    ```zsh
    git clone https://github.com/conradwt/k8s-gateway-api-using-kong.git
    ```

2.  change directory

    ```zsh
    cd k8s-gateway-api-using-kong
    ```

3.  create Minikube cluster

    ```zsh
    minikube start -p gateway-api-kong
    ```

4.  install MetalLB

    ```zsh
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
    ```

5.  locate the subnet

    ```zsh
    docker network inspect gateway-api-kong | jq '.[0].IPAM.Config[0]["Subnet"]'
    ```

    The results should look something like the following:

    ```json
    "194.1.2.0/24",
    ```

    Then one can use an IP address range like the following:

    ```
    194.1.2.100-194.1.2.110
    ```

6.  update the `01-metallb-address-pool.yaml`

    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: demo-pool
      namespace: metallb-system
    spec:
      addresses:
        - 194.1.2.100-194.1.2.110
    ```

    Note: The IP range needs to be in the same range as the K8s cluster, `gateway-api-kong`.

7.  apply the address pool manifest

    ```zsh
    kubectl apply -f 01-metallb-address-pool.yaml
    ```

8.  apply Layer 2 advertisement manifest

    ```zsh
    kubectl apply -f 02-metallb-advertise.yaml
    ```

9.  apply deployment manifest

    ```zsh
    kubectl apply -f 03-nginx-deployment.yaml
    ```

10. apply service manifest

    ```zsh
    kubectl apply -f 04-nginx-service-loadbalancer.yaml
    ```

11. check that your service has an IP address

    ```zsh
    kubectl get svc nginx-service
    ```

    The results should look something like the following:

    ```text
    NAME            TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
    nginx-service   LoadBalancer   10.106.207.172   194.1.2.100   80:32000/TCP   17h
    ```

12. test connectivity to `nginx-service` endpoint via external IP address

    ```zsh
    curl 194.1.2.100
    ```

    The results should look something like the following:

    ```text
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```

13. install the Gateway API CRDs

    ```zsh
    kubectl apply -f 05-k8s-gateway-api-v1.1.0.yaml
    ```

14. create the Gateway and GatewayClass resources

    ```zsh
    kubectl apply -f 06-gateway.yaml
    ```

15. install Kong

    ```zsh
    helm repo add kong https://charts.konghq.com
    helm repo update
    ```

16. install Kong Ingress Controller and Kong Gateway

    ```zsh
    helm install kong kong/ingress -n kong --create-namespace
    ```

17. populate $PROXY_IP for future commands:

    ```zsh
    export PROXY_IP=$(kubectl get svc --namespace kong kong-gateway-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo $PROXY_IP
    ```

18. verify the proxy IP

    ```zsh
    curl -i $PROXY_IP
    ```

    The results should look something like the following:

    ```text
    HTTP/1.1 404 Not Found
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    X-Kong-Response-Latency: 0
    Server: kong/3.0.0

    {"message":"no Route matched with those values"}
    ```

19. deploy the X service

    # TODO rewrite for our defined service.

    ```zsh
    kubectl apply -f 07-sample-service.yaml
    ```

20. create HTTPRoute for our deployed service

    # TODO rewrite for our defined service.

    ```zsh
    kubectl apply -f 08-sample-httproute.yaml
    ```

21. test the routing rule

    # TODO rewrite for our defined service.

    ```zsh
    curl -i $PROXY_IP/echo
    ```

    The results should look like this:

    ```text
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=utf-8
    Content-Length: 140
    Connection: keep-alive
    Date: Fri, 21 Apr 2023 12:24:55 GMT
    X-Kong-Upstream-Latency: 0
    X-Kong-Proxy-Latency: 1
    Via: kong/3.2.2

    Welcome, you are connected to node docker-desktop.
    Running on Pod echo-7f87468b8c-tzzv6.
    In namespace default.
    With IP address 10.1.0.237.
    ...
    ```

## References

- https://docs.konghq.com/kubernetes-ingress-controller/3.2.x/get-started
