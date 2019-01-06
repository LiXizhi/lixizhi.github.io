---
layout: post
title:  "Prometheus #2 - Prometheus와 Grafana를 이용한 Spring Boot 2.0 어플리케이션 모니터링"
date:   2018-12-29 15:14:54
categories: prometheus
comments: true
---
* content
{:toc}

이번 글에서는 `Prometheus`를 이용해 `Spring Boot` 메트릭을 수집하고 `Grafana`로 시각화하는 실습을 포스팅할 것이다.
<br>

### 1. Prometheus가 메트릭을 수집할 수 있도록 Spring Boot Application 작성
SpringBoot 2.0이상부터는 `Micrometer`라는 메트릭 엔진을 지원한다. 이 글에서는 `Micrometer`를 사용하여 모니터링 메트릭을 생성할 것이다. `Micrometer`에 대한 자세한 설명은 [링크][Micrometer-Describe]를 참고하기 바란다. 그리고 `Spring Actuator`로 메트릭들을 Prometheus가 가져갈(Pull)수 있도록 Http EndPoint를 노출시킬 것이다. 

`pom.xml`에 다음과 같이 dependency를 추가한다.

```java
<!-- 메트릭을 Endpoint로 노출 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Micrometer core dependecy  -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
</dependency>
<!-- Micrometer Prometheus registry  -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
<br>

`application.properties`에 다음의 설정을 추가한다.

```java
management.endpoint.metrics.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.prometheus.enabled=true
management.metrics.export.prometheus.enabled=true
```
<br>

이제 Application을 실행하고 `http://localhost:8080/actuator`로 아래와 같이 Prometheus 메트릭 노출 Endpoint가 존재하는지 확인한다.

![prometheus-execute-01](https://user-images.githubusercontent.com/19832483/50539347-8423a400-0bc2-11e9-8657-02583d7c4237.png){: .u-mid-img}
<br><br>

노출된 Endpoint Link인 `http://localhost:8080/actuator/prometheus`로 아래와 같이 Prometheus가 수집할 Metric들이 노출되는지 확인한다.

![prometheus-execute-02](https://user-images.githubusercontent.com/19832483/50539349-8685fe00-0bc2-11e9-96f8-bafbc3918496.png){: .mid-img}
<br>

위의 설정만으로도 `Micrometer`와 `Actuator`가 기본적인 메트릭을 노출시키지만, 실서비스에서는 주로 `Custom Metric`을 측정하여 모니터링 하기때문에 이를 간단하게 작성해보자. 코드는 아래와 같다.
```java
@SpringBootApplication
@RestController
public class PrometheusDemoApplication {

	@Autowired
	private MeterRegistry meterRegistry;

	private Counter counter;

	@PostConstruct
	public void init() {
		counter = meterRegistry.counter("api.call.count");
	}

	public static void main(String[] args) {
		SpringApplication.run(PrometheusDemoApplication.class, args);
	}

	@GetMapping("/test")
	public String test() {
		counter.increment();
		return "test";
	}
}
```
위의 코드를 보면 Spring에서 자동 설정해준 `MeterRegistry`를 Bean으로 주입받고, `Counter` 메트릭을 만들어낸다. `Counter`안의 문자열 `api.call.count`이 변형되어 `api_call_count_total`가 메트릭 이름이 된다. `/test` HTTP API를 호출하면 해당 `Counter`는 증가한다. 이 외에도 많은 종류의 메트릭과 태그를 작성할 수 있다.

Spring Boot Application을 재실행하고, `http://localhost:8080/actuator/prometheus` 요청 후 응답으로 아래의 사진과 같이 `api_call_count_total` 메트릭이 나타나는지 확인한다.

![prometheus-execute-03](https://user-images.githubusercontent.com/19832483/50545449-4fadf780-0c57-11e9-88a7-a7fb22eb9a53.png)
<br><br>

위의 과정을 모두 마쳤다면, Spring Boot Application에서 Metric을 Endpoint로 노출시키는 작업은 끝난것이다.
<br><br>

### 2. Prometheus 설치 및 실행
여기서는 `설치 파일을 받아서 실행하는 방법`과 `docker를 이용해서 실행하는 방법` 두 가지를 소개하겠다.  

#### 1. 설치 파일을 받아서 실행
[링크][Prometheus-Install]에서 각 OS에 맞는 파일을 받는다. 필자는 Mac OS이므로 `prometheus-{version}.darwin-amd64.tar.gz`를 다운로드하였다. 파일을 원하는 위치에 받고, 압축을 푼다.
압축을 풀고 해당 디렉토리 안을 보면 `prometheus.yml`이란 설정 파일이 있다. 해당 파일의 내용을 다음과 같이 수정하자.
```yaml
global:
  scrape_interval:     10s # 10초마다 Metric을 Pulling
  evaluation_interval: 10s
scrape_configs:
  - job_name: 'spring-app-demo'
    metrics_path: '/actuator/prometheus' # 위에서 작성한 Spring Application에서 노출시킨 메트릭 경로를 입력한다.
    static_configs:
      - targets: ['localhost:8080'] # 해당 타겟의 도메인과 포트를 입력한다.
```

<br>
설정 파일 작성이 완료되었으면, 이제 다음의 명령어로 Prometheus를 실행한다.
```console
$ nohup ./prometheus &
```

<br>
`http://localhost:9090` 요청시 아래와 같이 웹 페이지가 보이면 Prometheus Server가 성공적으로 실행된 것이다.

![prometheus-execute-04](https://user-images.githubusercontent.com/19832483/50539350-884fc180-0bc2-11e9-8594-1477f6e28ba2.png){: .mid-img}

<br>
아래의 그림처럼 웹 페이지에서 수집된 메트릭 항목을 하나 선택후에 `Graph` 탭을 누르고, `Execute` 버튼을 누르면 메트릭에 해당하는 그래프가 보여질 것이다.

![prometheus-execute-05](https://user-images.githubusercontent.com/19832483/50539351-8980ee80-0bc2-11e9-99d3-76648e5dd606.png){: .u-mid-img}
<br>
<br>
#### 2. Docker를 이용해서 실행
필자의 경우 Docker를 이용해서 실행하는 방법은 생각대로 잘 되지 않았다. Prometheus Container에서 메트릭을 수집하기 위해서는 Host의 Application 쪽으로 Http 요청을 보내야 하는데, Docker의 특성상 기본적으로 Container에서 Host로 접근이 불가능하다. 접근을 가능하게 하려면 Mac OS 기준으로 Docker Container에서 도메인을 `host.docker.internal`로 요청을 해야한다고 [Docker 문서][Docker-MacOS-Networking]에 명시되어 있다. 필자는 이 방법을 사용해 봤지만 Prometheus Container에서 Host Application으로 요청시 EOF 에러가 발생하는데, 이 원인에 대해서는 나중에 분석할 예정이다.(몇 시간동안 삽질을 해봤지만 소득이 없었다..) 필자는 Prometheus Docker Image에서 발생하는 문제라고 보고 있다. 혹시나 이런 문제를 해결하신 분이 이 글을 본다면 어떻게 해결했는지 댓글을 남겨주시면 좋을것 같다.

Container를 실행하기에 앞서 Prometheus의 설정파일을 쉽게 관리하기 위해 Prometheus Container안에 있는 prometheus.yml을 Host의 prometheus.yml과 마운트 시킬것이다. Host에서 설정파일을 관리할 디렉토리를 만들고 다음과 같이 prometheus.yml을 작성한다.

```yml
global:
  scrape_interval:     10s # 10초마다 Metric을 Pulling
  evaluation_interval: 10s
scrape_configs:
  - job_name: 'spring-app-demo'
    metrics_path: '/actuator/prometheus' # 위에서 작성한 Spring Application에서 노출시킨 메트릭 경로를 입력한다.
    static_configs:
      - targets: ['host.docker.internal:8080'] # 해당 타겟의 도메인과 포트를 입력한다.
```
<br>

이제 위의 작성한 파일을 Container와 공유하면서 Prometheus를 실행할 수 있도록 다음의 명령어를 수행한다.

```console
$ docker run -p 9090:9090 -v {생성한 디렉토리}/prometheus.yml:/etc/prometheus/prometheus.yml --name prometheus -d prom/prometheus --config.file=/etc/prometheus/prometheus.yml
```
`http://localhost:9090` 요청시 웹페이지가 보이면 Prometheus Server가 성공적으로 실행된 것이다. 필자는 위에서 말했듯이 Prometheus Container는 잘 동작하지만, 외부의 Metric 수집이 되지 않는다.
<br><br>

### 3. Prometheus가 수집한 데이터를 Grafana로 시각화하기
Prometheus의 웹 페이지에서 쿼리를 수행하여 우리가 원하는 메트릭을 그래프로 시각화할 수 있다. 하지만 매번 모니터링을 위해 수동으로 쿼리를 수행하는 것은 비효율적이고 시각화하는데 한계가 있기 때문에 보통 시각화 도구를 이용해서 모니터링할 메트릭 항목을 선정하여 시각화한다. 여기서는 `Grafana`를 이용해서 모니터링을 위해 메트릭들을 시각화하는 방법을 소개할 것이다.

필자는 `Docker`를 이용해서 `Grafana`를 설치하였다. 이 포스트에서는 `Grafana`가 이미 설치되었다고 가정하고 다음의 과정을 진행하겠다.

`Grafana`에 우리가 사용하는 Prometheus `Data Source`를 추가하기 위해 `Configuration` -> `Data Source` -> `Add DataSource` -> `Prometheus`순으로 UI를 클릭한다. 그리고 아래와 같이 설정한다.

![prometheus-execute-06](https://user-images.githubusercontent.com/19832483/50545454-6e13f300-0c57-11e9-9226-37c8f45ec343.png){: .u-mid-img}

위 사진의 `HTTP`항목을 보면 필자는 요청 도메인을 `host.docker.internal`로 설정하였는데, [2-2](#2-docker를-이용해서-실행)에서 설명했던 이유로 해당 도메인을 사용하였다.(다행히도 grafana는 잘 동작한다.)

이제 위에서 추가한 Data Source를 이용해서, 현재 측정하고 있는 메트릭인 `api_call_count_total`를 그래프로 시각화해보자. 아래와 같이 설정한다.

![prometheus-execute-07](https://user-images.githubusercontent.com/19832483/50545456-8b48c180-0c57-11e9-8815-7dd43eae9b7b.png){: .u-mid-img}
<br><br>

그럼 아래의 그림처럼 시각화된 그래프가 보일것이다.

![prometheus-execute-08](https://user-images.githubusercontent.com/19832483/50545457-8edc4880-0c57-11e9-8507-034bd4bde07a.png){: .u-mid-img}
<br><br>

우리는 위 과정을 통해 `Spring Boot Application`을 작성해 메트릭을 노출시키고, `Prometheus`를 설치 및 설정하고, `Grafana`로 시각화하는 작업까지 모두 마쳤다. 실제로는 소개하지 않은 더 많은 유용한 기능들이 있다. 필요한 기능이 있으면 문서를 통해 찾아보면 될 것이다. 이 글이 Prometheus를 접해보지 않은 분들께 많은 도움이 되길 바란다.

이상 포스팅을 마치겠다.

[Micrometer-Describe]:https://dzone.com/articles/using-micrometer-with-spring-boot-2
[Prometheus-Install]:https://prometheus.io/download/
[Docker-MacOS-Networking]:https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds