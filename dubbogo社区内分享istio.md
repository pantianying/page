# 组内会议

[TOC]

### istio相关分享
istio目前主要支持在k8s下的服务流量治理工作，所以学习之前首先需要了解k8s基础

### topology



##### version before 1.5

<img src="https://github.com/pantianying/page/blob/main/istio-1.4-.png?raw=true" alt="**istio-1.4-.png**" style="zoom:33%;" />

##### version after 1.5

<img src="https://github.com/pantianying/page/blob/main/istio-1.5%2B.png?raw=true" alt="**istio-1.5+.png**" style="zoom:50%;" />

###  sidecar注入原理

oc get mutatingwebhookconfigurations

configmap:istio-sidecar-injector



### 流量劫持原理

envoy端口

| 15000 | TCP  | Envoy admin port (commands/diagnostics)                      | Yes  |
| ----- | ---- | ------------------------------------------------------------ | ---- |
| 15001 | TCP  | Envoy outbound                                               | No   |
| 15006 | TCP  | Envoy inbound                                                | No   |
| 15008 | TCP  | Envoy tunnel port (inbound)                                  | No   |
| 15020 | HTTP | Merged Prometheus telemetry from Istio agent, Envoy, and application | No   |
| 15021 | HTTP | Health checks                                                | No   |
| 15090 | HTTP | Envoy Prometheus telemetry                                   | No   |

istiod端口

| Port  | Protocol | Description                                               | Local host only |
| ----- | -------- | --------------------------------------------------------- | --------------- |
| 15010 | GRPC     | XDS and CA services (Plaintext)                           | No              |
| 15012 | GRPC     | XDS and CA services (TLS, recommended for production use) | No              |
| 8080  | HTTP     | Debug interface (deprecated)                              | No              |
| 443   | HTTPS    | Webhooks                                                  | No              |
| 15014 | HTTP     | Control plane monitoring                                  | No              |

Kubelet --> istio-cni 

NetworkAttachmentDefinition
kubectl get network-attachment-definitions
目的是让Multus CNI链调用istio-cni插件

```shell
uname=envoy
uid=1337
iptalbes -t nat -F
iptables -t nat -I PREROUTING -p tcp -j REDIRECT --to-ports 16666
iptables -t nat -N ENVOY_OUTPUT
iptables -t nat -A OUTPUT -p tcp -j ENVOY_OUTPUT
iptables -t nat -A ENVOY_OUTPUT -p tcp -d 127.0.0.1/32 -j RETURN
iptables -t nat -A ENVOY_OUTPUT -m owner --uid-owner ${uid} -j RETURN
iptables -t nat -A ENVOY_OUTPUT -p tcp -j REDIRECT --to-ports 16666
```

查看对应进程network ns中的iptable规则

nsenter -n --target pid
iptables -t nat -L -v
iptables-save > /tmp/iptables.log



```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:ISTIO_REDIRECT - [0:0]
:ISTIO_IN_REDIRECT - [0:0]
:ISTIO_INBOUND - [0:0]
:ISTIO_OUTPUT - [0:0]
-A PREROUTING -p tcp -j ISTIO_INBOUND
-A OUTPUT -p tcp -j ISTIO_OUTPUT
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006
-A ISTIO_INBOUND -p tcp -m tcp --dport 22 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15090 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15021 -j RETURN
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -o lo -m owner ! --uid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --gid-owner 1337 -j ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -o lo -m owner ! --gid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN
-A ISTIO_OUTPUT -d 100.0.0.0/8 -j RETURN
-A ISTIO_OUTPUT -j ISTIO_REDIRECT
COMMIT
```



### 细节原理和配置

#### istio概念 

 `virtualService/DestinationRule/ServiceEntry/Gateway/sidecar`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # interpreted as reviews.foo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: 2

```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: foo
spec:
  host: reviews # interpreted as reviews.foo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain # not used
  addresses:
  - 192.192.192.192/24 # VIPs
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: istio-config
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

#### envoy概念

`Downstream Upstream`

`xds and route/listener/cluster/endpoint`

<img src="https://github.com/pantianying/page/blob/main/envoy%E6%A6%82%E5%BF%B5.jpg?raw=true" alt="envoy模块1" style="zoom:50%;" />



配置文件解读 todo


#### dubbo协议

https://github.com/aeraki-framework/aeraki







### mesh方案落地

#### 服务改造

<img src="https://github.com/pantianying/page/blob/main/app_mesh_%20transform.png?raw=true" alt="java服务改造" style="zoom: 67%;" />

#### 内外调用方案
<img src="https://github.com/pantianying/page/blob/main/IRA.png?raw=true" alt="IRA" style="zoom:50%;" />
<img src="https://github.com/pantianying/page/blob/main/mesh%E5%86%85%E5%A4%96%E8%B0%83%E7%94%A8.png?raw=true" alt="内外调用" style="zoom:50%;" />

#### log metric trace方案，多租户等等