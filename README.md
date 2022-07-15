# CFK Istio Playground

Confluent for Kubernetes on top of Istio Service Mesh deployment example.

For that we will be base on the [Confluent For Kubernetes Examples for External static host access Deployments with Istio](https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/networking/external-access-static-host-based)


And Kafka Client Deployments on a separate namespace on the cluster that will be part of the mesh.

# Istio Escosystem Install

Follow the [installation guide](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/#install-hahahugoshortcode-s2-hbhb) for istioctl (for mac the best way I founded was through brew)

We will start with a very basic Istio installation just running:

~~~shell
istioctl install
~~~

Then we will install the rest of the tools, mostly the common observability tools (prometheus, grafana, jaeger) and a Kiali as mesh dashboard.

#### Prometheus
[install](https://istio.io/latest/docs/ops/integrations/prometheus/)

#### Grafana
[install](https://istio.io/latest/docs/ops/integrations/grafana/#option-1-quick-start)

#### Jaeger
 [install](https://istio.io/latest/docs/ops/integrations/jaeger/#installation)

If you need to prevent Istio to generate too much tracing info you can apply this CR:

~~~
apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  spec:
    meshConfig:
      enableTracing: true
      defaultConfig:
        tracing:
          sampling: 50
~~~

#### Kiali
[install](https://kiali.io/docs/installation/quick-start/)

~~~shell
kubectl apply -f istio-config/
~~~

### Check Ips/port

With these commands you will get the istio ingress ips and ports

~~~bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
~~~

To get access to  our observability stack you can simply add to your `etc/hosts` file the following

~~~text
$INGRESS_HOST istio.grafana
$INGRESS_HOST istio.prometheus
$INGRESS_HOST istio.kiali
$INGRESS_HOST istio.jaeger
~~~

More detailed information on The Istio getting started sample application [documentation](https://istio.io/latest/docs/setup/getting-started/#bookinfo)
### CFK Install

You just need to follow the instructions you can found on:

 [Confluent For Kubernetes Examples for External static host access Deployments with Istio](https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/networking/external-access-static-host-based)
