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

**注意：** 一定要在 ./files/tls-ingress 目录下

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

```bash
curl  -HHost:demo-nginx-2.com -XGET http://10.99.196.111/demo-nginx-2
```

## 通过 gateway 和 virtualservice 访问集群内 https 资源

- 创建客户端和服务端认证
  
```bash
// 创建根证书和私钥用于下面的签名
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
--- 生成 example.com.crt 和 example.com.key 文件

// 为 nginx.example.com 创建证书和私钥
openssl req -out nginx.example.com.csr -newkey rsa:2048 -nodes -keyout nginx.example.com.key -subj "/CN=nginx.example.com/O=some organization"
--- 生成 nginx.example.com.csr 和 nginx.example.com.key
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in nginx.example.com.csr -out nginx.example.com.crt
--- 生成 nginx.example.com.crt
```

- 创建 secrets
  
```bash
kubectl create secret tls nginx-server-certs --key nginx.example.com.key --cert nginx.example.com.crt
```

- 创建 nginx 配置文件

```bash
cat <<\EOF > ./nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name nginx.example.com;
    ssl_certificate /etc/nginx-server-certs/tls.crt;
    ssl_certificate_key /etc/nginx-server-certs/tls.key;
  }
}
EOF
```

- 创建 configmap 文件
  
```bash
kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
```

- 创建 service 文件

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 443
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
          readOnly: true
        - name: nginx-server-certs
          mountPath: /etc/nginx-server-certs
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-configmap
      - name: nginx-server-certs
        secret:
          secretName: nginx-server-certs
---           
EOF
```

- 验证 nginx service 是否部署成功
  
```bash
kubectl exec "$(kubectl get pod  -l run=my-nginx -o jsonpath={.items..metadata.name})" -c istio-proxy -- curl -sS -v -k --resolve nginx.example.com:443:127.0.0.1 https://nginx.example.com
```

- 创建 ingress gateway

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - nginx.example.com
---    
EOF
```

- 创建 virtualservice
  
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx.example.com
  gateways:
  - mygateway
  tls:
  - match:
    - port: 443
      sniHosts:
      - nginx.example.com
    route:
    - destination:
        host: my-nginx
        port:
          number: 443
---          
EOF
```

- 验证
  
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}') 

curl -v --resolve "nginx.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert example.com.crt "https://nginx.example.com:$SECURE_INGRESS_PORT"
```


**参考文档:**
- [Ingress Gateway without TLS Termination](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-sni-passthrough/)