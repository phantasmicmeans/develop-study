spring boot 2.3이 최근 릴리즈 되었습니다. 릴리즈 노트를 살펴보다 몇가지 흥미로운 것들을 발견해 공유드립니다.

### Validation Starter no longer included in web starters
백기선님께서도 공유해주신 항목이고 조금 의외인 항목입니다. 
- [Do not include the validation starter in web starters by default](https://github.com/spring-projects/spring-boot/issues/19550)


spring-boot-starter-web 모듈 내에 validation 모듈이 존재했으나

spring boot 2.3 이상부터는 web 모듈에 validation 모듈이 존재하지 않습니다. 
따라서 직접 spring-boot-starter-validator 



### Graceful shutdown
Graceful shutdown은 다음 4개의 embedded server들에 지원됩니다. 

- jetty
- Reactor Netty
- Tomcat
- Undertow

또한 servlet-based stack / reactive stack 모두 지원합니다. 
이 stop processing은 특정 기간동안 새로운 request를 수용하지 않고 남아있는 request에 대해서만 처리를 가능케합니다!

Jetty, Reactor Netty, Tomcat은 network layer에서 모든 request accept 과정을 멈추고, Undertow의 경우 request는 accept 하나 503 response를 내린다고 하네요

다음과 같은 간단한 설정을 통해 graceful shutdown과 period를 설정할 수 있습니다

```
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=20s
```


### Liveness and Readliness probes

### JAVA 14 support 

### Automatic creation of developmentOnly Gradle Configuration 

### Build OCI images with Cloud Native Buildpacks

### Slice test for Web Services
