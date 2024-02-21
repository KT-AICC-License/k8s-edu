# lecture-4
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec4
```

## 1. install helm
```sh
## install-vm에서 실행 
sudo snap install helm --classic

helm version
```

## 1.1 rke2 etcd-expose-metrics 확인 
- https://docs.rke2.io/install/requirements
- rke2는 기본적으로 etcd-expose-metrics: false로 되어 있다 
- 따라서 이를 true로 변경 해야 한다 rke2 설치시 /etc/rancher/rke2/config.yaml 에 추가 한다 
```bash
cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
tls-san:
  - $EXTERNAL_IP
etcd-expose-metrics: true
EOF
```
- rke2 설치후 중간에 설정시 /etc/rancher/rke2/config.yaml 추가후 다음과 같이 재 실행 한다 
```sh
# sudo systemctl restart rke2-server.service

```

## 1.3 install promethues-stack
- kube-prometheus-stack으로 설치시 promethus,grafana,node-exporter가 패키지로 설치됨
```sh
kubectl create ns monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```


custom-values.yaml
```yaml
kubeControllerManager:  
  service:
    enabled: true
    port: 10250
    targetPort: 10250
kubeScheduler:  
  service:
    enabled: true
    port: 10250
    targetPort: 10250
```
- rke2는  kubelet metric port 10250이다 

```sh
helm install prometheus prometheus-community/kube-prometheus-stack  -f prometheus-custom-values.yaml -n monitoring
```


## 1.4 access port
- prometheus-kube-prometheus-prometheus  9090
- prometheus-grafana 80
  
prometheus-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ing
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: "prometheus.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus 
            port:
              number: 9090

```
grafana-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ing
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: "grafana.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
```

```sh 
## ingress-rule 적용
k apply -f prometheus-ing.yaml
k apply -f grafana-ing.yaml
```

## 1.5 prometheus > status > targets 체크 
-  prometheus ui > status
-  http://prometheus.3.39.152.82.sslip.io
-  모두 UP 상태가 되어야 한다 
  
## 1.6 grafana UI
- http://grafana.3.39.152.82.sslip.io
- 로그인: admin/prom-operator
- datasource 및 dashboard가 이미 설정및 설치 되어 있다 

## 1.7 clear
```bash
## prometheus-stack 삭제
helm uninstall prometheus -n monitoring

```

# 2. opensearch

## 2.1 worker 3개 필요 
- opensearch 설치 하기 위해서 최소한 worker가 3개 필요하다 
- worker를 하나 만들고 lec0에서 했던 agent를 추가 한다 

## 2.2 opensearch는 pv가 필요하며  pv 때문에 설치(Rancher Local Path Provisioner)
```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

```

```sh
#  sysctl -w vm.max_map_count=262144 && sysctl -w fs.file-max=65536

kubectl create ns monitoring
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/opensearch -l

helm install opensearch bitnami/opensearch --version 0.6.1 -f opensearch-custom-values.yaml -n monitoring 
 
```
- running 할때까지 시간이 걸림

## 2.3 dashboard ingress
dashboard-ing.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: opensearch-dashboard-ing
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: "dashboard.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: opensearch-dashboards 
            port:
              number: 5601

```
```sh
## opensearch dashboard ingress 적용
k apply -f dashboard-ing.yaml

```
- dashboard 접속 : http://dashboard.3.39.152.82.sslip.io

## 2.4 sidecar 어플리케이션 log 수집
```sh
kubectl create ns nginx
kubectl apply -f sidecar-nginx-log.yaml

```
- nginx ui 접속
- http://nginx.3.39.152.82.sslip.io/
- 몇번 접속을 계속 한다 

## 2.5 opensearch dashboard에서 확인 
- index management > Indices >  server-nginx-log-* 확인
- Dashboards Management > Index patterns > Create index pattern > server-nginx-log-* 설정
- Discover 메뉴에서 로그 확인 

## 2.6 clear 
```sh 
kubectl delete -f sidecar-nginx-log.yaml
```

## 2.7 cluster-level 로그 수집 

### 2.7.1 nginx deployment
```bash
kubectl apply -f nginx.yaml
```

### 2.7.2 install fluent-bit daemonset
- fluent-bit 을 daemonset을 배포한다 
```sh
helm install fluent-bit bitnami/fluent-bit -f fluentbit-daemonset-custom-values.yaml --create-namespace  --namespace monitoring

```
### 2.7.3 nginx 접속하여 로그 발생 
- nginx ui 접속
- http://nginx.3.39.152.82.sslip.io/
- 
### 2.7.4 log 확인 
- index management > Indices >  nginx-* 확인

## 2.8 clear 
```sh
helm uninstall opensearch -n monitoring
helm uninstall fluent-bit -n monitoring
## pvc 를 강제적으로 삭제한다 
k get pvc -n monitoring 
k delete pvc data-opensearch-data-0 -n monitoring 
k delete pvc data-opensearch-data-1 -n monitoring 
k delete pvc data-opensearch-master-0 -n monitoring 
k delete pvc data-opensearch-master-1 -n monitoring 
k delete -f dashboard-ing.yaml
k delete -f nginx.yaml


```
