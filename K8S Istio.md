# Learning Kubernetes

[toc]

## 19. 최종 연습문제

### 19.1 PHP GuestBook with Redis

#### 19.1.1 과제 개요

![방명록 GKE 다이어그램](https://cloud.google.com/kubernetes-engine/images/guestbook_diagram.svg?hl=ko)


- 레디스 마스터 구성
- 레디스 슬레이브 구성
- Guestbook 프론트앤드 구성
- 프로트앤드 애플리케이션 서비스로 노출 시키기
- 리포지토리 만들고  ArgoCD 연동하기

#### 19.1.2 레디스 마스터 Deployment 로 구성

| 항목                               | 값                                        |
| ---------------------------------- | ----------------------------------------- |
| 파일명                             | redis-leader-deployment.yaml              |
| Deployment name                    | redis-leader                              |
| Deployment Label                   | app: redis                                |
| deployment Selector / pod metadata | app: redis / role: leader / tier: backend |
| replica                            | 1                                         |
| Image                              | docker.io/redis:6.0.5                     |
| Port                               | 6379                                      |

> kubectl logs -f POD-NAME

#### 19.1.3 레디스 마스터 서비스 구성

| 항목              | 값                                        |
| ----------------- | ----------------------------------------- |
| Type              | ClusterIP                                 |
| Service 이름      | redis-leader                              |
| Service Label     | app: redis / role: leader / tier: backend |
| port / targetPort | 6379 / 6379                               |
| selector          | app: redis / role: leader / tier: backend |

#### 19.1.4 레디스 슬레이브 Deployment 구성

| 항목                  | 값                                          |
| --------------------- | ------------------------------------------- |
| 파일명                | redis-follower-deployment.yaml              |
| Deployment Name       | redis-follower                              |
| Deployment Label      | app: redis                                  |
| Deployment matchLabel | app: redis / role: follower / tier: backend |
| Replicas              | 2                                           |
| Pod Label             | app: redis / role: follower / tier: backend |
| Container name        | follower                                    |
| Image                 | gcr.io/google_samples/gb-redis-follower:v2  |
| containerPort         | 6379                                        |


#### 19.1.5 레디스 슬레이브 서비스 구성

| 항목              | 값                                        |
| ----------------- | ----------------------------------------- |
| Type              | ClusterIP                                 |
| Service 이름      | redis-slave                               |
| Service Label     | app: redis / role: slave / tier: backend  |
| port / targetPort | 6379 / 6379                               |
| selector          | app: redis / role: slaver / tier: backend |

#### 19.1.6 GuestBook 애플리케이션 Deployment 생성

| 항목                      | 값                                   |
| ------------------------- | ------------------------------------ |
| 파일명                    | frontend-deployment.yaml             |
| Deployment Name           | frontend                             |
| Deployment Label          | app: guestbook                       |
| Deployment matchLabel     | app: guestbook / tier: frontend      |
| Replicas                  | 3                                    |
| Pod Label                 | app: guestbook / tier: frontend      |
| Container name            | php-redis                            |
| Image                     | gcr.io/google_samples/gb-frontend:v5 |
| 환경변수 : GET_HOSTS_FROM | "dns"                                |
| containerPort             | 80                                   |

#### 19.1.7 GuestBook 애플리케이션 LoadBalancer 서비스 구성

| 항목          | 값                              |
| ------------- | ------------------------------- |
| Service name  | frontend                        |
| service Label | app: guestbook / tier: frontend |
| Type          | LoadBalancer                    |
| Port          | 80                              |
| selector      | app: guestbook / tier: frontend |

#### 19.1.8 FrontEnd 앱 스케일링 해보기

replica 를 5개로 스케일링 해보기

#### 19.1.9 해당 메니페스트 파일을 Argo CD에 연동하여 배포하기