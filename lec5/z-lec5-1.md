# lecture-5-1
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec5
```


# 1. namespace

## 1.1 web1 namespace
```sh
k create ns web1
```

nginx.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80
```
```sh
## web1 namespace에 배포 
k apply -f nginx.yaml -n web1
```

## 1.2 web2 namespace
```sh
k create ns web2

```
httpd.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
  labels:
    app: "httpd"
spec:
  type: ClusterIP
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: "httpd"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
  labels:
    app: "httpd"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: "httpd"
  template:
    metadata:
      labels:
        app: "httpd"
    spec:
      containers:
      - name: httpd
        image: httpd:latest
        ports:
        - name: http
          containerPort: 80
```
```sh
## web2 namespace에 배포 
k apply -f httpd.yaml -n web2
```
## ingress-rule
- ingress-rule은 각 namespace에 위치 해야 한다 
  
web1-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web1-ing
spec:
  ingressClassName: nginx
  rules:
  - host: "nginx.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```
```sh

k apply -f web1-ing.yaml -n web1

curl http://nginx.3.39.152.82.sslip.io
```

web2-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web2-ing
spec:
  ingressClassName: nginx
  rules:
  - host: "apache.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
```sh
k apply -f web2-ing.yaml -n web2

curl http://apache.3.39.152.82.sslip.io
```

# 2. ConfigMap
- configmap을 통해서 nginx index.html을 변경 테스트 
  
configmap-example.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap
data:
  index.html: |
    <html><body><h1> ===== nginx configmap test index html ==== </h1></body></html>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80

        volumeMounts:
          - name: nginx-index-config-vol
            mountPath:  /usr/share/nginx/html/index.html
            subPath: index.html      
      
      volumes:
      - name: nginx-index-config-vol
        configMap:
          name: nginx-configmap
          items:
            - key: index.html
              path: index.html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
```sh
## 기존 배포된 nginx-deployemt가 변경된다 
k apply -f configmap-example.yaml -n web1

## 기존 nginx ingress 사용, 변경된 index 페이지 노출 
curl http://nginx.3.39.152.82.sslip.io/
```

# secret
secret-example.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: db
data:
  MYSQL_DATABASE: my_database ## 생성할 database 
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: db
stringData:
  MYSQL_ROOT_PASSWORD: "admin1234"  ## db 비번을 secret에 설정 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: db
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mariadb:10.7
        envFrom:
        - configMapRef:
            name: mysql-config
        - secretRef:
            name: mysql-secret
```
```sh
## db namespace를 생성한다 
k create ns db
## mysql 배포한다 
## namespace yaml에 설정되어 있다 
k apply -f secret-example.yaml 

## db namespace의 mysql pod로 들어가서 실행한다 
mysql -uroot -padmin1234
show databases;

+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database         |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
## "my_database" database 존재 확인
```
## secretKeyRef 기반으로 생성 테스트 
secret-example2.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: db
data:
  MYSQL_DATABASE: my_database ## 생성할 database 
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: db
stringData:
  MYSQL_ROOT_PASSWORD: "admin123456"   ## db 비번을 다시 변경해 본다  

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: db
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mariadb:10.7
        envFrom:
        - configMapRef:
            name: mysql-config
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
```
```sh
## 기존 mysql를 삭제한다 
k delete -f secret-example.yaml

k apply -f secret-example2.yaml

## db namespace의 mysql pod로 들어가서 실행한다 
mysql -uroot -padmin123456!
show databases;
```
# clear 
```sh
k delete -f configmap-example.yaml -n web1
k delete -f httpd.yaml -n web2
k delete -f secret-example2.yaml
```
