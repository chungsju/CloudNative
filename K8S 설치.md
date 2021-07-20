---
sort: 1
---
# Kubernetes Cluster 구성

[toc]

## 1. 환경 이해

`Kubernetes` 의 메뉴얼 환경 구성을 진행합니다.
본 Blog는 CentOS7 기반과 `Docker` Container 엔진을 기반으로 작성되었습니다.  

## 2. Docker 설치

### 2.1 설치 참조 URL
https://docs.docker.com/engine/install/centos/
해당 URL에서 CentOS7 이외에 기타 OS플랫폼에서의 `Docker` 설치 가이드를 참조하실수 있습니다.

  

### 2.2 Docker yum REPOSITORY 셋업
```{bash}
$ sudo sudo yum install -y yum-utils
$ sudo sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

  

### 2.3 Docker 설치

```{bash}
$ sudo yum install -y docker-ce docker-ce-cli containerd.io
```

  

### 2.3 Docker 데몬 구동

```{bash}
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

  

### 2.4 Docker 데몬 확인

```{bash}
$ sudo docker version
Client: Docker Engine - Community
 Version:           20.10.3
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        48d30b5
 Built:             Fri Jan 29 14:34:14 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.3
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       46229ca
  Built:            Fri Jan 29 14:32:37 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.3
  GitCommit:        269548fa27e0089a8b8278fc4fc781d7f65a939b
 runc:
  Version:          1.0.0-rc92
  GitCommit:        ff819c7e9184c13b7c2607fe6c30ae19403a7aff
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

  

### 2.5 도커 hub 사용을 위한 계정생성 (옵션)
`docker push` 명령어로 Docker Image를 생성 후  퍼블릭 Docker Hub에 배포하거나 별도 프라이빗 레지스트리(예:  AWS ECR)에 접근 하기 위해서는  `docker login` 이 필요합니다.
```{bash}
sudo docker login
```
> [docker hub](https://hub.docker.com/) (hub.docker.dom) 에 가입 후 퍼블릭 Docker Hub에  로그인이 가능 합니다.

  

### 2.6 사용자 유저 Docker Group 에 추가 (옵션)
`docker` 커맨드는 기본적으로 root 권한을 필요로 합니다.
일반 사용자 유저에서 `docker`를 사용하기 위해서는 해당 유저를 Docker Group에 추가해 주어야 합니다.

```{bash}
$ sudo groupadd docker
$ sudo usermod -aG docker [non-root user]
```

일반 유저로 Docker 데몬 확인
> 일반 유저가 Docker 그룹에 등록된 후, `docker` 명령어를 실행하기 위해서는 세션 재 접속이 필요합니다.
```{bash}
$ docker version
```

  


## 3. Kubernetes 설치

### 3.1 Kubernetes 개념
> Kubernetes 상세 개념은 https://kubernetes.io/ko/docs/concepts/ 를 참조 하시기 바랍니다.
> 
Kubernetes는 다음과 같은 기능을 제공해 하며 Container기반의 분산 시스템을 탄력적으로 실행하기 위한 프레임 워크를 제공합니다.


- **서비스 디스커버리와 로드 밸런싱** Kubernetes는 DNS 이름을 사용하거나 자체 IP 주소를 사용하여 Container를 노출할 수 있다. Container에 대한 트래픽이 많으면, Kubernetes는 네트워크 트래픽을 로드밸런싱하고 배포하여 배포가 안정적으로 이루어질 수 있다.
- **스토리지 오케스트레이션** Kubernetes를 사용하면 로컬 저장소, 공용 클라우드 공급자 등과 같이 원하는 저장소 시스템을 자동으로 탑재 할 수 있다.
- **자동화된 롤아웃과 롤백** Container를 사용하여 배포된 Container의 원하는 상태를 서술할 수 있으며 현재 상태를 원하는 상태로 설정한 속도에 따라 변경할 수 있다. 예를 들어 Kubernetes를 자동화해서 배포용 새 Container를 만들고, 기존 Container를 제거하고, 모든 리소스를 새 Container에 적용할 수 있다.
- **자동화된 빈 패킹(bin packing)** Container화된 작업을 실행하는데 사용할 수 있는 Kubernetes 클러스터 노드를 제공한다. 각 Container가 필요로 하는 CPU와 메모리(RAM)를 Kubernetes에게 지시한다. Kubernetes는 Container를 노드에 맞추어서 리소스를 가장 잘 사용할 수 있도록 해준다.
- **자동화된 복구(self-healing)** Kubernetes는 실패한 Container를 다시 시작하고, Container를 교체하며, '사용자 정의 상태 검사'에 응답하지 않는 Container를 죽이고, 서비스 준비가 끝날 때까지 그러한 과정을 클라이언트에 보여주지 않는다.
- **시크릿과 구성 관리** Kubernetes를 사용하면 암호, OAuth 토큰 및 SSH 키와 같은 중요한 정보를 저장하고 관리 할 수 있다. Container 이미지를 재구성하지 않고 스택 구성에 시크릿을 노출하지 않고도 시크릿 및 애플리케이션 구성을 배포 및 업데이트 할 수 있다.
  
  

**[Kubernetes 컴포넌트 구성도]**
![Kubernetes 컴포넌트](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

  

### 3.2 kubectl 설치 (Kubernetes client CLI 툴)

`kubectl` 은 Kubernetes  클러스터와 통신하는 커맨드라인 인터페이스 유틸입니다. 따라서 해당 툴은 Kubernetes 서버 환경 이외에 외부에서 접근 하는 client 노드에도 설치가 필요합니다.
```bash
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```
> `kubectl` 상세 설치 가이드는 https://kubernetes.io/ko/docs/tasks/tools/install-kubectl/ 를 참조하시기 바랍니다.

  

### 3.3 MiniKube 설치 (Single Node 개발 테스트용)

MiniKube는 Single Node에서 Kubernetes의 기능들을 테스트 해보기 위한 올인원 설치 파일입니다.

  

#### 3.3.1 MiniKube RPM 설치

```bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
$ sudo rpm -ivh minikube-latest.x86_64.rpm
```
> 상세 설치 가이드는 https://minikube.sigs.k8s.io/docs/start/ 를 참조하시기 바랍니다.

  

#### 3.3.2 MiniKube Start
```bash
$ minikube start
```

  

#### 3.3.3 MiniKube 설치 검증
```bash
$ kubectl get po -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-7nhl5            1/1     Running   0          10m
kube-system   etcd-minikube                      1/1     Running   0          10m
kube-system   kube-apiserver-minikube            1/1     Running   0          10m
kube-system   kube-controller-manager-minikube   1/1     Running   0          10m
kube-system   kube-proxy-m7c57                   1/1     Running   0          10m
kube-system   kube-scheduler-minikube            1/1     Running   0          10m
kube-system   storage-provisioner                1/1     Running   0          10m
```

  

#### 3.3.3 MiniKube 대쉬보드 
```bash
$ minikube dashboard
🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:40214/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
👉  http://127.0.0.1:40214/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

브라우저로 대쉬보드 접속
http://127.0.0.1:40214/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
![minikube dashboard](./img/minikube_dashboard.png)

> 해당 대쉬보드의 port는 proxy port 이며, `minikube dashboard`로 proxy 연결시 마다 port가 변경됩니다. 
> 만약 브라우저가 minikube가 설치되어 있는 Node가 아닌 외부에서 접속시에는 해당 노드에 포트 포워딩(ssh 터널링) 접속 후 브라우저 접속을 하면 됩니다.
> 포트 포워딩 접속 예 : `-L localport:127.0.0.1:targetport` 
>
> ```bash
> ssh -i ~/key.pem centos@15.165.49.47 -L 40214:127.0.0.1:40214
> ```
> putty를 이용한 윈도우에서 포트 포워딩(ssh 터널링) 방법


> ![putty](./img/putty_tunnels.png)

  

### 3.4 Kubernetes Cluster 설치 (메뉴얼 Cluster 구성)

#### 3.4.1 Kubernetes 구성도
![Kubernetes 구성도](./img/kube_archi.png)
최대한 운영 환경과 유사한 Kubernetes Cluster 구성을 위해 총 3개의 VM 환경하에 Cluster를 구성하도록 합니다.
1 VM : **Master Node** `kube-apiserver`를 통해 Cluster 전체를 컨트롤 하는 시스템
2 VM : **Worker Noder** API 서버의 요청을 `kubelet`을 통해 수행하고 실제 워크 로드를 생성하여 서비스 하는 시스템

  

#### 3.4.2 Cluster 구성 공통사항 (Master, Worker Node)

##### 3.4.2.1 Docker 설치
[Docker 설치 바로가기](#2. Docker 설치)

  

##### 3.4.2.2 Memory Swap 비활성

```bash
$ sudo swapoff -a
```

  

##### 3.4.2.3 kubernetes 설치

1. Kubernetes YUM repository 등록
```bash
$ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```
2. SELinux 모드 변경
```bash
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
3. Kubernetes 설치
```bash
$ sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
$ sudo systemctl enable --now kubelet
$ sudo systemctl start kubelet
```
>kubeadm : kubernetes 클러스터를 구축하기 위해 사용하는 툴입니다.
kubelet : 클러스터의 모든 머신에서 실행되며 Pod 및 컨테이너 시작 등의 작업을 수행하는 구성 요소입니다.
kubectl : 클러스터와 통신하는 커맨드라인 인터페이스 유틸입니다.
다른 OS 및 상세 설치 문서는 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ 를 참조하시기 바랍니다.

  

#### 3.4.3  Cluster 구성 - Master Node 

1. control-plane 초기화 
```bash
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
결과 확인
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 172.31.54.195:6443 --token 75zx8f.y2lxd7mgtsr3vv69 \
    --discovery-token-ca-cert-hash sha256:ee36574cb63e7577abe18abe816f05d26887009cee5109943a8dc761b2f9567c
```
> pod-network-cidr pod가 생성될 Network CIDR을 지정합니다. 내부 IP대역과 충돌시 변경 필요합니다.
> 
> 가장 마지막의 
> kubeadm join 172.31.54.195:6443 --token 75zx8f.y2lxd7mgtsr3vv69 \
    --discovery-token-ca-cert-hash sha256:ee36574cb63e7577abe18abe816f05d26887009cee5109943a8dc761b2f9567c 
명령 및 토큰은 Worker Node에서 필요하기 때문에 복사해 놓습니다.
2. `kubectl` 사용을 위한 디폴트 kube config 파일 설정
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> 해당 작업은 kubectl 을 사용할 모든 Client (개인 작업 PC)에 config 파일 복사가 필요합니다.
3. CNI(Container Network Interface) 설정
```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
>CNI는 ClusterDNS(CoreDNS)를 통해 배포된 Pod 끼리 통신할 수 있도록 합니다.

  

#### 3.4.4  Cluster 구성 - Worker Node 
Cluster 조인
```bash
kubeadm join 172.31.54.195:6443 --token 75zx8f.y2lxd7mgtsr3vv69 \
    --discovery-token-ca-cert-hash sha256:ee36574cb63e7577abe18abe816f05d26887009cee5109943a8dc761b2f9567c
```
> 앞서 Master Node 에서 `kubeadm init`명령어의 결과물을 복사합니다.

  


#### 3.4.5  Cluster 구성 검증

1. Node 구성 정보 확인
```bash
$ kubectl get nodes
NAME                                               STATUS   ROLES                  AGE   VERSION
ip-172-31-48-90.ap-northeast-2.compute.internal    Ready    <none>                 34d   v1.20.2
ip-172-31-56-200.ap-northeast-2.compute.internal   Ready    <none>                 34d   v1.20.2
ip-172-31-62-29.ap-northeast-2.compute.internal    Ready    control-plane,master   34d   v1.20.2
```
2. System Pods 정보 확인
```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                                                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-82kkr                                                    1/1     Running   0          19h
kube-system   coredns-74ff55c5b-qzqxg                                                    1/1     Running   0          19h
kube-system   etcd-ip-172-31-54-195.ap-northeast-2.compute.internal                      1/1     Running   0          19h
kube-system   kube-apiserver-ip-172-31-54-195.ap-northeast-2.compute.internal            1/1     Running   0          19h
kube-system   kube-controller-manager-ip-172-31-54-195.ap-northeast-2.compute.internal   1/1     Running   0          19h
kube-system   kube-flannel-ds-w8snt                                                      1/1     Running   0          6m12s
kube-system   kube-proxy-q86l7                                                           1/1     Running   0          19h
kube-system   kube-scheduler-ip-172-31-54-195.ap-northeast-2.compute.internal            1/1     Running   0          19h
```

  

#### 3.4.6  Kubernetes Dashboard 설정 (옵션)

1. Dashboard 설치
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```
> 상세 설치 가이드는 https://github.com/kubernetes/dashboard/blob/master/README.md#getting-started 를 참조하시기 바랍니다.
2. Dashboard 접속할 Client에서 Dashboard Proxy 연결
```bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```
> MiniKube의 Dashboard와 마찬가지로 Client가 아닌 서버단에서 proxy 생성 후 Client에서 포트 포워딩(ssh 터널링)으로 접근할 수도 있습니다.
3. `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login` URL로 부라우저 접속
![kubernetes_dashboard](./img/kuber_dashboard_token.png)
4. Dashboard 접속 토큰 생성

- serviceaccount 생성

```bash
$ cat <<EOF | kubectl create -f -
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: admin-user
   namespace: kube-system
EOF

serviceaccount/admin-user created
```

- ClusterRoleBinding을 생성

```bash 
$ cat <<EOF | kubectl create -f -
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
   name: admin-user
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: cluster-admin
 subjects:
 - kind: ServiceAccount
   name: admin-user
   namespace: kube-system
EOF

clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

- 사용자 계정의 토큰 보기

```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') 

Name:         admin-user-token-p9ldd
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=041cb7ec-946a-49b6-8900-6dc90fc08464

Type:  kubernetes.io/service-account-token
 
Data
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXA5bGRkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwNDFjYjdlYy05NDZhLTQ5YjYtODkwMC02ZGM5MGZjMDg0NjQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.TJtTZh5c-DpbmGwEF5QJ3yS9lADpqRCLJRnsr-t0xLOGPY4u3gNGXq8F4C4vL__PpfY-u5xmFTv-H3Oe6jTmjBnduR_bFNP1BqWGl3T1si-FC08a_aNqKbmjUKJ_4SWsSe_DjgUZTawr04C5dtNcBd0o0clQG_nkQFk7Gko2tiFhF0PwdtgSOh29JjlhAj_vzpd-vUNxMUJ0Y1ulPEfSCAA0pmIjBYpl_oBVMOd8Xkszz3FYC9ij9WevttnFMjxsEL5x6hGQw62OYQkx_YsE-P7db3cG0ieHECScHFp_Z0rg_lXBnDYFtj6hyPlpS7EvcVSRRUBhasPr3GlqHoIycg
```

>  마지막에 출력된 token 값을 복사하여 로그인 하시면 됩니다.
>  출처 : https://waspro.tistory.com/516

  

#### 3.4.7  Trouble shooting
##### 3.4.7.1 AWS EC2 상에서 kubectl 접속 시 vip 접근 문제
AWS EC2에서 kubernetes Cluster 생성시 apiserver 가 vip로 생성되어 외부에 config 파일을 복사하더라도 접속이 안되는 문제가 있습니다.

외부에서 접근 허용할 hostname 등록  `--apiserver-cert-extra-sans`
```bash
kubeadm init phase certs apiserver --apiserver-cert-extra-sans ip-172-31-62-29.ap-northeast-2.compute.internal
```
> 참조 출처 https://blog.dudaji.com/kubernetes/2020/04/08/add-ip-to-kube-api-cert.html

  

##### 3.4.7.2 AWS EC2 상에서 CNI(Container Network Interface) 설정
- EC2 인스턴스의 source/destination checks 체크 해제

instance -> action -> network ->Change source/destination checks 

<img src="./img/disable_destination_check1.png" width ="50%" >


Source/destination checking -> stop 체크

<img src="./img/disable_destination_check2.png" width ="50%" >

> 상세 내용은   https://docs.aws.amazon.com/vpc/latest/userguide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck 참조 바랍니다.


- Cluster 구성 노드간의 Security Group에 Custom Protocol 추가 

All traffic, All TCP, All UDP 가 아닌 `IP in IP` Protocol 명시적 추가 필요 
![custom protocol](./img/custom_protocol.png)

  

### 3.5 Kubernetes Cloud 서비스 사용
각 Cloud 벤더에서는 Kubernetes Cluster를 메뉴얼하게 설정하지 않고 바로 사용할수 있는 Kubernetes PaaS 서비스를 제공하고 있습니다.

- Google Kubernetes Engine : [https://cloud.google.com/kubernetes-engine?hl=ko](https://cloud.google.com/kubernetes-engine?hl=ko)
- AWS EKS : [https://aws.amazon.com/ko/eks/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc&eks-blogs.sort-by=item.additionalFields.createdDate&eks-blogs.sort-order=desc](https://aws.amazon.com/ko/eks/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc&eks-blogs.sort-by=item.additionalFields.createdDate&eks-blogs.sort-order=desc)
- Azure Kubernetes Service : [https://azure.microsoft.com/ko-kr/services/kubernetes-service/](https://azure.microsoft.com/ko-kr/services/kubernetes-service/)
- Oracle OKE : [https://www.oracle.com/kr/cloud-native/container-engine-kubernetes/](https://www.oracle.com/kr/cloud-native/container-engine-kubernetes/)
