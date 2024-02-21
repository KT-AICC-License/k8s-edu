# lecture-11
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  ~/k8s-edu/lec11

```


# 1. docker install(없다면 )
```sh
## install-vm에서 실행 
## ubuntu user로 실행 
sudo apt update

snap version  ## 2.61.1 되어야 한다 
## 2.61.1 아니면 아래 실행하여 version ip 한다 
# sudo snap refresh
sudo snap install docker 

sudo docker ps 

## Docker 그룹 생성(snap docker install은 docker 그룹을 만들지 않는다)
sudo addgroup --system docker

## sudo 없이 Docker 명령 실행
sudo usermod -a -G docker $USER

## 재로그인후 체크 
docker ps # 안될경우 sudo reboot 
```

# 2. ansible argocd 설치 및 App 배포 (demo-gitOps/vas)
```sh
cd ansible
## host-vm host 수정 
## argode-ing  host 수정 
## vars.yaml master-host 및 gitOps 변경

## ping test
ansible -i host-vm all -m ping

ansible-playbook -i host-vm playbook.yml -t "argocd, app-deploy" -e "@vars.yml"

```
- https://argocd.3.39.152.82.sslip.io/ 접속한다 
- 로그인 : admin/admin1234
- vas app이 정상적으로 생성되었는지 확인
- vas app 접속 : http://vas.3.39.152.82.sslip.io/

# demo app 수정 
```sh
## install-vm에서 실행 
cd ~/k8s-edu/lec11/apps/demo
cat ~/k8s-edu/lec11/apps/demo/src/main/java/com/example/demo/controller/DemoController.java
## return "hello world VAS !!! version : 1.0.0 "; 에서 소스 수정해 놓았다
## 변경하고 싶다면 
# vi ~/k8s-edu/lec11/apps/demo/src/main/java/com/example/demo/controller/DemoController.java

## docker build 
docker build -t [docker-hub 계정]/vas:1.0.0 . 
# ex} docker build -t saturn203/vas:1.0.0 . 
docker images
docker login 
docker push [docker-hub 계정]/vas:1.0.0
# ex} docker push saturn203/vas:1.0.0  
```

## demo-gitOps image tag update 
```sh
## install-vm에서 실행 
## demo-gitOps 없는경우 아래와 같이 git clone 한다 
# cd ~
# git clone https://github.com/io203/demo-gitops.git 

cd ~/demo-gitops/
vi vas/kustomization.yaml

 # newTag: 1.0.0 수정

 git add . ; git commit -m "vas:1.0.0 수정"; git push origin

```

## argocd vas app auto sync
- argocd 기본적으로 180초(3분) 마다 refresh가 이루어져 자동 sync 된다  






