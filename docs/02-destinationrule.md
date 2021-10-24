# 02-destinationrule

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

- 创建 busybox 资源
  
```bash
kubectl apply -f -<<EOF
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

- 验证 httpd 和 tomcat 资源是否可以连通

```bash
kubectl exec -it hexiaohong-client-xxxxxx /bin/sh

// 验证 httpd 资源是否可通
wget -q -O - http://httpd-svc:8080

// 验证 tomcat 资源是否可通
wget -q -O - http://tomcat-svc:8080
```

- 创建名为 web-svc 的 service 资源

```bash
kubectl apply -f -<<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
---
EOF
```

- 验证 web-svc 是否可通

```bash
kubectl exec -it po/hexiaohong-client-xxxx    /bin/sh
// 验证 service 是否可通
wget -q -O - http://web-svc:8080
```

- 创建 destination rule 资源
  
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: demo-des
spec:
  host: web-svc
  subsets:
  - name: v1
    labels:
      server: httpd
  - name: v2
    labels:
      server: tomcat
---
EOF
```

- 创建 virtualservice 资源

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-svc-vs3
spec:
  hosts:
  - web-svc
  http:
  - route:
    - destination:
        host: web-svc
        subset: v1
---
EOF
```

- 验证 web-svc 返回的结果
  
```bash
kubectl exec -it po/hexiaohong-client-xxxx    /bin/sh

// 验证 service 是否可通
wget -q -O - http://web-svc:8080
```

- 修改 virtualservice 指定的资源版本为 v2
  
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-svc-vs3
spec:
  hosts:
  - web-svc
  http:
  - route:
    - destination:
        host: web-svc
        subset: v2
---
EOF
```

- 验证 web-svc 返回的结果
  
```bash
kubectl exec -it po/hexiaohong-client-xxxx    /bin/sh

// 验证 service 是否可通
wget -q -O - http://web-svc:8080
```
