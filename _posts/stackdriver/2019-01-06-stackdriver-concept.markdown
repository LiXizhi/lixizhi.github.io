---
layout: post
title:  "Stackdriver #1 - Stackdriver란 무엇인가?"
date:   2019-01-06 00:14:54
categories: stackdriver
comments: true
---
* content
{:toc}

### Stackdriver란 무엇인가?
`Stackdriver`는 클라우드나 On-Prem 환경의 애플리케이션에서 발생하는 분산 로그, 메트릭, 이벤트를 GCP의 Stackdriver에서 한 곳에 수집하여 모니터링에 대한 다양한 서비스를 제공한다.
<br><br>

### Stackdriver의 장점
보통 애플리케이션의 로그를 대용량으로 한 곳에서 수집하고 가공할 때 `Log Producer`->`Message Q`->`Log Consumer`-> `Log Storage`->`Reporting` 과정을 거친다. 위 단계와 관련된 인프라 및 애플리케이션을 구축하는건 많은 비용이 들고 운영시 관리 포인트가 늘어나게 된다. 하지만 Stackdriver를 사용하면 `Log Producer`는 Stackdriver에서 제공해주는 라이브러리 및 에이전트를 사용하고, 그 뒷 단계는 Stackdriver에 이미 구축이 되어있으니 이에 대한 구축 및 관리 비용이 줄게된다. 

그리고 logging 기능 외 모니터링, 디버깅, 프로파일링, 알람 등 다른 풍부한 기능들도 제공한다.

또한 Cloud(GCP, AWS) 환경이 아닌 On-Prem 환경에서도 Stackdriver를 사용할 수 있다.
<br><br>

### Stackdriver가 제공하는 기능
[링크][Stackdriver-Feature]에 제공하는 기능이 명시되어 있다. 제공하는 기능들에 대한 실습은 다음 포스트에서 진행할 예정이다.

[Stackdriver-Feature]:https://cloud.google.com/stackdriver/?hl=ko