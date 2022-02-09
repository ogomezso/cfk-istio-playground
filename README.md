# cfk-istio-playground
Confluent for Kubernetes on top of Istio Service Mesh deployment example

# papers
https://assets.confluent.io/media/?mediaId=D2CFEAC9-AAA0-4A3D-AE2C1AAFFC6881D7
# talks
## Confluent:
Gwen Saphira:  Kafka and the Service Mesh
https://www.youtube.com/watch?v=Fi292CqOm8A
Some ideas for protocool
4 ways to integrate
Kai Waehner: Microservice Architectures with Apache Kafka, Kubernetes and Service Mesh (Envoy, Istio, Linkerd)
https://www.youtube.com/watch?v=Us_C4RFOUrA
Key Ideas:
4 Patterns:
Proxys on both sides, Confluent Components and Client ones, Control plane rules them all
Producers talks rest and the proxy will redirect to Kafka
card Confluent will no need of proxy???
card  Consumers need to know Kafka!
Producers have point to point with other services via proxy but redirects to Kafka too.
card The talk just mention it for logging but we can have to make this info available for other tools too (????)
Use Proxies to communicate with Kafka,
card Kafka itself doesn't have proxy?
## Istio Conn
https://events.istio.io/istiocon-2021/sessions/the-benefits-of-integrating-apache-kafka-with-istio-on-kubernetes/
https://www.youtube.com/watch?v=Exbs37vToZU
# blog
Kai Waiehner https://www.kai-waehner.de/blog/2019/09/24/cloud-native-apache-kafka-kubernetes-envoy-istio-linkerd-service-mesh/
https://banzaicloud.com/blog/kafka-on-istio-performance/
# Docs
## Istio
[Official Page](https://istio.io/latest/docs/)
[Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)
## Envoy
### [kafka protocol filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/kafka_broker_filter)
# Installation
## GKE
Just create a cluster and connect to kubectl
Will try `demo` profile. `istioctl install --set profile=demo`
installed on new namespace `istio-system`:
`kubectl -n istio-system get deploy`
```
					  NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
					  istio-egressgateway    1/1     1            1           2m52s
					  istio-ingressgateway   1/1     1            1           2m52s
					  istiod                 1/1     1            1           3m9s
```
Generate Manifest and verify installation:
```
					  istioctl manifest generate --set profile=demo > generated_manifest.yaml
					  istioctl verify-install generated_manifest.yaml
```
## istioctl
[install](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/#install-hahahugoshortcode-s2-hbhb)
## Prometheus
[install](https://istio.io/latest/docs/ops/integrations/prometheus/)
## Grafana
[install](https://istio.io/latest/docs/ops/integrations/grafana/#option-1-quick-start)
## Jaeger
[install](https://istio.io/latest/docs/ops/integrations/jaeger/#installation)
``` Tracing.yaml
			  apiVersion: install.istio.io/v1alpha1
			  kind: IstioOperator
			  spec:
			    meshConfig:
			      enableTracing: true
			      defaultConfig:
			        tracing:
			          sampling: 50
```
## Kiali
[install](https://kiali.io/docs/installation/quick-start/)
## Confluent for Kubernetes
Repos
Istio Operator Resources : https://github.com/confluentinc/confluent-operator/tree/master/playbooks/features/istio
Playground: https://github.com/ogomezso/cfk-istio-playground.git
### Basic Install
Place on `GKE-Demo` folder
Create a *namespace*  called *confluent*
```
					  kubectl apply -f confluent-namespace.yaml
```
Set namespace fot istio sidecar autocreation:
```
					  kubectl label namespace confluent istio-injection=enabled
```
Enable `Default Istio mutual TLS` for all confluent namespace.
```
					  kubectl apply -f istio/peer-tls.yaml
```
This enable all comms between `envoy proxies` to be secured by *mTls*
Apply `Envoy HTTP Filter`
```
					  kubectl apply -f istio/envoy-filter.yaml
```
This avoid the http2 conversion (by default in Istio) and create the correct http routes between component proxies.
Install `CP-Platform` using  the specific component CR yaml on recommended order:
```
					  kubectl apply -f components/zookeeper.yaml
					  kubectl apply -f components/broker.yaml
					  kubectl apply -f components/schemaregistry.yaml
					  kubectl apply -f components/connect.yaml
					  kubectl apply -f components/ksqldb.yaml
					  kubectl apply -f components/rest-proxy.yaml
					  kubectl apply -f components/controlcenter.yaml
```
Then apply istio `destination rules`:
```
					  kubectl apply -f istio/destination-rules.yaml
```
Rules for retry or timeouts, etc
Create Gateways:
```
					  kubectl apply -f istio/cp-gateway.yaml
```
Create external connections for http endpoints
On local testing you need to add:
```
						   istio.schemaregistry
						   istio.ksqldb
						   istio.c3
						   istio.connect
						   istio.replicator
						   istio.kafka-rest
						   istio.restproxy
```
to your `/etc/hosts` file pointing to `istio-ingress-gateway` host
```
							  export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```
# DEMO
## Bookinfo over HTTP
We will use the [Book Info Demo](https://istio.io/latest/docs/examples/bookinfo/) released with on Istio Samples
We will create a separate namespace for the app:
``` book_info_namespace.yaml
				  kind: Namespace
				  apiVersion: v1
				  metadata:
				    name: book-info
				    labels:
				      name: book-info
```
Label the namespace to automatically inject istio sidecars: 
`kubectl label namespace book-info istio-injection=enabled`
Use the same CRs of the original but adding the namespace:
```
                kubectl apply -f bookinfo-app/bookinfo-app.yaml
```
Describing any pod we can see the `istio sidecar`deployed
```
				  kubectl describe pod productpage-v1-6b746f74dc-7trfv -n book-info
```
Check
```
				  kubectl exec -n book-info "$(kubectl get pod -n book-info -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```
Set up book-info gateway
```
			  apiVersion: networking.istio.io/v1alpha3
			  kind: Gateway
			  metadata:
			    name: bookinfo-gateway
			    namespace: book-info
			  spec:
			    selector:
			      istio: ingressgateway
			    servers:
			    - port:
			        number: 80
			        name: http
			        protocol: HTTP
			      hosts:
			      - "istio.bookinfo"
			  ---
			  apiVersion: networking.istio.io/v1alpha3
			  kind: VirtualService
			  metadata:
			    name: bookinfo
			    namespace: book-info
			  spec:
			    hosts:
			    - "istio.bookinfo"
			    gateways:
			    - bookinfo-gateway
			    http:
			    - match:
			      - uri:
			          exact: /productpage
			      - uri:
			          prefix: /static
			      - uri:
			          exact: /login
			      - uri:
			          exact: /logout
			      - uri:
			          prefix: /api/v1/products
			      route:
			      - destination:
			          host: productpage
			          port:
			            number: 9080
```
Check Ips/port for that
```
			  export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
			  export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
			  export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
			  export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')
			  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```
add `/etc/hosts`: `$INGRESS_HOST bookinfo-gateway`
Expose Telemetry services to external:
Grafana Gateway
```
					  apiVersion: networking.istio.io/v1alpha3
					  kind: Gateway
					  metadata:
					    name: grafana-gateway
					    namespace: istio-system
					  spec:
					    selector:
					      istio: ingressgateway
					    servers:
					    - port:
					        number: 80
					        name: http-grafana
					        protocol: HTTP
					      hosts:
					      - "istio.grafana"
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: VirtualService
					  metadata:
					    name: grafana-vs
					    namespace: istio-system
					  spec:
					    hosts:
					    - "istio.grafana"
					    gateways:
					    - grafana-gateway
					    http:
					    - route:
					      - destination:
					          host: grafana
					          port:
					            number: 3000
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: DestinationRule
					  metadata:
					    name: grafana
					    namespace: istio-system
					  spec:
					    host: grafana
					    trafficPolicy:
					      tls:
					        mode: DISABLE
```
Prometheus Gateway:
```
					  apiVersion: networking.istio.io/v1alpha3
					  kind: Gateway
					  metadata:
					    name: prometheus-gateway
					    namespace: istio-system
					  spec:
					    selector:
					      istio: ingressgateway
					    servers:
					    - port:
					        number: 80
					        name: http-prom
					        protocol: HTTP
					      hosts:
					      - "istio.prometheus"
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: VirtualService
					  metadata:
					    name: prometheus-vs
					    namespace: istio-system
					  spec:
					    hosts:
					    - "istio.prometheus"
					    gateways:
					    - prometheus-gateway
					    http:
					    - route:
					      - destination:
					          host: prometheus
					          port:
					            number: 9090
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: DestinationRule
					  metadata:
					    name: prometheus
					    namespace: istio-system
					  spec:
					    host: prometheus
					    trafficPolicy:
					      tls:
					        mode: DISABLE
```
Kiali Gateway:
```
					  apiVersion: networking.istio.io/v1alpha3
					  kind: Gateway
					  metadata:
					    name: kiali-gateway
					    namespace: istio-system
					  spec:
					    selector:
					      istio: ingressgateway
					    servers:
					    - port:
					        number: 80
					        name: http-kiali
					        protocol: HTTP
					      hosts:
					      - "istio.kiali"
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: VirtualService
					  metadata:
					    name: kiali-vs
					    namespace: istio-system
					  spec:
					    hosts:
					    - "istio.kiali"
					    gateways:
					    - kiali-gateway
					    http:
					    - route:
					      - destination:
					          host: kiali
					          port:
					            number: 20001
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: DestinationRule
					  metadata:
					    name: kiali
					    namespace: istio-system
					  spec:
					    host: kiali
					    trafficPolicy:
					      tls:
					        mode: DISABLE
```
Jaeger Gateway:
```
					  apiVersion: networking.istio.io/v1alpha3
					  kind: Gateway
					  metadata:
					    name: tracing-gateway
					    namespace: istio-system
					  spec:
					    selector:
					      istio: ingressgateway
					    servers:
					    - port:
					        number: 80
					        name: http-tracing
					        protocol: HTTP
					      hosts:
					      - "istio.jaeger"
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: VirtualService
					  metadata:
					    name: tracing-vs
					    namespace: istio-system
					  spec:
					    hosts:
					    - "istio.jaeger"
					    gateways:
					    - tracing-gateway
					    http:
					    - route:
					      - destination:
					          host: tracing
					          port:
					            number: 80
					  ---
					  apiVersion: networking.istio.io/v1alpha3
					  kind: DestinationRule
					  metadata:
					    name: tracing
					    namespace: istio-system
					  spec:
					    host: tracing
					    trafficPolicy:
					      tls:
					        mode: DISABLE
```
`/etc/hosts`
``` urls
					  35.242.190.10 istio.bookinfo-gateway
					  35.242.190.10 istio.grafana
					  35.242.190.10 istio.prometheus
					  35.242.190.10 istio.kiali
					  35.242.190.10 istio.jaeger
```
## Observability
[TCP Metrics](https://istio.io/latest/docs/tasks/observability/metrics/tcp-metrics/)