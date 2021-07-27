# Let's Play with Istio

## Prerequisites

Before playing with Istio, you should first play with [Kubernetes](https://github.com/patricekrakow/play-with-kubernetes).

## Installation of Istio

1\. Download and install the latest release of the `istioctl` with `curl`:

```text
$ curl -sL https://istio.io/downloadIstioctl | sh -

Downloading istioctl-1.10.3 from https://github.com/istio/istio/releases/download/1.10.3/istioctl-1.10.3-linux-amd64.tar.gz ...
istioctl-1.10.3-linux-amd64.tar.gz download complete!

Add the istioctl to your path with:
  export PATH=$PATH:$HOME/.istioctl/bin 

Begin the Istio pre-installation check by running:
         istioctl x precheck 

Need more information? Visit https://istio.io/docs/reference/commands/istioctl/
```

2\. Add the istioctl client to your path and verify the installation:

```text
$ export PATH=$PATH:$HOME/.istioctl/bin
$ istioctl version
no running Istio pods in "istio-system"
1.10.1
```

3\. Deploy the Istio operator:

```text
$ istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.10.1
Operator controller will watch namespaces: istio-system
✔ Istio operator installed
✔ Installation complete
```

4.\ Verify the installation of the Istio operator:

```text
$ kubectl get pods -n istio-operator
NAME                             READY   STATUS    RESTARTS   AGE
istio-operator-9f7c4485c-wh7w7   1/1     Running   0          15m
```

4\. Create a file `one-istio-operator.yaml` to create an Istio operator resource instance:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: one-istio-operator
spec:
  profile: default
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
```

5\. Create the `one-istio-operator` IstioOperator from the `one-istio-operator.yaml` file:

```text
$ kubectl apply -f one-istio-operator.yaml
istiooperator.install.istio.io/one-istio-operator created
```

6\. ... Verify the installation of Istio:

```text
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-7c9df99b48-792kt   1/1     Running   0          4m37s
istiod-6d754459f7-74fbf                 1/1     Running   0          4m48s
```

## Scenario 1a

### Description of the Scenario 1a

The scenario 1a will be almost identical to the [scenario 1](https://github.com/patricekrakow/play-with-kubernetes/blob/main/README.md#description-of-the-scenario-1). But, this time, the `service-a` will not need to know the hostname of the `service-b` in which the `GET /uuid` API endpoint is implemented. The API endpoint will be defined as `GET api.company.com /uuid`, where `api.company.com` will be the hostname to be used by the HTTP client service. It is important to understand that `api.company.com` will not be resolved by the DNS server, but by the Istio service mesh, giving full control of the requests routing to the API owner/producer. On the other hand, for the API consumer, it will give a similar experience as when you consume an API from an external company. When, your HTTP client service calls an API endpoint of an external company it uses a nice hostname like `api.company.com` and you have no clue on which server your requests will be actually processed.

The HTTP client service `service-a` will be then implemented by following Bash script using `curl`

```bash
while curl http://api.company.com/uuid; do sleep 1.0; done;
```

```text
$ istioctl kube-inject -f ../play-with-kubernetes/scenario-01.yaml -o scenario-01.injected.yaml
```

```text
$ kubectl apply -f scenario-01.injected.yaml
namespace/scenario-01-ns unchanged
serviceaccount/service-b-sa unchanged
serviceaccount/service-a-sa unchanged
deployment.apps/service-b-deploy configured
service/service-b unchanged
deployment.apps/service-a-deploy configured
```

```text
$ kubectl get pods -n scenario-01-ns
NAME                                READY   STATUS    RESTARTS   AGE
service-a-deploy-7bd7b85dcd-tqx94   2/2     Running   0          49s
service-b-deploy-7fcbdb5677-kwc7s   2/2     Running   0          49s
```

```text
$ kubectl logs -n scenario-01-ns --tail 10 service-a-deploy-7bd7b85dcd-tqx94
{
  "uuid": "1baf4626-f138-4a8d-8633-dd6b7f9e8564"
}
100    53  100    53    0     0  14445      0 --:--:-- --:--:-- --:--:-- 17666
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "65f256d2-86d9-43e4-bec9-91b1e43039d0"
}
100    53  100    53    0     0   9125      0 --:--:-- --:--:-- --:--:-- 10600
```