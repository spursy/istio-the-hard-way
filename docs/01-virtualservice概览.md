# 01-virtualservice

## 创建 httpd 和 tomcat 资源

- 创建 httpd 资源

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
  labels:
    server: httpd
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      server: httpd
      app: web
  template:
    metadata:
      labels:
        server: httpd
        app: web
    spec:
      containers:
      - name: busybox
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh", "-c", "echo 'this is httpd' > /var/www/index.html; httpd -f -p 8080 -h /var/www"]
---
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    server: httpd
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
---
EOF
```

![创建httpd资源](/screen-shot/01/创建httpd资源.png)

- 创建 tomcat 资源

```bash
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
  labels:
    server: tomcat
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      server: tomcat
      app: web
  template:
    metadata:
      labels:
        server: tomcat
        app: web
    spec:
      containers:
      - name: tomcat
        image: docker.io/kubeguide/tomcat-app:v1
        imagePullPolicy: IfNotPresent     
--- 
apiVersion: v1
kind: Service
metadata:
  name: tomcat-svc
spec:
  selector:
    server: tomcat
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
---
EOF
```

![创建tomcat资源](/screen-shot/01/创建tomcat资源.png)

- 创建 busybox 资源
  
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hexiaohong-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hexiaohong-client
  template:
    metadata:
      labels:
        app: hexiaohong-client
    spec:
      containers:
      - name: busybox
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh", "-c", "sleep 3600"]
---
EOF
```

![创建busybox资源](/screen-shot/01/创建busybox资源.png)

- 验证 httpd 和 tomcat 资源是否可以连通

```bash
kubectl exec -it hexiaohong-client-xxxxxx /bin/sh

// 验证 httpd 资源是否可达
wget -q -O - http://httpd-svc:8080

// 验证 tomcat 资源是否可达
wget -q -O - http://tomcat-svc:8080
```

![验证httpd的连通性](/screen-shot/01/验证httpd的连通性.png)

![验证tomcat的连通性](/screen-shot/01/验证tomcat的连通性.png)

## 创建 virtualservice 相关资源

- 创建名为 web-svc 的 service 资源

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: hexiaohong-client
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
---
EOF    
```

![创建web-svc资源](/screen-shot/01/创建web-svc资源.png)

- 创建 virtual service

```bash
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-svc-vs
spec:
  hosts:
  - web-svc.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: httpd-svc
      weight: 80
    - destination:
        host: tomcat-svc
      weight: 20
---
EOF
```

- hosts 是 service 资源，也可使用 service 的简写，如 web
- host 是目标 service 资源

![创建virtualservice资源](/screen-shot/01/创建virtualservice资源.png)

## 验证 virtualservice 资源

- 验证 virtualservice 是否生效

```bash
kubectl exec -it hexiaohong-client-xxxxxx /bin/sh

// 验证 web-svc 资源是否连通
wget -q -O -  http://web-svc:8080
```

![验证tomcat的连通性](/screen-shot/01/验证web-svc请求到tomcat资源.png)

![验证web-svc请求到httpd资源](/screen-shot/01/验证web-svc请求到httpd资源.png)

- 修改 virtualservice，使得部分 route 携带 header 参数
  
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-svc-vs
spec:
  hosts:
  - web-svc.default.svc.cluster.local
  http:
  - match:
    - headers:
        to:
          exact: httpd
    route:
    - destination:
        host: httpd-svc
  - route:
    - destination:
        host: tomcat-svc
---
EOF        
```  

- 验证 virtualservice 是否生效

```bash
kubectl exec -it hexiaohong-client-xxxxxx /bin/sh

// 验证 web-svc 资源是否连通
wget -q -O -  http://web-svc:8080 --header 'to:httpd'

wget -q -O -  http://web-svc:8080
```

![验证httproute的header-match请求](/screen-shot/01/验证httproute的header-match请求.png)