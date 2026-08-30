# Prometheus Monitoring

Prometheus 기반의 서버 및 애플리케이션 모니터링 시스템입니다.

## 1. 개요

Prometheus를 이용하여 서버 및 애플리케이션의 Metrics를 수집하고,
Grafana를 통해 수집된 Metrics를 시각화합니다.

주요 구성요소는 다음과 같습니다.

* Prometheus: Metrics 수집 및 저장
* Node Exporter: Linux 서버 시스템 Metrics 제공
* Grafana: Metrics 조회 및 Dashboard 구성

## 2. Architecture

```text
                         ┌──────────────┐
                         │    Grafana   │
                         │    :3000     │
                         └──────┬───────┘
                                │
                                │ PromQL
                                ▼
                         ┌──────────────┐
                         │  Prometheus  │
                         │    :9090     │
                         └──────┬───────┘
                                │
                             scrape
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
      ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
      │ Node Exporter│    │ Node Exporter│    │ Node Exporter│
      │    :9100     │    │    :9100     │    │    :9100     │
      └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
             │                  │                  │
          Server A           Server B           Server C
```

Prometheus는 각 모니터링 대상의 `/metrics` endpoint에 주기적으로
접근하여 Metrics를 수집합니다.

## 3. Components

| Component     | Port | Description             |
| ------------- | ---: | ----------------------- |
| Prometheus    | 9090 | Metrics 수집 및 저장         |
| Grafana       | 3000 | Metrics 시각화             |
| Node Exporter | 9100 | Linux 서버 시스템 Metrics 제공 |

## 4. Prometheus

### 4.1 Configuration

Prometheus 설정 파일:

```text
prometheus.yml
```

예시:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets:
          - "10.10.30.10:9100"
          - "10.10.30.11:9100"
```

### 4.2 실행

```bash
./prometheus \
  --config.file=prometheus.yml
```

Prometheus Web UI:

```text
http://<PROMETHEUS_HOST>:9090
```

## 5. Node Exporter

Node Exporter는 Linux 서버의 시스템 Metrics를 Prometheus가 수집할 수 있도록
`/metrics` endpoint를 제공합니다.

### 5.1 실행

```bash
./node_exporter
```

기본 포트:

```text
9100
```

### 5.2 Metrics 확인

```bash
curl http://localhost:9100/metrics
```

정상적으로 실행 중이라면 CPU, Memory, Disk, Network 등의 Metrics가 출력됩니다.

## 6. Monitoring Target 추가

새로운 서버를 모니터링하려면 `prometheus.yml`의 `targets`에
Node Exporter 주소를 추가합니다.

```yaml
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets:
          - "10.10.30.10:9100"
          - "10.10.30.11:9100"
          - "10.10.30.12:9100"
```

설정 변경 후 Prometheus를 Reload합니다.

```bash
curl -X POST http://localhost:9090/-/reload
```

또는 Prometheus 프로세스를 재시작합니다.

## 7. Target 상태 확인

Prometheus Web UI에서 Target 상태를 확인합니다.

```text
http://<PROMETHEUS_HOST>:9090/targets
```

정상 상태:

```text
State: UP
```

비정상 상태:

```text
State: DOWN
```

Target이 `UP`이면 Prometheus가 해당 Node Exporter의 Metrics를
정상적으로 수집하고 있다는 의미입니다.

## 8. Grafana

Grafana는 Prometheus에 저장된 Metrics를 Dashboard 형태로 시각화합니다.

Grafana 접속:

```text
http://<GRAFANA_HOST>:3000
```

### 8.1 Prometheus Data Source 등록

Grafana에서 다음과 같이 Prometheus를 Data Source로 등록합니다.

```text
Type:
Prometheus

URL:
http://<PROMETHEUS_HOST>:9090
```

### 8.2 Dashboard

PromQL을 이용하여 다음과 같은 Metrics를 조회할 수 있습니다.

* CPU Usage
* Memory Usage
* Disk Usage
* Network Traffic
* Load Average
* Filesystem Usage
* TCP Connections

## 9. PromQL 예시

CPU 사용률:

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Memory 사용률:

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

Filesystem 사용률:

```promql
100 * (
  1 - node_filesystem_avail_bytes{fstype!=""}
      / node_filesystem_size_bytes{fstype!=""}
)
```

Network 수신량:

```promql
rate(node_network_receive_bytes_total[5m])
```

Network 송신량:

```promql
rate(node_network_transmit_bytes_total[5m])
```

## 10. Troubleshooting

### 10.1 Node Exporter 상태 확인

```bash
systemctl status node_exporter
```

또는 프로세스를 확인합니다.

```bash
ps -ef | grep node_exporter
```

### 10.2 Node Exporter Port 확인

```bash
ss -lntp | grep 9100
```

정상적으로 실행 중이라면 `9100` 포트가 LISTEN 상태여야 합니다.

### 10.3 Metrics 접근 확인

Prometheus 서버에서 대상 서버의 Metrics endpoint에 접근합니다.

```bash
curl http://<TARGET_IP>:9100/metrics
```

정상적으로 Metrics가 출력되는지 확인합니다.

### 10.4 방화벽 확인

Node Exporter의 `9100/TCP` 접근이 허용되어 있는지 확인합니다.

```bash
firewall-cmd --list-all
```

필요한 경우:

```bash
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

## 11. Network Requirements

Prometheus가 각 Monitoring Target의 Node Exporter에 접근할 수 있어야 합니다.

```text
Prometheus
    |
    | TCP/9100
    ▼
Node Exporter
```

Grafana는 Prometheus에 접근할 수 있어야 합니다.

```text
Grafana
    |
    | TCP/9090
    ▼
Prometheus
```

필요한 네트워크 접근 관계:

| Source     | Destination   |     Port | Purpose           |
| ---------- | ------------- | -------: | ----------------- |
| Prometheus | Node Exporter | TCP/9100 | Metrics 수집        |
| Grafana    | Prometheus    | TCP/9090 | Metrics 조회        |
| User       | Grafana       | TCP/3000 | Dashboard 접속      |
| User       | Prometheus    | TCP/9090 | Prometheus Web UI |

## 12. 운영 시 확인사항

### Prometheus

* Prometheus 프로세스 정상 여부
* Disk 사용량
* Target UP/DOWN 상태
* Scrape 실패 여부
* Prometheus 로그

### Node Exporter

* 프로세스 정상 여부
* `9100/TCP` LISTEN 여부
* `/metrics` 접근 가능 여부

### Grafana

* Prometheus Data Source 정상 여부
* Dashboard 정상 조회 여부
* PromQL 오류 여부

## 13. 구성 파일

```text
prometheus/
├── prometheus.yml
└── README.md
```

Prometheus 설정:

```text
prometheus.yml
```

Grafana Dashboard 및 Data Source 설정은 Grafana에서 관리합니다.
