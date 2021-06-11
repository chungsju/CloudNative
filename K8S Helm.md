# Learning Kubernetes

[toc]

## 16. Helm 차트 구성 및 사용(GCP 에서 수행)

### 16.1 Helm 차트 다운로드 및 설치

- 다운로드 (https://github.com/helm/helm/releases)
> 윈도우 환경의 경우 Windows amd64를 다운 받아 압축 해제
> helm.exe 경로를 PATH 환경에 추가

- 버전 확인

```{bash}
helm version

version.BuildInfo{Version:"v3.5.2", GitCommit:"167aac70832d3a384f65f9745335e9fb40169dc2", GitTreeState:"dirty", GoVersion:"go1.15.7"}
```

- helm 차트 리포지토리 추가

```{bash}
helm repo add bitnami https://charts.bitnami.com/bitnami
```

- 리포지토리 업데이트

```{bash}
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

### 16.2 mysql Helm 차트 다운로드 및 구성

- mysql helm 검색

```{bash}
helm search repo mysql

NAME            	CHART VERSION	APP VERSION	DESCRIPTION
bitnami/mysql    	1.6.3        	5.7.28     	Fast, reliable, scalable, and easy to use open-...
bitnami/mysqldump	2.6.0        	2.4.1      	A Helm chart to help backup MySQL databases usi...
```

- 피키지 메타 정보 보기

```{bash}
helm show chart bitnami/mysql

apiVersion: v1
appVersion: 5.7.28
description: Fast, reliable, scalable, and easy to use open-source relational database
  system.
home: https://www.mysql.com/
icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png
keywords:
- mysql
- database
- sql
maintainers:
- email: o.with@sportradar.com
  name: olemarkus
- email: viglesias@google.com
  name: viglesiasce
name: mysql
sources:
- https://github.com/kubernetes/charts
- https://github.com/docker-library/mysql
version: 1.6.3
```

- mysql helm 차트 설치 및 Deployment

```{bash}
helm install bitnami/mysql --generate-name

AME: mysql-1588321002
LAST DEPLOYED: Fri May  1 08:16:55 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-1588321002.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-1588321701 -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h mysql-1588321701 -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/mysql-1588321002 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

```{bash}
helm ls

NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS         C
HART            APP VERSION
mysql-1588321701        default         1               2020-05-01 17:28:25.322363879 +0900 +09 deployed       m
ysql-1.6.3      5.7.28
```

- helm 차스 uninstall

```{bash}
heml list

NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS         C
HART            APP VERSION
mysql-1588321701        default         1               2020-05-01 17:28:25.322363879 +0900 +09 deployed       m
ysql-1.6.3      5.7.28


helm uninstall mysql-1588321701
release "mysql-1588321701" uninstalled
```

### 16.3 wordpress Helm Chart 구성

- wordpress의 username과 패스워드를 셋팅한 상태로 helm Chart 구성
```{bash}
helm install helm-wordpress bitnami/wordpress --set wordpressUsername=myuser,wordpressPassword=mypassword
```

- helm 릴리스에 설정된 값 확인
```{bash}
helm get values helm-wordpress
USER-SUPPLIED VALUES:
wordpressPassword: mypassword
wordpressUsername: myuser
```

- 접속 IP 얻기
```
kubectl get svc --namespace default helm-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"
34.64.232.189 
```

- 해당 IP로 웹에서 로그인