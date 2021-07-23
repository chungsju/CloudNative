# Learning Kubernetes

[toc]

##  18. Istio

### 18.1 Istio 설치하기

- istio 릴리즈 다운받기 (https://github.com/istio/istio/releases/tag/1.10.1)
- bin 폴더 Path 환경변수 추가 
- istio 설치
```{bash}
istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```

- Istio injection 활성화
```{bash}
kubectl label namespace default istio-injection=enabled
namespace/default labeled
```
> injection이 활성화된 네임스페이스에서 향후 생성되는 Pod에는 Istio Envoy 사이드카가 자동으로 구동됨

- 모니터링 툴 활성화
 ```{bash}
kubectl apply -f samples/addons
 ```

### 18.2 샘플 어플리케이션 설치

- 서비스와 디플로이먼트 설치
```{bash}
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

- Gateway 및 VirtualService 설치
```{bash}
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

- DestinationRule 설치
```{bash}
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

- Istio Gateway 외부 주소 확인
```{bash}
kubectl get service istio-ingressgateway -n istio-system
```

- 웹브라우저에서 http://gatewayIP/productpage 접속 확인


### 18.3 대쉬보드 접근

- 부하발생
특정 Pod에 접속(kubectl exec -it pod이름 /bin/bash) 하여 아래 명령어 실행
```
for i in $(seq 1 10000); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; sleep 0.1; done
```

- kiali 대쉬보드
```{bash}
istioctl dashboard kiali
```

- Grafana 대쉬보드
```{bash}
istioctl dashboard grafana
```

- Jaeger 대쉬보드
```{bash}
istioctl dashboard jaeger
```

### 18.3 가중치 기반 라우팅 

- Review V1과 V3을 각 50:50으로 라우팅


```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50

```

- kiali 대쉬보드를 통해 reviews 가 v1과 v3 5:5인지 확인
```{bash}
istioctl dashboard kiali
```

### 18.4 컨텐츠 기반 라우팅

- head에 end-user 값이 jason이 있는 요청은 reviews v2로 요청

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1

```

- productinfo 페이지에서 jason으로 로그인
리뷰 별이 검정색(v2)으로 나오는것을 확인


### 18.5 circuit breaker 테스트

- 샘플 타겟 서비스 생성

```
kubectl apply -f samples/httpbin/httpbin.yaml
```

- Circuit Beaker를 위한 DestinationRule 파일 생성 및 적용
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

- DestinationRule 적용 확인
```
kubectl get destinationrule httpbin -o yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
...
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 3m
      consecutive5xxErrors: 1
      interval: 1s
      maxEjectionPercent: 100
```

- 샘플 타겟을 호출할 Client Pod 생성
```
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
```

- Client Pod에서 동시 접속 부하 발생
```
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
20:33:46 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.3.1 running at 0 queries per second, 6->6 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 6] for exactly 20 calls (10 per thread + 0)
20:33:46 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
20:33:47 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
20:33:47 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 59.8524ms : 20 calls. qps=334.16
Aggregated Function Time : count 20 avg 0.0056869 +/- 0.003869 min 0.000499 max 0.0144329 sum 0.113738
# range, mid point, percentile, count
>= 0.000499 <= 0.001 , 0.0007495 , 10.00, 2
> 0.001 <= 0.002 , 0.0015 , 15.00, 1
> 0.003 <= 0.004 , 0.0035 , 45.00, 6
> 0.004 <= 0.005 , 0.0045 , 55.00, 2
> 0.005 <= 0.006 , 0.0055 , 60.00, 1
> 0.006 <= 0.007 , 0.0065 , 70.00, 2
> 0.007 <= 0.008 , 0.0075 , 80.00, 2
> 0.008 <= 0.009 , 0.0085 , 85.00, 1
> 0.011 <= 0.012 , 0.0115 , 90.00, 1
> 0.012 <= 0.014 , 0.013 , 95.00, 1
> 0.014 <= 0.0144329 , 0.0142165 , 100.00, 1
# target 50% 0.0045
# target 75% 0.0075
# target 90% 0.012
# target 99% 0.0143463
# target 99.9% 0.0144242
Sockets used: 4 (for perfect keepalive, would be 2)
Code 200 : 17 (85.0 %)
Code 503 : 3 (15.0 %)
Response Header Sizes : count 20 avg 195.65 +/- 82.19 min 0 max 231 sum 3913
Response Body/Total Sizes : count 20 avg 729.9 +/- 205.4 min 241 max 817 sum 14598
All done 20 calls (plus 0 warmup) 5.687 ms avg, 334.2 qps
```