# Introduction
The purpose of this document is to explain how thanos querier can connect to a thanos sidecar of a prometheus instance deployed on non-local (remote cluster which thanos is not deployed on there) cluster securely using TLS to get status of that prometheus metrics. Since the request type of thanos querier is GRPC and this protocol can't handle domain names (and use IPs and ports for connecting to thanos sidecar of prometheus instances), envoy is used to rewrite thanos querier request and send that to remote thanos seidecar service.
>The traffic between envoy and thanos-sidecar is mTLS, so either thanos-sidecar ingress and envoy should use same tls certificates.

Through this documentary, following topology is going to be applied.

![CDN monitoring(2)](https://github.com/rezaebrahimi1/envoy-thanos/assets/71483991/a94ce63f-94de-4eb5-bdf9-f3508ff2bd0a)

Now let's walk through above topology. The installation part (part 3) of this document is based on the above image and following descriptions:

1- Thanos querier get configured such that send traffic to envoy services (one service for each call):

=> dnssrv+_envoy-cl1._tcp.envoy-cl1.envoy.svc.cluster.local

=> dnssrv+_envoy-cl2._tcp.envoy-cl2.envoy.svc.cluster.local

2- Envoy services forward traffic to envoy pod using following configuration

>envoy service for cluster2 (port 10003 is a arbitrary port)
```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: envoy-cl2
  name: envoy-cl2
  namespace: envoy
spec:
  ports:
  - name: envoy-cl2
    port: 10003
    protocol: TCP
    targetPort: 10003
  selector:
    name: envoy
  sessionAffinity: None
  type: ClusterIP
```
>envoy service for cluster1 (port 10002 is a arbitrary port
```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: envoy-cl1
  name: envoy-cl1
  namespace: envoy
spec:
  ports:
  - name: envoy-cl1
    port: 10002
    protocol: TCP
    targetPort: 10002
  selector:
    name: envoy
  sessionAffinity: None
  type: ClusterIP
```
3,4,5- Envoy pod get grpc traffics from envoy services, rewrite them with pre-specified domain names (cl1m.example.com and cl2m.example.com for cluster 1 , 2 respectively) and route them to thanos-sidecar (here haproxy is used as an interface between thanos pod and thanos-sidecar). This task is done using following envoy config which deployed as configmap:
>Envoy configmap
```
static_resources:
  listeners:
  - name: sidecar_cl2
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10003
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: sidecar_cl2
                  host_rewrite_literal: cl2m.example.com
          http_filters:
          - name: envoy.filters.http.router
 
  - name: sidecar_cl1
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10002
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: sidecar_cl1
                  host_rewrite_literal: cl1m.example.com
          http_filters:
          - name: envoy.filters.http.router
  clusters:
  - name: sidecar_cl2
    connect_timeout: 30s
    type: LOGICAL_DNS
    http2_protocol_options: {}
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: sidecar_cl2
      endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: cl2m.example.com
                  port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificates:
            - certificate_chain:
                filename: /certs/fullchain.pem
              private_key:
                filename: /certs/privkey.pem
          alpn_protocols:
          - h2
          - http/1.1
        sni: cl2m.example.com
  - name: sidecar_cl1
    connect_timeout: 30s
    type: LOGICAL_DNS
    http2_protocol_options: {}
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: sidecar_cl1
      endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: cl1m.example.com
                  port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificates:
            - certificate_chain:
                filename: /certs/fullchain.pem
              private_key:
                filename: /certs/privkey.pem
          alpn_protocols:
          - h2
          - http/1.1
        sni: cl1m.example.com
```
5- Haproxy route traffic to thanos-sidecar ingress resource

6- Thanos sidecar ingress route traffic to thanos sidecar service and atlast thanos sidecar pod on observed cluster.

# Prerequisites

1- Thanos deployed on observer cluster

2- Prometheus with thanos-sidecar deployed on observed cluster

3- fullchain.pem and privkey.pem for clsuter1,2 rewrited grpc calls (here cl1m.example.com and cl2m.example.com).

# Installation
1- Run following Commands to deploy envoy on observer cluster:
```
**On observer cluster**
git clone https://github.com/rezaebrahimi1/envoy-thanos
kubectl create ns envoy
cd observer
kubectl create secret generic certs --from-file=./certs -n envoy
kubectl create configmap config --from-file=./config -n envoy
kubectl apply -f deploy.yaml -n envoy
kubectl apply -f service-envoy-cl1.yaml -n envoy
kubectl apply -f service-envoy-cl2.yaml -n envoy
```
2- Run following command to deploy thanos-sidecar ingress resource
```
# On observed cluster01 (IAAS)
cd observed/
kubectl create secret tls envoy --namespace <thanos-sidecar namespace> --key privkey.pem --cert fullchain.pem
kubectl apply -f ing-sidecar-cl1.yaml -n <thanos-sidecar namespace>
 
# On observed cluster02 (asiatech)
cd observed/
kubectl create secret tls envoy --namespace <thanos-sidecar namespace> --key privkey.pem --cert fullchain.pem
kubectl apply -f ing-sidecar-cl2.yaml -n <thanos-sidecar namespace>
```
References
https://thanos.io/tip/operating/cross-cluster-tls-communication.md/
