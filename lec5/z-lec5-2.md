# lecture-5-2
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec5
```

# 1. emptyDir
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
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
        - name: log-volume
          mountPath: /var/log/nginx

      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:2.1.0
        volumeMounts:
        - name: log-volume
          mountPath: /var/log/nginx

      volumes: # 볼륨 선언
      - name: log-volume
        emptyDir: {}
```
```sh 

k apply -f emptydir-vol.yaml

## emptyDir volume인 log-volume으로 설정해 놓았기 때문에 
##  fluent-bit container에서 이제 nginx 의 access.log  error.log 를 읽을수 있도록 가능해 졌다 
## nginx pod의 fluent-bit container로 접속하여 아래와 같이  nginx의 로그파일이 조회 되는지 확인한다 
ls /var/log/nginx

## clear 
k delete -f emptydir-vol.yaml
```

# 2. hostPath
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-hostpath-nginx
    image: nginx:1.17
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # 파일 디렉터리가 생성되었는지 확인한다.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```
```sh
## nginx를 배포한다 
k apply -f hostpath-vol.yaml

## pod의 디렉토리및 파일이 생성 되었는지 확인  
k exec -it test-webserver -- ls /var/local
k exec -it test-webserver -- ls /var/local/aaa

## 실제 pod가 배포된 node의 에서 디렉토리및 파일이 생성 되었는지 확인  
##  pod가 배포된 노드 확인 
k get pod test-webserver -o wide
## node에 ubuntu로 로그인 하여 host에  생성 되었는지  조회 한다 
ls /var/local/aaa

## 다른 노드에서 확인한다 
## ls /var/local/aaa 조회 되지 않을 것이다 

## clear 
k delete -f hostpath-vol.yaml
```

# 3. pv/pvc

## 3.1  노드에 index.html 파일 생성
```sh
# 사용자 노드에서 슈퍼유저로 명령을 수행하기 위하여
# "sudo"를 사용한다고 가정한다
## worker-1 에만 생성해 본다 
sudo mkdir -p /mnt/data
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
cat /mnt/data/index.html
```

## 3.2 pv
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

```
```sh
kubectl apply -f task-pv-volume.yaml
kubectl get pv task-pv-volume
```
## 3.3 pvc
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
```sh
kubectl apply -f task-pv-claim.yaml

## pvc 조회한다  status가 Bound 되어 있어야 한다  
kubectl get pvc task-pv-claim
## pv 조회한다  status가 Bound 되어 있어야 한다 
kubectl get pv task-pv-volume
```

## 3.4 pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
   volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
```
```sh
## pod 배포
kubectl apply -f task-pv-pod.yaml
kubectl get pod task-pv-pod -o wide

## pod 안으로 들어간다 
kubectl exec -it task-pv-pod -- /bin/bash

## 최신버전 nginx image 들은 보안 때문에 curl 이 없을수 있다 curl을 설치하자  
apt update
apt install curl
## nginx를 조회 해 본다 
curl http://localhost/    ## 'Hello from Kubernetes storage' 조회 안될수도 있다  
## 파일이 존재하는지 확인 
cat /usr/share/nginx/html/index.html

## 조회 안될 경우 이는 /mnt/data/index.html 생성한 노드에 pod가 생성되지 않는 경우이다 
## 다른 노드에도  /mnt/data/index.html를   생성해 놓으면 정상적으로 조회 된다 

```
## 3.5 clean 
```sh
kubectl delete pod task-pv-pod
kubectl delete pvc task-pv-claim
kubectl delete pv task-pv-volume
```

# 4. storageClass

## 4.1 nfs storageClass ( 실습 없음)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  pathPattern: "${.PVC.namespace}/${.PVC.annotations.nfs.io/storage-path}" # waits for nfs.io/storage-path annotation, if not specified will accept as empty string.
  onDelete: delete
```
## 4.2 pvc example
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    nfs.io/storage-path: "test-path" # not required, depending on whether this annotation was shown in the storage class description
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi

```

# 5. Rancher Local-Path-Provisioner
- local-path-storage를 지원하는 provider이다 

```sh
## Local-Path-Provisioner 배포 
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

## pod 확인
kubectl get pod -n local-path-storage 

## log 확인
kubectl logs -f -n local-path-storage  -l app=local-path-provisioner

```
## 5.1  pvc 생성 
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 128Mi
```
```sh
## 온라인 예제로 실행 
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pvc/pvc.yaml
## local-path-pvc pvc를 조회하면 status가  pending 되어 있다 pod consubmer가 생성되면 binding 된다  
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: local-path-pvc
```
```sh
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pod/pod.yaml

kubectl get pod  
## pod가 생성되면 pvc는 pod의 요청으로 pv를  자동으로  생성되면서 먼저 pv와 Binding 되면서 pvc도 binding 된다 
kubectl get pvc
## local-path-pvc   Bound    pvc-9875c93c-bb64-4761-b588-2b92341156af   128Mi 

kubectl get pv                
## pvc-9875c93c-bb64-4761-b588-2b92341156af   128Mi   


kubectl exec volume-test -- sh -c "echo k8s-edu-test-local-path-test > /data/test.txt"

# delete pod
kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pod/pod.yaml

# recreate
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pod/pod.yaml

# check volume content
kubectl exec volume-test -- sh -c "cat /data/test.txt"

## 생성된 pv에서 보면 hostPath 경로를 확인 가능(describe) 
hostPath:   
  path: /opt/local-path-provisioner/pvc-b728be50-6af5-4f62-b48c-8be928139647_default_local-path-pvc

## worker 노드에서 test.txt 확인해 보자 (1개에서 확인 가능)
cat /opt/local-path-provisioner/pvc-b728be50-6af5-4f62-b48c-8be928139647_default_local-path-pvc/test.txt

## nginx deployment
k apply -f local-path-vol-nginx.yaml

```

## 5.2 clear
```sh
kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
k delete -f local-path-vol-nginx.yaml
k delete pvc local-path-pvc
```

# 6.  longhorn
```sh
## install-vm 에서 실행 
## sudo 에서 실행 
## jq 설치
sudo snap install jq

## longhorn dependency package들을  check 해준다 
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/scripts/environment_check.sh | bash


## longhorn-iscsi  설치해 준다(default namespace 설치된다)
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-iscsi-installation.yaml

## nfs client설치해 준다 (default namespace 설치된다)
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-nfs-installation.yaml


## 모든 pod가 생성되면 다시 한번 longhorn check 해본다 
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/scripts/environment_check.sh | bash

## longhorn check가 모두 OK 이면 longhorn 설치할 준비가 완료 되었다 
## 일부 [WARN]로그는 실습에서는 무시하자 
##  install longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml

## longhorn pod 확인
kubectl get pod -n longhorn-system 

```

## 6.1  longhorn 예제 
```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 128Mi
```

longhorn-test-pod.yaml
```yaml

apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test-pod
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: longhorn-test-pvc

```
```sh
## pvc
k apply -f longhorn-test-pvc.yaml
## pod
k apply -f longhorn-test-pod.yaml

## 볼륨에 데이터 저장
kubectl exec longhorn-test-pod -- sh -c "echo ====local-path-test========== > /data/test.txt"

## pod 삭제
k delete pod longhorn-test-pod 
## pod 재생성
k apply -f longhorn-test-pod.yaml

kubectl exec longhorn-test-pod -- sh -c "cat /data/test.txt"

## nginx-deployment로 테스트 

## ReadWriteMany 로 생성
k apply -f longhorn-test-pvc2.yaml
k apply -f longhorn-test-nginx-deploy.yaml
## nginx pod 안에서 다음 실행 
echo ====local-path-test========== > /data/test.txt
cat /data/test.txt
## nginx deploy 삭제 
k delete -f longhorn-test-nginx-deploy.yaml
## nginx 재 생성 
k apply -f longhorn-test-nginx-deploy.yaml
## nginx pod 안에서 체크 
cat /data/test.txt

```
## 6.2 longhorn UI
```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ing
  namespace: longhorn-system
spec:
  ingressClassName: nginx
  rules:
  - host: "longhorn.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80

```
```sh 
k apply -f longhorn-ui-ing.yaml
```
- UI 접속: http://longhorn.3.39.152.82.sslip.io

## 6.3  app delete 
```sh
k delete -f longhorn-test-pod.yaml
k delete -f longhorn-test-nginx-deploy.yaml

## 강제 삭제시 
k delete pod nginx-vol-deployment-578bbd869f-x89lp --grace-period=0 --force
```
## 6.4 longhorn uninstall
```sh 
## [주의] longhorn 삭제 하려면  ui의 삭제 설정해줘야 한다 
## 상단메뉴 :  Setting 클릭 >  Deleting Confirmation Flag: 체크박스를 체크 한다 >  맨 하단의 save 버튼 클릭( 화면이동 없음)

## longhorn를 삭제하기위한 uninstall pod를 생성
kubectl create -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/uninstall/uninstall.yaml
kubectl get job/longhorn-uninstall -n longhorn-system -w

## long-ui 및 부가적인 것들 삭제하기 위해서 실행 
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
## longhorn-iscsi 삭제
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-iscsi-installation.yaml
## longhorn-nfs 삭제
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-nfs-installation.yaml
## uninstall job도 삭제
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/uninstall/uninstall.yaml


```


# 7. POD 의 Request / Limit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"


```

# 8. ETCD 
```sh
kubectl -n kube-system exec -it etcd-ip-172-26-13-104 -- /bin/bash

etcd --version
etcdctl version

## 환경변수 설정 
export ETCDCTL_ENDPOINTS='https://127.0.0.1:2379' 
export ETCDCTL_CACERT='/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt' 
export ETCDCTL_CERT='/var/lib/rancher/rke2/server/tls/etcd/server-client.crt' 
export ETCDCTL_KEY='/var/lib/rancher/rke2/server/tls/etcd/server-client.key' 
export ETCDCTL_API=3 

## health 
etcdctl endpoint health
## etcd member 리스트 
etcdctl member list
## endpoint status 조회
etcdctl endpoint status --cluster -w table
```
