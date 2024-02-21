# lecture-6
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  ~/k8s-edu/lec6
```


# 1. CI/CD 
- git : github
- image Registry: docker-hub
- giOps: github
- build :  docker build
- deploy: argocd
- github/docker-hub 계정 필요 

# 2. install argo-cd 
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## argocd password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# mB5LvCcBW53I2I4G

```
## 2.1 rke2 Updating Nginx Helm
- rke2에 미리 설치된 rke2-ingress-nginx-controller는 enable-ssl-passthrough 에 대한 설정이 없다 
- 다음과 같이 설정을 추가 할수 있다 
```bash
kubectl apply -f rke2-ingress-nginx.yaml

```
- 1개씩 update 가 된다 (완료 될때 까지 기다린다)

## 2.2 lightsail master 서버 443 오픈 
- master-1 서버의 network에서 443 추가 
  
argocd-ingress
```sh
kubectl apply -f argocd-ing.yaml
```
### 2.3 access argocd ui
- https://argocd.3.39.152.82.sslip.io/
- admin/mB5LvCcBW53I2I4G
- changepassword: admin1234
- 재로그인 

# 3. docker 설치 
```bash
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
## docker build
- exam6 > apps > demo
- demo 프로젝트에서 docker build를 한다 
- -t 의 saturn203 은 각자 개인 docker-hub계정으로 변경한다 
```sh
docker info
docker login
docker build -t saturn203/vas:0.0.1 .   ## 마지막의 .을 생략하면 안됨
docker images
docker run -p 8080:8080 saturn203/vas:0.0.1 
curl localhost:8080 
docker push saturn203/vas:0.0.1

```
- docker-hub에서 push 사항을  확인한다 

# 4. demo-gitops 
- 각자 개인 github 에서 demo-gitops 를 public으로 생성한다 
- 생성된 demo-gitops 를 클론 한다 
```sh
cd ~
git clone [repository-uri]
## ex) git clone https://github.com/io203/demo-gitops.git 

## lec6 > vas 폴더를 vas-gitops 안에 copy 한다 
## vas-ing.yaml host를 변경한다 
cp -rf ~/k8s-edu/lec6/vas ~/demo-gitops/
cd ~/demo-gitops
git config --global --edit   ## 자신의 유저명, 메일로 작성 
git commit --amend --reset-author ## 수정없이 그냥 나간다 

git push origin main
## github login 
## password : 개인 github >  Settings > Developer Settings > Personal access tokens > Tokens(classic) >  Generate new token >  token값


```

# 5.  argocd git repository 설정 
- argocd 홈  >  Settings > Repositories > connect REPO
- VIA HTTPS 선택 
- type: git
- project: default
- Repository URL : https://github.com/io203/demo-gitops.git
- Username :  각 개인 계정 
- Password :  개인 github >  Settings > Developer Settings > Personal access tokens > Tokens(classic) >  Generate new token >  token값

# 6. vas namespace 생성 
```sh
kubectl create ns vas 
```

# 7.  argocd applications
- Applications > NEW APP
- Name: vas
- Project Name: default
- Repository URL :  선택 
- Revision: main
- Path :  vas 선택 
- Cluster URL :  https://kubernetes.default.svc 선택 
- Namespace:  vas
- kustomize : Images 부분에 이미지와 tag 버전이 맞는지 확인 
- 위의 CREATE 버튼 클릭
- SYNC 버튼 클릭 >  SYNCRONIZE

## 7.1 vas 서비스 확인 
- http://vas.3.39.152.82.sslip.io/

# 8. clear
```sh
## arog UI 에서 vas application delete 한다  
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

cd ~/k8s-edu/lec6
k delete -f argocd-ing.yaml

```
