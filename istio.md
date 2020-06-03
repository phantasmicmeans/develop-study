### Mesh Network
> 메시 네트워크(mesh network)는 각각의 노드가 네트워크에 대해 데이터를 릴레하는 네트워크 토폴로지이다. 모든 메시 노드들은 네트워크 내의 데이터 분산에 협업한다.  

<img src="https://user-images.githubusercontent.com/20153890/78812095-d7131a80-7a05-11ea-8354-689911ddc7eb.png" width="200">

출처 - wikipedia

## Service Mesh 
MicroService Architecture 시스템의 서비스간 통신이 Mesh Network 형태를 띄는것에 빗대어 명명 
MSA 시스템이 커지고 서비스가 점점 증가하고 관리가 어려워지고.. 통신, 메트릭, 보안 등등 서비스간의 상호 동작이 복잡해지고 있다. 

### A. 서비스가 몇개더라?
- 몇개의 서비스가 있지? 
- 서비스 당 인스턴스는 몇개나 존재하지? 
- 서비스 인스턴스에 로드밸런싱은 어떻게 하지? 
- 잘 떠있나?

### B. 저기서 장애가!
```A -> B -> 외부 API ```

- 외부 API 장애 start 
- B: 타임아웃 예정 스레드 증가 
- A: 그에 따른 B에 대한 타임아웃을 기다리는 스레드 증가 

어딘가에서 A를 호출하고 있다면? 쾅~ 장애 전파

### 문제 해결 
A를 풀자 - **Service Discovery Pattern**
- Registry에서 각 서비스 인스턴스 정보 관리 
- 인스턴스 정보를 꺼내서 로드밸런싱!
- Netflix Eureka + ribbon 조합 

B를 풀자 - **Circuit Breaker Pattern** 
- 일정 수치 이상 에러 발생 -> fallback method 실행 
- 장애 전파 차단 및 전체 시스템 보호 
- Netflix Hystrix 

### 이정도면 ? 
위 해결법을 적용하면 꽤 많이 좋아질 수 있다. 그러나 
- 이것들을 적용하려면 개발자가 다 해야한다. 
- 결국 내 애플리케이션이 위를 지원해야하고, 또 다른 애플리케이션도 모두 지원되어야 한다. 
- 내 바람대로 동작 할 소프트웨어가 전부 구축 되어야 한다. 

이를 인프라 레벨에서 컨트롤 할 수 있다면 좋지 않을까

즉 Service Mesh란, software적 접근이 아닌 인프라 레벨에서 서비스간 통신 및 그 외 여러 부분들을 담당하고 관리하자는 컨셉이다.

그럼 어떤식으로 인프라 레벨에서 이를 관리할까 

### Proxy 
Service Mesh 컨셉은 서비스 간 호출의 경우 / 서비스간 라우팅의 경우 직접 호출하는 것이 아닌 프록시를 두고 아래와 같이 진행한다. 

서비스간 | 라우팅 
:--------:|:--------:
<img src="https://user-images.githubusercontent.com/20153890/78814421-8e5d6080-7a09-11ea-9b24-d582d2447f8d.png" width=350> | <img src="https://user-images.githubusercontent.com/20153890/78817302-eeee9c80-7a0d-11ea-9322-d0f826fe2612.png" width=450>

이를 통해 각 서비스로 들어오는 트래픽을 proxy가 통제하고 차단 할 수 있다. 위에서 B상황의 경우 circuit breaking을 이 proxy가 행할 수 있다.

#### Data Plane & Control Plane 
서비스의 증가하면 프록시 수도 증가하게 된다. 프록시 수의 증가는 또다시 관리 포인트에 대한 비용을 생성한다. 
이 문제를 해결하기 위해 각 프록시에 대한 config 정보를 중앙에서 컨트롤 하는 구조를 취해야한다. 

<img src="https://user-images.githubusercontent.com/20153890/78818596-e008e980-7a0f-11ea-950a-a50423afaf8a.png" width=300>

Data Plane은 각 프록시들로 이루어져 config에 따라 트래픽을 컨트롤 하는 부분을 말하고,
Control Plane은 Data plane의 config 값들을 저장하고, 프록시들에 이를 전달하는 컨트롤러 역할을 한다. 

### Envoy Proxy 
Istio는 envoy 프록시를 사용한다. 이는 Lyft사에 의해 개발 되었으며 다양한 기능을 제공한다.

- TCP, HTTP1, HTTP2, gRPC protocol 지원 
- TLS client certification 
- L7 라우팅 지원 및 URL 기반, 버퍼링, 서버간 부하 분산량 조절
- Auto retry / Circuit Breaker 지원 / 다양한 로드밸런싱 기능 제공 
- Zipkin을 통한 분산 트랜잭션 성능 측정 제공
- Dynamic configuration 지원, 중앙 레지스트리에 설정 및 설정 정보를 동적으로 읽어옴
- MongoDB에 대한 L7 라우팅 기능

서비스들은 Envoy proxy와 함께 Data Plane을 구성할 수 있다.

&nbsp;

## Istio 
Envoy Proxy로 구성된 Data Plane을 컨트롤 하는 것이 Istio이다. k8s에서는 **envoy proxy가 pod에 sidecar 형태로 배포된다.** istio-sidecar-injector가 자동으로 istio(envoy) proxy를 pod에 주입한다. 
 
**istio architecture**
<img src="https://user-images.githubusercontent.com/20153890/78819538-5e19c000-7a11-11ea-8e80-d0ec4bd4cbaa.png" width=600>

istio의 core component는 아래와 같다.
- Envoy 
- Pilot 
- Citadel 
- Gallery 

#### Pilot 
- Envoy sidecars를 위한 service-discovery를 제공 
- Istio에 배포된 envoy 의 생명 주기를 담당하며, 각 envoy는 pilot으로부터 가져온 다른 인스턴스 정보들로 로드밸런싱을 하게 된다.
- Traffic Management API 를 사용해 Pilot이 envoy proxy가 더 세밀한 구성을 할 수 있게 도와준다. 
    - traffic management capabilities for intelligent routing
    - retry, circuit breaker, timeout 등 

<img src="https://media.oss.navercorp.com/user/16518/files/9529b900-7a85-11ea-88b6-a6a615e3fd50" width=300>

- 새로운 인스턴스가 생성되면 platform adapter에게 알림
- 인스턴스가 생성 되었으니 트래픽 규칙과 구성 변경을 알리기 위해 envoy proxy들에 이를 알림 
#### Citadel
- 보안에 관련된 기능을 담당하는 모듈 
- 서비스를 사용하기 위한 사용자 인증 및 인가 담당 
- Istio는 TLS를 이용해 통신할 수 있는데, 이때 certification을 관리하는 역할을 한다. 

#### Gallery  
- provides configuration management services for Istio.

&nbsp;

## Traffic Management 
Istio는 pilot(service discovery)과 envoy proxy를 활용해 트래픽을 컨트롤 하게 된다. 

pilot을 통해 envoy proxy는 트래픽을 관련 서비스로 전달 한다. 또한 기본적인 디스커버리와 로드밸런싱 외에 트래픽의 특정 비율을 새 버전의 Pod으로 보내는 등 더 많은 세밀한 규칙을 지정 할 수 있다.

아래 그림이 Istio의 Traffic Management feature이다. 

<img src="https://media.oss.navercorp.com/user/16518/files/7a167380-7a9c-11ea-878c-1870a787f2c3">

출처 - https://istio.io/docs/tasks/traffic-management/

&nbsp;

위 feature 들을 활용하려면 필수적으로 알아야 할  것들이 있는데 아래와 같다.

- Gateway
- Virtual Service 
- DestinationRule 

위 3개의 요소들을 더 이해하기 편하도록 도식도를 그렸다. 글을 다 읽고 보면 더 이해가 잘 될지도...?

![스크린샷 2020-04-14 오후 5 12 38](https://media.oss.navercorp.com/user/16518/files/2bd5eb80-7e73-11ea-8d68-23b8607cd8b4)

&nbsp;

### Gateway 
Gateway는 외부로부터 트래픽을 받으며 가장 앞단에 존재한다. mesh에 대한 inbound 및 outbound traffic을 관리하여 mesh에 진입하거나 나갈 트래픽을 지정할 수 있다. gateway는 서비스에 붙는 envoy 처럼 sidecar 로 실행 되는 것이 아니고, standalone envoy proxy 형태로 생성된다. 

Istio를 사용한다고 해서 외부에서 들어오는 트래픽을 반드시 Istio gateway를 사용해야 하는 것은 아니다. 기존의 Kubernetes Ingress를 그대로 사용할 수 도 있다.

아래 예시는 ext-host.example.com 로 https traffic 을 포트 443을 통해 mesh로의 접근을 허용한다. 

```yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

gateway를 virtual service에 바인딩 하여야 의도한 대로 동작한다. routing rule은 아래 virtual service에 정의하여야 한다. 

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
  - ext-host-gwy
``` 

&nbsp;

### Virtual Service

istio traffic routing 기능의 핵심 구성 요소이고, 이스티오의 트래픽 관리를 유연하게 만드는 핵심적인 역할을 한다.  

virtual service는 k8s service로 요청이 라우팅 되는 방법을 구성할 수 있다. 또한 워크로드에 트래픽을 보내기 위한 다양한 트래픽 라우팅 규칙을 지정하는 방법들을 제공한다.

virtual service 가 없는 경우, envoy는 모든 서비스 인스턴스 간에 round robin 로드 밸런싱을 사용해 트래픽을 분산 시키다. 그러나 [여기](https://istio.io/docs/ops/best-practices/traffic-management/#set-default-routes-for-services) 모든 서비스에 virtual service를 적용시키 것이 best라고 한다. 

client는 virtual service에 요청을 보내고, envoy는 virtual service rule에 따라 각기 다른 버전으로 라우팅 하는 형태이다. (ex. 20%는 새 버전, 80%는 이전 버전으로 라우팅) 

이를 통해 새 서비스 버전으로 전송되는 트래픽의 비율을 점차적으로 늘릴 수 있다.(아래에서 다시 설명) 또한 http/1.1, http2, grpc, tcp 까지 라우팅 규칙을 구성할 수 있다. 

아래는 2가지 개별 서비스에 트래픽을 보내는 예시이다. /reviews -> reviews, /ratings -> raintings 
```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
```
이 외에도 더 자세한 사항 : https://istio.io/docs/reference/config/networking/virtual-service/

&nbsp;

### Destination Rule 
Virtual Service 와 함께 destination rule은 이스티오의 트래픽 라우팅 기능의 핵심 부분이다. 
virtual service가  k8s service로 트래픽을 보내도록 라우팅 했다면, destination rule은 그 서비스 내에서 어느 Pod으로 트래픽을 보낼지, 그리고 어떻게 트래픽을 보낼 것인지 에 대한 정의이다.

Random / Weighted / Least requests 
- https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/load_balancing

아래 예시는 v1, v3는 default load balancer, v2는 roung robin load balancer를 설정한다. 
그리고 reviews destination service에 대해 세가지 하위 subset을 구성한다. 

```yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM  // default는 랜덤으로 정의 
  subsets:
  - name: v1
    labels:
      version: v1 // pod label 
  - name: v2
    labels:
      version: v2 // // pod label 
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN // v2 subset 경우 round robin 
  - name: v3
    labels:
      version: v3 // // pod label 
```

이렇게 destination rule이 subsets을 가지고 있으면, virtual service 구성에 아래처럼 subset 별로 weight를 줄 수 있다. 

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews 
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 10
    - destination:
        host: reviews
        subset: v2
      weight: 80
    - destination:
        host: reviews
        subset: v3
      weight: 10
```


&nbsp;

### Network Resilence and testing
istio는 mesh의 장애를 방지하기 위해 failure-recovery 및 fault-injection 기능을 제공한다. 
 
#### timeout
envoy proxy 가 응답을 기다리는 시간으로 http의 경우 default는 15초이다. 아래는 rating service 의 v1에 대한 호출의 timeout을 10초로 지정하고, v2에 대해 retry 시도를 3번으로 설정한다. 

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
  - route:
    - destination:
        host: ratings
        subset: v2
    retries:
      attempts: 3
      perTryTimeout: 2s // 연결 시간 
```

#### circuit breakers
limit에 도달하여 circuit breaker 발동시 해당 host에 대한 연결을 중지시킨다. 아래는 예시이다.
maxConnection & http1MaxPendingRequests & maxRequestsPerConnection이 모두 1로 정의 되어 있는 상태이고, circuit open시 'outlier detection 이라 불리는 행위를 취한다. 
```yml
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
      consecutiveErrors: 1 // error 1회 발생시 -> unhealthy 
      interval: 1s // 1초간 
      baseEjectionTime: 3m // 3분간 로드밸런싱 풀에서 제거 
      maxEjectionPercent: 100 // 모든 인스턴스를 제거할 수 있음. envoy의 default option은 10 
``` 

### fault injection
fault injection은 시스템에 오류를 도입해 상태를 견디고 복구할 수 있도록 하는 test 방법이고 다음과 같은 행위를 할 수 있다.

- delays : network latency가 증가하거나 오버로드 된 unstream을 흉내낸다. 
- aborts : 일반적인 http error나 tcp connection failures를 나타낸다. 

아래 virtual service는 rating 서비스에 대해, 1000건의 요청 중 1건에 대해 5초 지연을 도입한다.

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

참고자료 : [compare hystrix & istio circuit breaking](https://blog.christianposta.com/microservices/comparing-envoy-and-istio-circuit-breaking-with-netflix-hystrix/)

### kiali 
- [product page](http://10.105.93.69/productpage)
- [kiali console](http://10.105.231.10/kiali/console/graph/namespaces/?edges=noEdgeLabels&graphType=versionedApp&namespaces=media-one-devops&unusedNodes=true&injectServiceNodes=true&pi=10000&duration=600&layout=dagre)
- ID/PWD - admin/admin
- [istio example](https://istio.io/docs/examples/bookinfo/)을 활용 및 변형 적용  
- Virtual Service 적용 - productpage / reviews 
- DestinationRule - reviews -> (v1 - weight:10, v2 - weight: 80, v3 - weight: 10)

![스크린샷 2020-04-14 오후 6 11 09](https://media.oss.navercorp.com/user/16518/files/5461e380-7e7b-11ea-8fb5-ed0d797c8a52)



### ref
- https://istio.io/docs/concepts/what-is-istio/
- https://istio.io/docs/ops/deployment/architecture/ 
- https://istio.io/docs/concepts/traffic-management/
- https://istio.io/docs/reference/config/networking/destination-rule/
- https://istio.io/docs/reference/config/networking/virtual-service/
- https://istio.io/docs/reference/config/networking/gateway/
