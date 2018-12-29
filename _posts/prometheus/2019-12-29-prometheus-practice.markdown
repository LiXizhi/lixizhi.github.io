---
layout: post
title:  "Prometheus #2 - Prometheus & Grafana를 이용한 Spring Boot 메트릭 모니터링"
date:   2018-12-25 15:14:54
categories: prometheus
comments: true
---
이번 글에서는 Prometheus를 이용해 Spring Boot 모니터링 메트릭을 수집하고 Grafana로 시각화하는 실습을 포스팅할 것이다.

우선 Docker를 이용해 Prometheus 이미지를 가져오고, 이를 실행시켜보자.
```bash
	# docker volume을 생성한다.
	$ docker volume create --driver local \
      --opt type=none \
      --opt device='마운트 경로' \
      --opt o=bind \
      prometheus-vol

	#
	$ 
```
