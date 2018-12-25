---
layout: post
title:  "모니터링 시스템 - Prometheus #1 - 아키텍쳐와 개념"
date:   2018-12-25 15:14:54
categories: prometheus
comments: true
---

### What is Prometheus?
#### 메트릭 정보를 수집하여 시스템을 모니터링하고 Alerting을 지원하는 오픈소스

### Architecture
![prometheus-architecture-01](https://user-images.githubusercontent.com/19832483/50424631-8cba6880-08aa-11e9-87b9-d7572088e7d9.png)

### Concept
- Data Model
	- Metric name
		- 측정되는 시스템의 기능
		- ex) http_requests_total
	- Label
		- Metric name에 대응되는 다수의 측정 항목
		- key와 value로 이루어짐
		- ex) method="POST", handler="/messages"
		- Influx의 tag에 해당
- Job And Instances
	- Instance
		- 스크랩할수 있는 엔드포인트 (수집을 당하는 곳)
		- 보통 단일 프로세스임
	- Job
		- 확장성이나 안정성을 위해 복제된 동일한 인스턴스의 모음
	- Example
		- job: api-server
			- instance 1: 1.2.3.4:5670
			- instance 2: 1.2.3.4:5671
			- instance 3: 5.6.7.8:5670
- Recoding rules
	- Influx의 Continous Query와 같은 역할을 한다.
	- 이미 존재하는 Metric으로 새로운 Time Series Recode를 만들어낸다.
	- ex) latency를 5초 동안 평균을 낸다.
- Storage
	- 기본적으로 로컬 on-disk 시계열 DB를 포함하고 있음
	- 그러나 remote storage도 허용
	- Local Stroage
		- 샘플들은 두시간 단위의 블록들로 그룹되어짐
		- 들어오고 있는 데이터들은 메모리상에서 유지
		- 그러나 WAL(write ahead log) 방식으로 장애 복구시 메모리에서 유지됐던것을 복구 가능
		-  tsdb 포맷을 사용하며 블록(디렉토리) 안에는 다음의 내용들로 구성됨
			- meta.json (메타 데이터)
			- wal (write ahead log)
			- chunks (청크 데이터)
			- tombstones (삭제 표시) - 데이터 삭제시 데이터를 바로 삭제하지 않고 표시해둠
		-  다음의 중요한 설정 가능
			- --storage.tsdb.path
				- 데이터를 저장할 디렉토리를 지정
				- default : ./data
			- --storage.tsdb.retention
				- 언제까지 디스크에 남겨놓을건지?
				- 디스크 크기를 구하는 공식
					- needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample 
- FEDERATION
	- 확장 가능한 Prometheus 모니터링 설정을 달성하거나
	- 특정 서비스의 Prometheus에서 관련 메트릭을 다른 것으로 가져 오는데 사용
	- 종류
		- Hierarchical Federation
		- Cross-service Ferderation
- Exporters
	- 모니터링 대상의 Metric 정보를 수집
	- Exporter Http EndPoint를 통해 Prometheus에서 Metric을 수집 (Pull 방식)
	- 서드 파티 시스템에서 Prometheus metrics를 export하기 위해 사용
- AlertManager
	- Prometheus Server와 같은 클라이언트 어플리케이션에 의해 보내진 Alert을 처리
	- 다음 종류의 Alert이 있음
		- Grouping
			- 비슷한 성격의 경고를 단일 알람으로 분류함
			- ex) 수백개의 인스턴스에서 발생하는 Alert을 하나의 그룹으로 묶어 단일 Alert으로 전송
		- Inhibition
			- 특정 다른 Alert이 이미 발생하면, 특정 Alert을 억제함
			- 클러스터안의 전체 인스턴스에 접근이 불가능할 경우 이 클러스터와 관련된 다른 모든 경고를 음소거하도록함
			- 상관없는 다른 Alert들을 억제함
		- Silences
			- 주어진 시간 동안 단순히 Alert을 음소거함  