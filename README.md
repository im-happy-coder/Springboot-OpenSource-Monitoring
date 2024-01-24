
# 오픈소스 모니터링 Tool를 이용하여 Application 모니터링하기

## Overview

Java Application의 성능 모니터링을 위해 Prometheus를 이용하여 SpringBoot Application의 JVM, Thread, DataSource 등의 데이터를 수집하고 Loki와 Promtail를 활용하여 Application Log를 수집하고
Grafana를 이용하여 Prometheus, Loki Dashboard를 생성해 수집한 데이터를 시각화 할 수 있다.
뿐만 아니라 Application에서 중요도가 높은 Error Log, 또는 JVM의 특이사항(GC FULL)등의 발생 시 Slack Alert 설정하여
Application의 이슈가 발생하였을 때 Alert 기능을 이용하여 이슈 상황에 대비할 수 있도록 모나토랑 오픈소스를 이용하여 구현함.

## Diagram

![prometheus_loki_promtail.jpg](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/prometheus_loki_promtail.jpg?raw=true)

## Skils

Spring v4.3.7

Springboot v1.4.5

Loki v2.8.0

Promtail v2.8.0

Pinpoint v2.5.1

Prometheus latest

Grafana v9.4.7

Docker v20.10.22

Docker-compose v2.15.1

## Install

### Pinpoint 설치

git clone https://github.com/pinpoint-apm/pinpoint-docker.git

cd pinpoint-docker

git checkout 2.5.1

docker-compose -f docker-compose.yml -f docker-compose-metric.yml build

docker-compose -f docker-compose.yml -f docker-compose-metric.yml up -d


- 아래 주소로 이동

http://192.168.202.11:8080/main


### SpringBoot Pinpoint VM arguments Setting

``` text
-javaagent:C:\pinpoint\pinpoint-agent-2.5.1\pinpoint-bootstrap-2.5.1.jar
-Dpinpoint.zookeeper.address=192.168.91.11
-Dpinpoint.applicationName=ttaeinee-nldf-mso
-Dpinpoint.agentId=ttaeinee-agent-2
-Dprofiler.transport.grpc.collector.ip=192.168.91.11
-Dprofiler.collector.span.port=9996
-Dprofiler.collector.stat.port=9995
-Dprofiler.collector.tcp.port=9994
-Dhbase.client.host=192.168.91.11
-Dhbase.client.port=2181
```

- SpringBoot Application 실행 후 Pinpoint 웹 콘솔 접속
- http://localhost:8080/main
- Pinpoint Application 모니터링

![pinpoint-1](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/pinpoint-1.jpg?raw=true)

- 트랜잭션 정보
  
![pinpoint-2](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/pinpoint-2.jpg?raw=true)

- JVM 모니터링
  
![pinpoint-3](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/pinpoint-3.jpg?raw=true)

- Server 자원 모니터링 
  
![pinpoint-4](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/pinpoint-4.jpg?raw=true)

### Prometheus 설치

docker run -d -p 9090:9090 --name prometheus_v1 -v /app/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus:latest

- prometheus yml 설정

``` yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ttaeinee-nldf-v1'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['192.168.202.222:9595']
```

- prometheus config 설정 확인
  
![prometheus-config-1](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/prometheus-config-1.jpg?raw=true)

### Prometheus Spring Application 설정

- Prometheus Maven Dependency 추가
 
``` xml
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-spring-legacy</artifactId>
        <version>1.0.6</version>
    </dependency>
    
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <version>1.0.6</version>
    </dependency>
```

- Spring Application 실행 후 Prometheus 연동 확인
  - http://192.168.202.11:9090/targets

![prometheus-target-1.jpg](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/prometheus-target-1.jpg?raw=true)

- Application에서 Prometheus value 확인
  - http://192.168.202.222:9595/prometheus

![spring-prometheus-1.jpg](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/spring-prometheus-1.jpg?raw=true)

- springboot 1.4.5 버전에서 prometheus url에 logfile endpoint 추가
  - application.yml 파일 수정

``` yml
endpoints:
  logfile:
    external-file: D:/ttaeinee-nldf-v1/logs/${spring.application.name}.log
``` 

- 아래 url로 접속하면 Spring Application 로그 확인
  - http://localhost:9595/logfile

## Grafana, Loki, Promtail 설치

- Loki 설치
  - wget https://github.com/grafana/loki/releases/download/v2.8.0/loki-linux-arm64.zip

  - wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/cmd/loki/loki-local-config.yaml

- Promtail 설치
  - wget https://github.com/grafana/loki/releases/download/v2.8.0/promtail-linux-arm64.zip

  - wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/clients/cmd/promtail/promtail-local-config.yaml

- Grafana 설치
  - docker run -d -p 3000:3000 --name=grafana grafana/grafana:9.4.7

- Loki yaml 수정
  - loki-local-config.yaml

``` yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```

- Promtail yaml 수정
  - promtail-local-config.yaml
``` yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.202.11:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - 192.168.202.11
    labels:
      job: varlogs
      __path__: /var/log/*log

- job_name: ttainee-nldf-mso-log
  static_configs:
  - targets:
      - 192.168.202.11
    labels:
      job: ttainee-nldf-v1-log
      __path__: D:/ttaeinee-nldf-mso/logs/ttaeinee-nldf-v1.log
```
- Loki, Promtail 실행 Shell Script

``` bash
#!/bin/bash

./loki-linux-amd64 -config.file=loki-local-config.yaml > /dev/null 2>&1 &

sleep 1

./promtail-linux-amd64 -config.file=promtail-local-config.yaml > /dev/null 2>&1 &
```

### SpringBoot LogBack 설정

- application. yml
``` yml
logging:
  target-dir: D:/ttaeinee-nldf-v1/logs/
```

- logback.xml

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<springProperty name="APP_NAME" source="spring.application.name" />
	<springProperty name="TARGET_DIR" source="logging.target-dir" />

	<include resource="org/springframework/boot/logging/logback/defaults.xml" />

	<property name="CONSOLE_LOG_PATTERN"
		value="%clr(%d{HH:mm:ss.SSS}){faint} %clr(%5p) %clr(${PID:- }){magenta} %clr(-){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n%wex" />
	<property name="FILE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} %5p ${PID:- } - [%t] %-40.40logger{39} : %m%n%wex" />

	<property name="LOG_FILE" value="${TARGET_DIR}/${APP_NAME}.log}" />
	<include resource="org/springframework/boot/logging/logback/console-appender.xml" />
	<include resource="org/springframework/boot/logging/logback/file-appender.xml" />

	<logger name="jdbc" level="OFF" />
	<logger name="jdbc.sqltiming" level="ERROR" additivity="false">
		<appender-ref ref="CONSOLE" />
		<appender-ref ref="FILE" />
	</logger>
	<logger name="jdbc.resultsettable" level="ERROR" additivity="false">
		<appender-ref ref="FILE" />
	</logger>

	<logger name="com.lottedfs" level="DEBUG" />
	<logger name="org.thymeleaf" level="ERROR" />
	<logger name="org.springframework" level="INFO" />
	<logger name="org.springframework.cache.interceptor.CacheInterceptor" level="TRACE" />

	<root level="DEBUG">
		<appender-ref ref="CONSOLE" />
		<appender-ref ref="FILE" />
	</root>
</configuration>
```

### Grafana와 Prometheus, Loki 연동하기

- Grafana Datasource 추가

![grafana-datasource-1](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-datasource-1.JPG?raw=true)

- Grafana Prometheus Dashboard 추가
  - Prometheus로 수집한 Spring Application data 추가
  - Loki를 이용한 Application Log Dashboard 추가

![grafana-dashboard-2](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-dashboard-2.jpg?raw=true)

![grafana-dashboard-3](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-dashboard-3.jpg?raw=true)

![grafana-dashboard-1](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-dashboard-1.jpg?raw=true)

### Grafana Loki Slack Alert 설정

- Alert Rule 추가
  
![grafana-alert-2](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-alert-2.jpg?raw=true)

- Slack 확인
  
![grafana-alert-1](https://github.com/im-happy-coder/Springboot-OpenSource-Monitoring/blob/main/img/grafana-alert-1.jpg?raw=true
)
## Referer

> pinpoint docker</br>
> https://github.com/pinpoint-apm/pinpoint-docker</br>
> loki, promtail download</br>
> https://github.com/grafana/loki/releases</br>
> grafana log alert 설정</br>
> https://creampuffy.tistory.com/213</br>
> slack grafana 연동</br>
> https://afsdzvcx123.tistory.com/entry/Grafana-Alert-Slack-%EC%97%B0%EB%8F%99%ED%95%98%EA%B8%B0</br>
> grafana slack 연동 및 alert 설정</br>
> https://jaeyung1001.tistory.com/entry/Grafana-Slack-Alert-%EB%A9%94%EC%84%B8%EC%A7%80-%EC%BB%A4%EC%8A%A4%ED%85%80%EB%A7%88%EC%9D%B4%EC%A7%95</br>
> https://jaeyung1001.tistory.com/entry/Grafana-Slack-Alert-%EB%A7%8C%EB%93%A4%EA%B8%B0</br>
> 그라파나와 로키로 애플리케이션 로그 조회하기</br>
> https://inma.tistory.com/164</br>
