---
layout: post
title:  "Prometheus #2 - Prometheus & Grafana를 이용한 Spring Boot 메트릭 모니터링"
date:   2018-12-29 15:14:54
categories: prometheus
comments: true
---
이번 글에서는 Prometheus를 이용해 Spring Boot 모니터링 메트릭을 수집하고 Grafana로 시각화하는 실습을 포스팅할 것이다.

우선 Docker를 이용해 Prometheus 이미지를 가져오고, 이를 실행시켜보자.
```console
	# docker volume을 생성한다. 이를 생성하는 이유는 테스트용도로 컨테이너 안의 Prometheus Config 파일을 쉽게 수정하기 위함이다.
	# 마운트 경로에는 마운트될 로컬 Path를 입력한다.
	$ docker volume create --driver local \
      --opt type=none \
      --opt device='마운트 경로' \
      --opt o=bind \
      prometheus-volume

	# 위에서 생성한 volume을 이용해서 Prometheus를 실행한다.
	$ docker run -p 9090:9090 -v prometheus-volume:/etc/prometheus -d prom/prometheus
```
