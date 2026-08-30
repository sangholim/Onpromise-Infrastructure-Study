# Prometheus Monitoring 설치 가이드

## 1. 설치 순서

Prometheus 기반 모니터링 시스템은 다음 순서로 구축합니다.

```text
┌──────────────────────┐
│ 1. Prometheus 설치   │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 2. Node Exporter 설치│
└──────────┬───────────┘
           ↓
┌────────────────────────────┐
│ 3. Prometheus Target 등록  │
└────────────┬───────────────┘
             ↓
┌────────────────────────────┐
│ 4. Metrics 수집 정상 여부 확인 │
└────────────┬───────────────┘
             ↓
┌──────────────────────┐
│ 5. Grafana 설치      │
└──────────┬───────────┘
           ↓
┌────────────────────────────┐
│ 6. Grafana → Prometheus 연결 │
└────────────┬───────────────┘
             ↓
┌──────────────────────┐
│ 7. Dashboard 구성    │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 8. Alerting 구성     │
└──────────────────────┘
```

## 2. Prometheus 설치

Prometheus 서버에 Prometheus를 설치합니다.

### 2.1 사용자 생성

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

### 2.2 디렉터리 생성

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

### 2.3 Prometheus 설치

Prometheus 바이너리를 다운로드하여 설치합니다.

```bash
tar xvf prometheus-<VERSION>.linux-amd64.tar.gz

cd prometheus-<VERSION>.linux-amd64
```

바이너리 복사:

```bash
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
```

설정 파일 복사:

```bash
sudo cp prometheus.yml /etc/prometheus/
```

권한 설정:

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

## 3. Prometheus 설정

설정 파일:

```text
/etc/prometheus/prometheus.yml
```

기본 설정:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"
```

## 4. Prometheus 서비스 등록

systemd 서비스 파일:

```text
/etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

서비스 등록 및 실행:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

상태 확인:

```bash
sudo systemctl status prometheus
```

포트 확인:

```bash
ss -lntp | grep 9090
```

Prometheus Web UI:

```text
http://<PROMETHEUS_IP>:9090
```

## 5. Node Exporter 설치

모니터링 대상 서버마다 Node Exporter를 설치합니다.

```text
Prometheus Server
      │
      │ TCP/9100
      ▼
Monitoring Server
Node Exporter
```

Node Exporter는 서버의 CPU, Memory, Disk, Network 등의 시스템 Metrics를 제공합니다.

### 5.1 사용자 생성

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### 5.2 설치

```bash
tar xvf node_exporter-<VERSION>.linux-amd64.tar.gz

cd node_exporter-<VERSION>.linux-amd64

sudo cp node_exporter /usr/local/bin/
```

권한 설정:

```bash
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

## 6. Node Exporter 서비스 등록

서비스 파일:

```text
/etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple

ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

서비스 실행:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

상태 확인:

```bash
sudo systemctl status node_exporter
```

포트 확인:

```bash
ss -lntp | grep 9100
```

Metrics 확인:

```bash
curl http://localhost:9100/metrics
```

## 7. Prometheus에 Node Exporter 등록

Prometheus 서버의 설정 파일을 수정합니다.

```text
/etc/prometheus/prometheus.yml
```

예:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"

  - job_name: "node"
    static_configs:
      - targets:
          - "10.10.30.10:9100"
          - "10.10.30.11:9100"
          - "10.10.30.12:9100"
```

설정 검증:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

정상이면 Prometheus를 Reload합니다.

```bash
sudo systemctl reload prometheus
```

또는:

```bash
sudo systemctl restart prometheus
```

## 8. Metrics 수집 확인

Prometheus Web UI에서 다음 메뉴로 이동합니다.

```text
Status
  └── Targets
```

또는:

```text
http://<PROMETHEUS_IP>:9090/targets
```

정상:

```text
State: UP
```

비정상:

```text
State: DOWN
```

PromQL로도 확인할 수 있습니다.

```promql
up
```

결과가 다음과 같으면 정상적으로 수집되고 있는 것입니다.

```text
up{instance="10.10.30.10:9100"} 1
up{instance="10.10.30.11:9100"} 1
```

## 9. Grafana 설치

Prometheus Metrics를 Dashboard로 시각화하기 위해 Grafana를 설치합니다.

```text
Prometheus
     │
     │ PromQL
     ▼
Grafana
```

Grafana 설치 후 서비스를 실행합니다.

```bash
sudo systemctl enable --now grafana-server
```

상태 확인:

```bash
sudo systemctl status grafana-server
```

기본 포트:

```text
3000
```

접속:

```text
http://<GRAFANA_IP>:3000
```

## 10. Grafana에 Prometheus 연결

Grafana에서:

```text
Connections
  → Data sources
  → Add data source
  → Prometheus
```

Prometheus URL을 입력합니다.

```text
http://<PROMETHEUS_IP>:9090
```

`Save & test`를 실행하여 연결 상태를 확인합니다.

## 11. Dashboard 구성

Prometheus Data Source가 정상적으로 연결되면 Grafana Dashboard를 구성합니다.

주요 모니터링 항목:

```text
CPU
Memory
Disk
Filesystem
Network
Load Average
TCP Connections
```

예를 들어 CPU 사용률:

```promql
100 - (
  avg by(instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

Memory 사용률:

```promql
100 * (
  1 - node_memory_MemAvailable_bytes
      / node_memory_MemTotal_bytes
)
```

## 12. 설치 완료 후 검증

전체 구성은 다음 순서로 검증합니다.

### 12.1 Node Exporter

```bash
curl http://<NODE_EXPORTER_IP>:9100/metrics
```

Metrics가 출력되는지 확인합니다.

### 12.2 Prometheus

```text
http://<PROMETHEUS_IP>:9090/targets
```

Target이 `UP`인지 확인합니다.

### 12.3 PromQL

Prometheus에서:

```promql
up
```

결과가 `1`인지 확인합니다.

### 12.4 Grafana

```text
http://<GRAFANA_IP>:3000
```

Prometheus Data Source가 정상인지 확인합니다.

### 12.5 Dashboard

CPU, Memory, Disk 등의 Metrics가 정상적으로 표시되는지 확인합니다.

## 13. 최종 구성

정상적으로 구축되면 다음과 같은 구조가 됩니다.

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
                      HTTP /metrics
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Server A      Server B      Server C
          :9100         :9100         :9100
             │             │             │
        Node Exporter  Node Exporter  Node Exporter
```

최종적으로 **Node Exporter → Prometheus → Grafana** 순서로 Metrics가 전달됩니다.
