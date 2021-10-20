# 01-通过 gateway 和 virtualservice 访问集群中的资源

## 通过 NodePort 的 Service 访问集群中的资源

```bash
// 创建 nginx deployment
kubectl create deploy nginx --image=nginx

// 创建 nginx NodePort Service
kubectl expose deploy nginx --type=NodePort --port=80

// 获取 nodePort 和  nodeIP
export nodePort=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
export nodeIP=$(minikube ip)

// 验证结果
curl -XGET $nodeIP:$nodePort
```

## 通过 gateway 和 virtualservice 访问集群内 http 资源

### 1. 不指定内部资源的 host

- 创建 deploy 和 service 相关资源
  
```bash
// 创建 deploy 资源
kubectl create deploy demo-nginx --image=nginx

// 创建 service 资源
kubectl expose deploy demo-nginx --type=ClusterIP --port=80
```

- 验证 svc 是否正常

```bash
// 启动 busybox 镜像
kubectl run -it --rm --restart=Never busybox --image=busybox /bin/sh

// 在命令行中校验是否连通
nslookup lookup demo-nginx
```

- 创建 gateway
  
```bash
kubectl apply -f -<<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-nginx-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---    
EOF    
```

- 创建 virtualservice

```bash
culr          
```

- 验证结果

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -XGET $INGRESS_HOST
curl -XGET $INGRESS_HOST/demo-nginx
```


### 2. 指定内部资源的 host

- 创建 deploy 和 service 相关资源

```bash
// 创建 deploy 资源
kubectl create deploy demo-nginx-2 --image=nginx

// 创建 service 资源
kubectl expose deploy demo-nginx-2 --type=ClusterIP --port=80
```

- 校验 svc 是否连通

```bash
// 启动 busybox 镜像
kubectl run -it --rm --restart=Never busybox --image=busybox /bin/sh

// 在命令行中校验是否连通
nslookup lookup demo-nginx-2
```

- 创建相关 gateway

```bash
kubectl apply -f -<<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-nginx-2-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "demo-nginx-2.com"
---    
EOF   
```

- 创建相关 virtual service
  
```bash
kubectl apply -f -<<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: demo-nginx-2
spec:
  hosts:
  - "demo-nginx-2.com"
  gateways:
  - demo-nginx-2-gateway
  http:
  - match:
    - uri:
        exact: /demo-nginx-2
    rewrite:
      uri: /    
    route:
    - destination:
        host: demo-nginx-2
        port:
          number: 80
---
EOF
```

- 验证结果

```
curl  -HHost:demo-nginx-2.com -XGET http://10.99.196.111/demo-nginx-2
```
