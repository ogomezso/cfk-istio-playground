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

Before start the installation my recommendation is to label the `confluent namespace` with auto sidecar injection:

~~~shell
kubectl label namespace confluent istio-injection=enabled
~~~

You just need to follow the instructions you can found on:

 [Confluent For Kubernetes Examples for External static host access Deployments with Istio](https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/networking/external-access-static-host-based)

> Note: For simplicity you can find the CR under the confluent-platform folder
> My recommendation is instead of install from a unique CR, install component by component. The recommended order can be: zookeeper, kafka, schemaregistry, connect, ksql, controlcenter

### Istio Resources

You can avoid the istio installation part form the example repo (we already installed istio with the default config)

As istio resources we will have:

#### istio-kafka-virtual-service

Istio abstraction in charge to do the wire between the istio gateway and the service.

In this case it will be used for external connections through mtls protocol 
#### istio-cp-Gateway

Istio gateway resource that will be able to route from `istio-ingressgateway` to `istio-kafka-virtual-services` 

### Clients

Create a clients namespace and apply the label for autoinjecting envoy sidecars:

~~~shell
kubectl create namespace clients
~~~~

~~~shell
kubectl label namespace clients istio-injection=enabled
~~~

#### External console producer

Generate the proper certificates and following the instructions  [Confluent For Kubernetes Examples for External static host access Deployments with Istio](https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/networking/external-access-static-host-based)

and run this command:

~~~shell
kafka-console-producer --bootstrap-server kafka.$DOMAIN:443 \
  --topic elastic-0 \
  --producer.config $TUTORIAL_HOME/client/kafka.properties
~~~

With this you will generate a console producer from your machine to the broker with mtls authentication through the `istio-ingress-gateway`

Take a look on the `kiali` console how it looks like.

#### Internal perf producer

Just apply the CRs under clients/perf-test and that will create a pod (with the proper sidecar) that creates a topic and runs a `kafka-producer-perf-test`.

As you can see on the CR that will be configured with boostrap pointing to the insecure internal bootstrap endpoint.

But as you can see on the `kiali` console the communication between this pods will be secured with mtls automatically. This happens because this actually happens between the both envoy sidecars. 