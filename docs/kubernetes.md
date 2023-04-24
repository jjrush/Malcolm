# <a name="Kubernetes"></a>Deploying Malcolm with Kubernetes

* [Deploying Malcolm with Kubernetes](#Kubernetes)
    - [System](#System)
        + [Ingress Controller](#Ingress)
    - [Configuration](#Config)
        + [PersistentVolumeClaim definitions](#PVC)
* [Future Enhancements](#Future)
    - [Live Traffic Analysis](#FutureLiveCap)
    - [Horizontal Scaling](#FutureScaleOut)
    - [Helm Chart](#FutureHelmChart)

## <a name="System"></a> System

### <a name="Ingress"></a> Ingress Controller

Malcolm's [ingress controller manifest]({{ site.github.repository_url }}/blob/{{ site.github.build_revision }}/kubernetes/00-ingress.yml) uses the [Ingress-NGINX controller for Kubernetes](https://github.com/kubernetes/ingress-nginx). A few Malcolm features require some customization when installing and configuring the Ingress-NGINX controller:

* To [forward](malcolm-hedgehog-e2e-iso-install.md#HedgehogConfigForwarding) logs from a remote instance of [Hedgehog Linux](hedgehog.md):
    - See ["Exposing TCP and UDP services"](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/) in the Ingress-NGINX documentation.
    - You must configure the controller to start up with the `--tcp-services-configmap=ingress-nginx/tcp-services` flag:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
…
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
…
  template:
…
    spec:
      containers:
        * args:
            * /nginx-ingress-controller
            * --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
            * --election-id=ingress-nginx-leader
            * --controller-class=k8s.io/ingress-nginx
            * --ingress-class=nginx
            * --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
            * --validating-webhook=:8443
            * --validating-webhook-certificate=/usr/local/certificates/cert
            * --validating-webhook-key=/usr/local/certificates/key
            * --enable-ssl-passthrough
            * --tcp-services-configmap=ingress-nginx/tcp-services
…
```
    - You must add the appropriate ports (minimally TCP ports 5044 and 9200) to the `ingress-nginx-controller` load-balancer service definition:
```yml
---
apiVersion: v1
kind: Service
metadata:
…
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  ipFamilies:
    + IPv4
  ipFamilyPolicy: SingleStack
  ports:
    + appProtocol: http
      name: http
      port: 80
      protocol: TCP
      targetPort: http
    + appProtocol: https
      name: https
      port: 443
      protocol: TCP
      targetPort: https
    + appProtocol: tcp
      name: lumberjack
      port: 5044
      targetPort: 5044
      protocol: TCP
    + appProtocol: tcp
      name: tcpjson
      port: 5045
      targetPort: 5045
      protocol: TCP
+ appProtocol: tcp
      name: opensearch
      port: 9200
      targetPort: 9200
      protocol: TCP
…
  type: LoadBalancer
```
    - You must add the appropriate ports (minimally TCP ports 5044 and 9200) to the `ingress-nginx-controller` deployment container's definition:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
…
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
…
  template:
…
    spec:
      containers:
…
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
            - containerPort: 443
              name: https
              protocol: TCP
            - containerPort: 8443
              name: webhook
              protocol: TCP
            - name: lumberjack
              containerPort: 5044
              protocol: TCP
            - name: tcpjson
              containerPort: 5045
              protocol: TCP
            - name: opensearch
              containerPort: 9200
              protocol: TCP
…
```
* To use [SSL Passthrough](https://kubernetes.github.io/ingress-nginx/user-guide/tls/) to have the Kubernetes gateway use Malcolm's TLS certificates rather than its own:
    - You must configure the controller to start up with the `--enable-ssl-passthrough` flag.
```
apiVersion: apps/v1
kind: Deployment
metadata:
…
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
…
  template:
…
    spec:
      containers:
        - args:
            - /nginx-ingress-controller
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
            - --election-id=ingress-nginx-leader
            - --controller-class=k8s.io/ingress-nginx
            - --ingress-class=nginx
            - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
            - --validating-webhook=:8443
            - --validating-webhook-certificate=/usr/local/certificates/cert
            - --validating-webhook-key=/usr/local/certificates/key
            - --enable-ssl-passthrough
            - --tcp-services-configmap=ingress-nginx/tcp-services
…
```
    - You must modify Malcolm's [ingress controller manifest]({{ site.github.repository_url }}/blob/{{ site.github.build_revision }}/kubernetes/00-ingress.yml) to specify the `host:` value and use [host-based routing](https://kubernetes.github.io/ingress-nginx/user-guide/basic-usage/):
```
…
spec:
  rules:
  - host: malcolm.example.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-proxy
            port:
              number: 443
…
```

### <a name="Limits"></a> System Limits

## <a name="Config"></a> Configuration

The steps to configure and tune Malcolm for a Kubernetes deployment are [very similar](malcolm-config.md#ConfigAndTuning) to those for a Docker-based deployment. Both methods use [environment variable files](malcolm-config.md#MalcolmConfigEnvVars) for Malcolm's runtime configuration.

Malcolm's configuration and runtime scripts (e.g., `./scripts/configure`, `./scripts/auth_setup`, `./scripts/start`, etc.) are used for both Docker- and Kubernetes-based deployments. To indicate to these scripts that you're working with Kubernetes rather than `docker-compose`, provide the script with the [kubeconfig file](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) used to communicate with the API server of the Kubernetes cluster (e.g., `./scripts/configure -f k3s.yaml` or `./scripts/start -f kubeconfig.yaml`, etc.). The scripts will detect whether YAML file specified is a kubeconfig file or a Docker compose file and act accordingly

### <a name="PVC"></a> PersistentVolumeClaim definitions

## <a name="Running"></a> Running Malcolm

# <a name="Future"></a> Future Enhancements

Deploying Malcolm with Kubernetes is a new (and still somewhat experimental) feature, and does not yet support the full range of Malcolm features. Development around these features is [ongoing](https://github.com/idaholab/Malcolm/issues?q=is%3Aissue+is%3Aopen+kubernetes). Some of the notable features that are still a work in progress for Kubernetes deployment include:

## <a name="FutureLiveCap"></a> Live Traffic Analysis

For now, network traffic artifacts for analysis are provided to a Malcolm deployment on Kubernetes via [forwarding](malcolm-hedgehog-e2e-iso-install.md#HedgehogConfigForwarding) from a remote instance of [Hedgehog Linux](hedgehog.md) or via PCAP [upload](upload.md#Upload). [Future work](https://github.com/idaholab/Malcolm/issues/175) is needed to design and implement monitoring of network traffic in the cloud.

## <a name="FutureScaleOut"></a> Horizontal Scaling

For now, the Malcolm services running in Kubernetes are configured with `replicas: 1`. There is [more investigation and development](https://github.com/idaholab/Malcolm/issues/182) needed to ensure Malcolm's containers work correctly when horizontally scaled.

## <a name="FutureHelmChart"></a> Helm Chart

For now, Malcolm's Kubernetes deployment is managed via vanilla [Kubernetes manifests]({{ site.github.repository_url }}/blob/{{ site.github.build_revision }}/kubernetes/). We need to [look into](https://github.com/idaholab/Malcolm/issues/187) what a Malcolm Helm chart would look like and how it would fit in with the [deployment scripts](https://github.com/idaholab/Malcolm/issues/172) for [configuring](#Config) and [running](#Running) Malcolm, if at all.