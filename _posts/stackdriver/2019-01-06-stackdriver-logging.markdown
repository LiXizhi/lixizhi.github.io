---
layout: post
title:  "Stackdriver #2 - Stackdriver Logging을 이용한 분산 로그 수집 및 애플리케이션 모니터링"
date:   2019-01-06 00:14:54
categories: stackdriver
comments: true
---
* content
{:toc}

이번 포스트에서는 `Stackdriver Logging`을 이용하여 어플리케이션에서 로그를 수집하고 검색하는 방법을 소개할 것이다. 그리고 수집된 로그를 기반으로 메트릭을 생성 및 시각화(대시보드)하고 모니터링 알람을 설정하는 방법도 소개한다. GCP를 이용하는 방법은 쉽고 다른 블로그에서 많이 설명하므로, 여기서는 `On Premise` 환경을 기준으로 설명하겠다. (사실, 둘의 차이는 GCP-API 사용자 인증 정보 생성, 적용하는 방법이 다른것 밖에 없다.)

## Stackdriver Logging 소개
`Stackdriver Logging`은 클라우드(GCP, AWS) 및 On-Prem 환경의 애플리케이션의 로그를 실시간으로 수집한다. 보통 애플리케이션의 로그를 대용량으로 한 곳에서 수집하고 가공할 때 Log Producer->Message Q->Log Consumer->Log Storage->Reporting 과정을 거친다. 위 단계와 관련된 인프라 및 애플리케이션을 구축하는건 많은 비용이 들고 운영시 관리 포인트가 늘어나게 된다. 하지만 Stackdriver를 사용하면 Log Producer는 Stackdriver에서 제공해주는 라이브러리 및 에이전트를 사용하고, 그 뒷 단계는 Stackdriver에 이미 구축이 되어있으니 이에 대한 구축 및 관리 비용이 줄게된다. 

수집된 로그는 최대 `30일` 동안 Stackdriver에 저장되며 `로그 뷰어`를 이용해 다양한 필터조건으로 검색이 가능하고 `Stackdriver Monitoring`과 연동하여 시각화 대시보드 및 알림에 포함시킬 로그 기반의 측정항목(Metric)을 정의할 수 있다. 또한 영구적인 보관을 위해 로그를 다른 저장소로 내보내는 것이 가능하다.
<br><br>

## 1. GCP-API 사용자 인증 정보 생성 및 적용
On-Prem 환경의 호스트 및 VM에서 Stackdriver Logging을 사용하기 위해서는 GCP API 인증이 필요하다. 이를 위해 다음의 단계를 수행한다. (여기서는 GCP 프로젝트가 이미 생성되어 있다고 가정한다.)
<br><br>

`API 및 서비스` -> `사용자 인증 정보`를 클릭한다.
<br>
![stackdriver-auth-01](https://user-images.githubusercontent.com/19832483/50733255-cefa7880-11cd-11e9-8c6a-961626789877.png){: .mid-img}
<br><br>

`사용자 인증 정보 만들기` -> `서비스 계정 키`를 클릭한다.
<br>
![stackdriver-auth-02](https://user-images.githubusercontent.com/19832483/50733256-d02ba580-11cd-11e9-8ec9-2ec17186d603.png){: .u-mid-img}
<br><br>

새 서비스 계정을 만들고, 역할에는 `로그 기록 관리자`를 선택한다. 단순히 Stackdriver에 log를 전송하는 VM에서는 로그 기록에 관한 권한만 필요하므로, 보안상 필요한 역할만 추가한다. 키 유형은 JSON을 선택한다.
<br>
![stackdriver-auth-03](https://user-images.githubusercontent.com/19832483/50733257-d1f56900-11cd-11e9-840c-e8aa2e5b447e.png){: .u-mid-img}
<br><br>

계정 생성 버튼을 클릭하면, 비공개 키를 가진 json 설정파일이 자동으로 다운로드 받아지는데, 이를 로그 전송할 호스트 및 VM의 어느 위치에 복사한다. 그리고 그 파일의 위치를 `GOOGLE_APPLICATION_CREDENTIALS`라는 이름을 가진 환경변수로 세팅한다.
```console
 export GOOGLE_APPLICATION_CREDENTIALS={다운로드 받은 파일이 존재하는 위치}
```
그러면 GCP 관련 라이브러리를 사용할 때 이 환경변수를 이용하여 인증관련 정보를 가져와 사용하게 된다. `Intellij`와 같은 IDE를 사용할 경우 환경변수 설정 후 IDE를 다시 시작해 주어야 IDE가 환경변수를 인식할 수 있다. 
<br><br>

## 2. Stackdriver 로깅
로깅을 하는 방법에는 대표적으로 `로깅 라이브러리`를 사용하는 방법과 `로깅 에이전트`를 사용하는 방법이 있다.

`로깅 라이브러리`는 각 언어에 제공되는 라이브러리를 사용하여 애플리케이션에서 직접 `grpc`로 Stackdriver에 로그를 전송하는 방법이다.

`로깅 에이전트`는 비즈니스 로직을 처리하는 애플리케이션과 독립적으로 `fluentd`에 기반을 둔 애플리케이션을 실행하여 이미 작성된 로그 파일을 이용하여 Stackdriver에 로그를 전송하는 방법이다.

필자는 로깅 에이전트를 On-Prem 환경에서 실행해보았는데, 정상적으로 동작하지 않았다. 로깅 에이전트를 사용하려면 fluentd 설정을 바꿔주어야 하므로, 여기서는 `SpringBoot 2.x`를 이용해서 Application을 작성하고, `로깅 라이브러리`로 로그를 전송하는 방법을 소개하겠다.
<br><br>

### 로깅 라이브러리를 이용하는 방법
로깅 라이브러리를 이용하는 방법에도 `logback`을 이용하는 방법과 `Stackdriver Logging Client Libraries`를 이용하는 두 가지 방법이 있다. 두 방법 모두 이 글에서 설명하겠다.

시작하기 앞서 다음과 같이 `pom.xml`에 의존성을 추가한다.
```xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>google-cloud-logging-logback</artifactId>
    <version>0.73.0-alpha</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```
`lombok` 의존성을 추가하는 이유는 `@slf4j` 어노테이션을 사용해 편리하고 깔끔하게 Logger 객체를 가져오기 위함이다.
<br><br>

#### logback을 이용하는 방법
`resource` 디렉토리에 logback 설정 파일인 `logback-spring.xml` 파일을 만들고, 아래의 내용을 복사한다. 구글 클라우드 로깅 라이브러리에서 제공하는 `com.google.cloud.logging.logback.LoggingAppender`를 이용해 logback 설정 파일을 작성하는 것이다. 여기서는 로그 레벨이 `ERROR` 이상인 로그들만 전송하도록 설정하겠다.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <appender name="CLOUD" class="com.google.cloud.logging.logback.LoggingAppender">
        <!-- 발생하는 로그들 중 로그 수준을 필터링하여 전송
            여기서는 로그 레벨이 ERROR 이상인 로그들만 전송
         -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>

        <!--Stackdriver에서 분류될 로그 종류-->
        <log>application.log</log> <!-- Optional : default java.log -->

        <!-- 로그를 전송하는 배치가 바로 flush할 기준 -->
        <flushLevel>ERROR</flushLevel> <!-- Optional : default ERROR -->
    </appender>

    <root level="INFO">
        <appender-ref ref="CLOUD" />
    </root>
</configuration>
```
<br>

테스트할 애플리케이션을 작성해보자. 애플리케이션 구동시 `DEBUG`, `INFO`, `ERROR` 로그를 기록하는 `ApplicationRunner.class`를 다음과 같이 작성한다.
```java
@Slf4j
@Component
public class ApplicationRunner implements org.springframework.boot.ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {

        log.debug("Application Started - {}", "debug");
        log.info("Application Started - {}", "info");
        log.error("Application Started - {}", "error");
    }
}
```
<br>

이제 Application을 실행해보자. 로그 레벨이 Error 이상인 경우 전송하기로 설정했으므로, **Application Started - error**라는 내용을 가진 로그만 전송될 것이다. 하지만 섬세한 로깅을 하기 위해서는 logback으로 한계가 있다.
<br><br>

#### 로깅 라이브러리를 이용하는 방법
로깅 라이브러리를 사용하면 약간 번거롭지만 섬세한 로깅이 가능하다. 예를들어 json 형식의 payload도 전달이 가능하고, labeling도 가능하다. 이번에는 다음과 같이 코드를 작성하여 JUnit으로 테스트 해보자.
```java
@RunWith(JUnit4.class)
public class StackDriverLibraryTest {

    @Test
    public void testLogging() throws Exception {
        Logging logging = LoggingOptions.getDefaultInstance().getService();

        String logName = "test-log";

        String text = "Hello, world!";

        LogEntry entry = LogEntry.newBuilder(Payload.StringPayload.of(text))
                .setSeverity(Severity.INFO)
                .setLogName(logName)
                .addLabel("userId", "asdf1234")
                .setResource(MonitoredResource.newBuilder("global").build())
                .build();

        logging.write(Collections.singleton(entry));
        Thread.sleep(2000);
    }
}
```
`test-log`란 이름을 가진 로그가 Stackdriver에 전송되었을 것이다. 
<br><br>

## 3. 로그 뷰어를 이용한 로그 검색
앞에서 전송한 로그들이 잘 수집되었는지 확인해보자. `로그 기록` -> `로그`를 클릭한다.
<br>
![stackdriver-search-01](https://user-images.githubusercontent.com/19832483/50736865-0b49cb00-1206-11e9-8a92-f7f5f5aaa7c9.png){: .mid-img}
<br><br>

아래와 같이 로그가 잘 전송되었는지 확인한다. 
<br>
![stackdriver-search-02](https://user-images.githubusercontent.com/19832483/50736709-113eac80-1204-11e9-9022-e5695dd473ed.png){: .u-mid-img}
<br><br>

필터를 이용해서 원하는 로그만 검색할 수 있다. 아래는 로그레벨이 `ERROR`인 로그들만 필터링하여 검색한 것이다. 필터에 대한 내용은 [링크][Stackdriver-Logging-Filter]를 참고 바란다.
<br>
![stackdriver-search-03](https://user-images.githubusercontent.com/19832483/50736711-13a10680-1204-11e9-9d62-94010d6d7f84.png){: .mid-img}
<br><br>

## 4. 수집된 로그를 이용한 메트릭 생성 및 시각화(대시보드 작성)
Stackdriver를 이용하면 수집된 로그를 기반으로 메트릭을 생성하고, `Stackdriver Monitoring`을 이용해 메트릭을 시각화할 수 있다. 직접 실습해보자.
<br><br>

### 로그 기반의 메트릭 생성
`Stackdriver 로그 기록` -> `로그 기반 측정항목`을 클릭한다.
<br>
![stackdriver-metric-01](https://user-images.githubusercontent.com/19832483/50771013-a0af9280-12cc-11e9-8f4d-db9f53eef5f7.png){: .small-img}
<br><br>

로그 기반 측정항목 페이지에서 시스템 측정항목과 사용자 정의 측정항목이 보일 것이다. `시스템 측정항목`은 Stackdriver에서 이미 사전 정의된 로그 기반 측정항목이다. 실무에서는 주로 커스텀한 메트릭을 정의하여 사용할 것이므로, `사용자 정의 측정항목`을 생성하는 예시를 설명하겠다. `Application Started` 내용을 포함한 로그를 선정하여 카운터 메트릭으로 만들고, 로그 레벨에 따라 그룹을 나누어 시각화 해보자.

상단의 `측정항목 만들기` 버튼을 클릭한다.
<br>
![stackdriver-metric-02](https://user-images.githubusercontent.com/19832483/50770959-6f36c700-12cc-11e9-8e59-2d30bc37759f.png){: .mid-img}
<br><br>

로그 뷰어와 비슷한 페이지가 나타나는데, 커스텀한 메트릭을 만들기 위해서는 다음과 같은 절차를 따른다.
<br><br>

#### 1. 로그 메트릭을 추출할 필터 작성
필자는 로그의 내용에 `"Application Started"`가 포함된 로그만 추출하도록 다음과 같이 필터를 입력하였다.
<br>
![stackdriver-metric-03](https://user-images.githubusercontent.com/19832483/50771030-ba50da00-12cc-11e9-9bd6-12dcc193b17d.png){: .mid-img}
<br><br>

#### 2. 측정 항목 편집기의 내용 작성
아래의 내용을 참고하여 작성한후, `측정항목 만들기` 버튼을 클릭한다.
<br>
![stackdriver-metric-04](https://user-images.githubusercontent.com/19832483/50771047-c63c9c00-12cc-11e9-89c1-c3a928b391e1.png){: .mid-img}
- 이름 : 메트릭 이름을 작성한다.
- 설명 : 메트릭에 관한 설명을 작성한다.
- 라벨(선택) : 메트릭을 시각화할 때 어떤 기준에 따라 그룹화할 수 있도록 지원한다.
    - 이름 : 라벨 이름을 작성한다.
    - 라벨 유형 : 문자열, 부울, 정수 중 하나를 선택한다.
    - 필드 이름 : 로그의 내용 중 어떤 프로퍼티 값을 기준으로 그룹화할지 이 값에 따라 결정된다. 여기서는 로그 레벨에 따라 그룹을 나누어 시각화할 것이므로, 아래의 로그 내용에 있는 `labels.levelName`을 필드 이름으로 작성하였다.
        - ![stackdriver-metric-05](https://user-images.githubusercontent.com/19832483/50771078-e10f1080-12cc-11e9-8a8c-43bb9f4fbedb.png){: .small-img}
    - 추출 정규 표현식(선택) : 필드의 값을 정제하여 추출해야 한다면 정규표현식을 사용한다.
- 단위 : 측정 항목의 유형이 카운터면 빈 값으로 둔다.
- 유형 : 카운터 또는 분포를 선택할 수 있다. 카운터를 사용할 것이다.

사용자 정의 측정항목에서 메트릭이 성공적으로 생성되었는지 확인한다.
<br><br>

### 메트릭을 시각화하기(대시보드 작성하기)
사용자 정의 측정항목에서 가장 오른쪽에 있는 버튼을 클릭 후 `측정항목 탐색기에서 보기`를 클릭한다.
<br>
![stackdriver-metric-06](https://user-images.githubusercontent.com/19832483/50774777-af4f7700-12d7-11e9-97a5-bee308d464fa.png){: .small-img}
<br><br>

`Stackdriver Monitoring` 페이지가 보여지는데, `Dashboards` -> `Create Dashboard`를 클릭한다.
<br>
![stackdriver-metric-07](https://user-images.githubusercontent.com/19832483/50774780-b080a400-12d7-11e9-862b-7f06dfe4409a.png){: .mid-img}
<br><br>

상단의 `Add Chart` 버튼을 누르고 다음과 같이 작성한다.
<br>
![stackdriver-metric-08](https://user-images.githubusercontent.com/19832483/50774782-b1b1d100-12d7-11e9-9015-07bf3553eae6.png){: .u-mid-img}
- Resource Type : `Global`로 지정하여 전송했으므로 해당 값을 입력한다.
- Metric : 메트릭의 이름을 이전에 `application-started-metric`으로 지정하였으므로, 해당 값을 입력한다.
- Group By : 로그 레벨에 따른 그룹핑을 위해 `logLevel`이라는 라벨을 만들었으므로, 해당 값을 입력한다.
- Aligner : 1분 `동안`의 count 값을 합산할 것이므로 `delta`를 입력한다.
- Reducer : 1분 동안의 count 값을 합산할 것이므로 `sum`을 입력한다.
- Alignment Period : `1분` 동안의 count 값을 합산할 것이므로 `1`을 입력한다.

옵션에 대한 자세한 설명은 [링크][Stackdriver-Monitoring-Chart-Option]를 참고하라. 작성했으면 `SAVE` 버튼을 누른다.

드디어 아래와 같이 로그 레벨에 따라 분류된 그래프를 작성하였다.
![stackdriver-metric-09](https://user-images.githubusercontent.com/19832483/50774784-b24a6780-12d7-11e9-805d-f03798cb6da3.png){: .mid-img}
<br><br>

## 5. 메트릭 기반 모니터링 알람 설정
메트릭 기반 모니터링 알람을 설정하는 것은 메트릭을 시각화하는 것과 비슷하다. Stackdriver Monitoring 페이지에서 `Alert` -> `Create a Policy`를 클릭하면 알람을 작성할 수 있다.
<br>
![stackdriver-alert-01](https://user-images.githubusercontent.com/19832483/50777579-571c7300-12df-11e9-9c26-4a2d70189d07.png){: .u-mid-img}
<br><br>

알람 조건을 작성하는 방법은 메트릭을 시각화하는 방법과 비슷하다. 나머지 알람 유형 설정이나 그외 설정에 대한 설명은 포스트 주제에서 벗어나므로 [링크][Stackdriver-Monitoring-Alarm]를 참고하자.
<br><br>

 실무에서 유용할 것이라고 판단되는 기능 위주로 포스트를 작성하였다. 이외에도 로그를 영구적으로 저장하기 위해 다른 저장소로 내보내기 기능을 포함한 설명하지 않은 기능이 있는데, 추가 기능들이 궁금하다면 [링크][Stackdriver-Logging-extra]를 참고하자. 다음에 `Stackdriver`에 관한 글을 포스팅하게 된다면 필자가 유용한 기능이라고 생각이 되는 `Stackdriver Monitoring` 또는 `Stackdriver Trace`에 관한 내용을 포스팅할 예정이다.


[Stackdriver-Logging-Filter]:https://cloud.google.com/logging/docs/view/overview?hl=ko
[Stackdriver-Monitoring-Chart-Option]:https://cloud.google.com/monitoring/charts/metrics-selector?&_ga=2.193715109.-907900056.1546769825#aggregation-options
[Stackdriver-Monitoring-Alarm]:https://cloud.google.com/monitoring/alerts/
[Stackdriver-Logging-extra]:https://cloud.google.com/logging/docs/how-to?hl=ko
