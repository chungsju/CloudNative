# Learning Kubernetes

[toc]

## 14. Namespace

### 14.1 네임스페이스 조회

```{bash}
kubectl get namespace
```

> kubectl get ns 와 동일함

[output]

```{bash}
NAME              STATUS   AGE
default           Active   17d
kube-node-lease   Active   17d
kube-public       Active   17d
kube-system       Active   17d
```

### 14.2 특정 네임스페이스의 POD 조회

```{bash}
kubectl get pod --namespace kube-system
# kubectl get po -n kube-system
```

> kubectl get pod -n kube-system 과 동일함

[output]

```{txt}
event-exporter-gke-67986489c8-68pqn                         2/2     Running   0          20d
fluentbit-gke-2c8zs                                         2/2     Running   0          14d
fluentbit-gke-8ss4h                                         2/2     Running   0          35h
fluentbit-gke-dtx6p                                         2/2     Running   0          14d
fluentbit-gke-qvmsx                                         2/2     Running   0          35h
fluentbit-gke-swgzd                                         2/2     Running   0          14d
fluentbit-gke-z9t9q                                         2/2     Running   0          35h
gke-metrics-agent-c5d52                                     1/1     Running   0          35h
gke-metrics-agent-j5x8p                                     1/1     Running   0          14d
gke-metrics-agent-mn79b                                     1/1     Running   0          14d
gke-metrics-agent-q6vm5                                     1/1     Running   0          35h
gke-metrics-agent-rhwxj                                     1/1     Running   0          14d
gke-metrics-agent-s4xbw                                     1/1     Running   0          35h
kube-dns-5d54b45645-22h6l                                   4/4     Running   0          20d
kube-dns-5d54b45645-9gjlk                                   4/4     Running   0          20d
kube-dns-autoscaler-58cbd4f75c-d86cv                        1/1     Running   0          20d
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-0jh0       1/1     Running   0          35h
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-1cjb       1/1     Running   0          35h
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-8d8p       1/1     Running   0          20d
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-hwv7       1/1     Running   0          35h
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-p8fn       1/1     Running   0          20d
kube-proxy-gke-myfirst-k8s-default-pool-54e76b14-qd40       1/1     Running   0          20d
l7-default-backend-66579f5d7-ctfk5                          1/1     Running   0          20d
metrics-server-v0.3.6-6f7ddb4685-x2mlc                      2/2     Running   0          34h
pdcsi-node-2k2jx                                            2/2     Running   0          20d
pdcsi-node-q92l8                                            2/2     Running   0          35h
pdcsi-node-qpvn8                                            2/2     Running   0          20d
pdcsi-node-scqj5                                            2/2     Running   0          35h
pdcsi-node-smgbt                                            2/2     Running   0          35h
pdcsi-node-zbffr                                            2/2     Running   0          20d
stackdriver-metadata-agent-cluster-level-64d756dff4-f5brd   2/2     Running   0          20d
```

###8.3 YAML 파일을 이용한 네임스페이스 생성

- YAML 파일 작성 : first-namespace.yaml 이름으로 파일 작성

```{bash}
apiVersion: v1
kind: Namespace
metadata:
  name: first-namespace
```

- YAML 파일을 이용한 네이스페이스 생성

```{bash}
kubectl apply -f first-namespace.yaml
```
> kubectl create namespace first-namespace 명령어와 동일 합니다.

[output]

```{txt}
namespace/first-namespace created
```

- 생성된 네임스페이스 확인

```{bash}
kubectl get namespace
kubectl get ns
```

[output]

```{txt}
NAME              STATUS   AGE
default           Active   17d
first-namespace   Active   5s
kube-node-lease   Active   17d
kube-public       Active   17d
kube-system       Active   17d
```

### 14.4 특정 네임스페이스에 POD 생성

- first-namespace 에 goapp 생성

```{bash}
kubectl run testpod --image=nginx -n first-namespace
```

[output]

```{txt}
pod/goapp-pod created
```

- 생성된 POD 확인하기

```{bash}
kubectl get pod -n first-namespace
```

[output]

```{txt}
NAME                       READY   STATUS    RESTARTS   AGE
testpod-6f4ccd6456-dxz4c   1/1     Running   0          5m5s
```

### 14.5 기본 네임스페이스 변경

- first-namespace 네임스페이스를 기본 네임스페이스로 변경하기

```{bash}
kubectl config set-context $(kubectl config current-context) --namespace first-namespace
```

- 변경된 기본 네임스페이스 확인
```{bash}
kubectl get pods
```


- 기본 네임스페이스를 default로 원복
```{bash}
kubectl config set-context $(kubectl config current-context) --namespace default
```

## 15 Service Account 및 RBAC

### 15.1 Service Account 생성

- Service Account 생성

```{bash}
kubectl create sa apiuser

# kubectl create sa apiuser --dry-run -o yaml

kubectl get sa

kubectl get sa --all-namespaces
```

### 15.2 롤 생성

- 롤 생성

```{yaml}
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

### 15.3 롤 바인딩

```{bash}
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-role-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: apiuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-role
  apiGroup: rbac.authorization.k8s.io
```

### 15.4 토큰 인코딩 및 서비스 테스트

- 서비스 어카운트의 token secret 얻기

```{bash}
kubectl describe sa apiuser
```
결과
```
Name:                apiuser
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   apiuser-token-xmdc2
Tokens:              apiuser-token-xmdc2
Events:              <none>
```

- token secret에서 token 얻기

```{bash}
kubectl describe secrets apiuser-token-xmdc2
```
> kubectl get secrets apiuser-token-xmdc2  -o jsonpath='{.data.token}' | base64 -d  명령어를 통해 data.token 부분만 바로 얻어올수도 있습니다.

```
Data
====
ca.crt:     1159 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImVFWmZWR254MjJ2ektScGxhUFItUFRwOEEzNDE4UXZuTC1QQnBRcGY4aHcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFwaXVzZXItdG9rZW4teG1kYzIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYXBpdXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNkY2I4NDVhLWUzNzYtNGI4My1hZDMwLWU1NTgxMDkxZDY5MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmFwaXVzZXIifQ.z7tKPfJkzUVaYzgmk3HJiGeZU1BuWEhVLDS4_IWjvTnvG80VwTnAsosESDDxnSknp5zERatkH1gBtysi_BBe_GEXHMIFHwThnUi42ff7A3NNcXX1xuvkVDW_IxHMxoPLtzei1f-RQ_7ByAVy8cSBU2YdxRb3HFEZmxlKnMDXIqywoG_KOi_72rruvV94OY0FGsaMTftPZsG4WbOfSMX4a3EVXvlcqtOU9wH2ALoaF6JFqXTscVW10qr9uruje6H_Vq6qFqHl-a8f1DPzLSWNrPJDfwbjBjmfHMSiV8taw3k8nxkSS-vIOdSx0xszzL-VpvWhfSuOBj6fNYRKZ-FRVQ
```

- Pod 리스트 조회
> GKE server IP는 ~/.kube/kubeconfig 파일 안에서 확인 가능합니다.
```{bash}
curl -k  https://GKEIP/api/v1/namespaces/default/pods/  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImVFWmZWR254MjJ2ektScGxhUFItUFRwOEEzNDE4UXZuTC1QQnBRcGY4aHcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFwaXVzZXItdG9rZW4teG1kYzIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYXBpdXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNkY2I4NDVhLWUzNzYtNGI4My1hZDMwLWU1NTgxMDkxZDY5MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmFwaXVzZXIifQ.z7tKPfJkzUVaYzgmk3HJiGeZU1BuWEhVLDS4_IWjvTnvG80VwTnAsosESDDxnSknp5zERatkH1gBtysi_BBe_GEXHMIFHwThnUi42ff7A3NNcXX1xuvkVDW_IxHMxoPLtzei1f-RQ_7ByAVy8cSBU2YdxRb3HFEZmxlKnMDXIqywoG_KOi_72rruvV94OY0FGsaMTftPZsG4WbOfSMX4a3EVXvlcqtOU9wH2ALoaF6JFqXTscVW10qr9uruje6H_Vq6qFqHl-a8f1DPzLSWNrPJDfwbjBjmfHMSiV8taw3k8nxkSS-vIOdSx0xszzL-VpvWhfSuOBj6fNYRKZ-FRVQ"
```

### 15.5 서비스어카운트 전용 kubeconfig 파일 배포


- kubeconfig 파일을 위한 certificate-authority-data 얻기
```{bash}
kubectl get secrets apiuser-token-xmdc2  -o jsonpath='{.data.ca\.crt}'
```

- kubeconfig 파일을 위한 token 얻기
```{bash}
kubectl get secrets apiuser-token-xmdc2  -o jsonpath='{.data.token}' | base64 -d
```

- new_config 파일 새성
> 아래 포맷에서 certificate-authority-data 와 token을 위의 출력 값으로 치환
> server 주소는 GKE의 마스터 주소
```
apiVersion: v1
kind: Config
clusters:
- name: default-cluster
  cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLakNDQWhLZ0F3SUJBZ0lRTXFCamlKS2wzajV4MmdGY3pyMjI4ekFOQmdrcWhraUc5dzBCQVFzRkFEQXYKTVMwd0t3WURWUVFERXlRNE16Z3lPREEyTmkxaFlUYzNMVFJsTWpndE9EQTNOaTFtWlRZNVkySXlNV1UyWVRZdwpIaGNOTWpFd05USXhNak0wT0RBMldoY05Nall3TlRJeE1EQTBPREEyV2pBdk1TMHdLd1lEVlFRREV5UTRNemd5Ck9EQTJOaTFoWVRjM0xUUmxNamd0T0RBM05pMW1aVFk1WTJJeU1XVTJZVFl3Z2dFaU1BMEdDU3FHU0liM0RRRUIKQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUUNXaENqTk9oZCtkN0NDbWZjSXF1SWFmRWJyRzkveFQvYmw4N3hoUFI2WQplcEdXdDI0K2p1alRYajVWM1A3bkZGVXh5R0ZJWWkvWTJZeG12Sm5rYzJaRnYxdXJtWm5VUU9XQ3BKV2kybnNjCnptUlRnMG1vTHdoSmlpNm1SMmVwY3dqTXpTNkJaNHYrNGxGd0Y0MUU1NWVWTllwbE5NZVRUVHpWclRMYW55aE8KNGNNWVh3RmdsS3NZOE9GZ0VMUzRIRHA4VldweFZyMlB6ZW9yNzJBSG1wV3J3S09MVFN1SW9tZW40VDdvdWhUNwpUY0ZWWDJJSHUwcER6UG9udUZEV1UxcFhxL0UyWC93bHdoZlp2aHJCbG5ySjNHWU1zTFhPU2N4eUJIUXRsemVUCndPM2E4NmxQY2IxblV4eDNYb1licThEQmErdmsxeVI0Mk5kalpIL3pDbU81QWdNQkFBR2pRakJBTUE0R0ExVWQKRHdFQi93UUVBd0lDQkRBUEJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJTOEp1MXFuVlN0S3B0MQpYT0U1TzJFZGVDaGprVEFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBSk5GcTRDd1BhWEk5bmx6emE0MkZXVElnCkFIU0NzaXB0NVJLMnlabmJVU21sb1p6cDMrWWVxVG40Q2tRSkEvT3l0U1h1V0RqcVh2WjlVZUVVcnBhelFCdUgKd1RhNVZPcFJOdW8xemx1VDgwVE5YaDlHS3FINFlXWUMzckZOd3AxVTRPWEhaR2RrQi9PYVA0d3ViTHBrQ3V3VAoyanU4VHorVG5wQ09lcDVJMWI1YnRQMXJtVUphVTErbFQwQUVrQk96YjhMNkVFbGZqVEQzYk9laU1TM1lEdjQrCk1PNDk0MjF0MCtGZi9JQi9vNEx5M050Z2krN1ZTRHplcExyb2lBc085bm96YzZ2L2M1VlpYQUhBR2J6d3pydmsKZlNOcTZqQ3BaWmNPRGJoZlpmTWl1dW9qaWtpc2lEYXR5c05EZkduNjVWWS9rT3B3cjJOM2tBc1g5MTNEOGc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://34.64.205.158
contexts:
- name: default-context
  context:
    cluster: default-cluster
    namespace: default
    user: default-user
current-context: default-context
users:
- name: default-user
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImVFWmZWR254MjJ2ektScGxhUFItUFRwOEEzNDE4UXZuTC1QQnBRcGY4aHcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFwaXVzZXItdG9rZW4teG1kYzIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYXBpdXNlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNkY2I4NDVhLWUzNzYtNGI4My1hZDMwLWU1NTgxMDkxZDY5MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmFwaXVzZXIifQ.z7tKPfJkzUVaYzgmk3HJiGeZU1BuWEhVLDS4_IWjvTnvG80VwTnAsosESDDxnSknp5zERatkH1gBtysi_BBe_GEXHMIFHwThnUi42ff7A3NNcXX1xuvkVDW_IxHMxoPLtzei1f-RQ_7ByAVy8cSBU2YdxRb3HFEZmxlKnMDXIqywoG_KOi_72rruvV94OY0FGsaMTftPZsG4WbOfSMX4a3EVXvlcqtOU9wH2ALoaF6JFqXTscVW10qr9uruje6H_Vq6qFqHl-a8f1DPzLSWNrPJDfwbjBjmfHMSiV8taw3k8nxkSS-vIOdSx0xszzL-VpvWhfSuOBj6fNYRKZ-FRVQ
```

- 새로운 new_config 파일로 kubectl 실행
```{bash}
kubectl --kubeconfig=./new_config get pods
```
