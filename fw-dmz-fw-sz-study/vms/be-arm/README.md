# EXT-DMZ
## 네트워크
- HOST-ONLY (10.10.30.11) \
  gateway 10.10.30.2 
``` {shell} \
  # ipv4 할당 및 gateway 설정
  nmcli connection modify enp0s8 ipv4.addresses 10.10.30.11/24
  nmcli connection modify enp0s8 ipv4.gateway 10.10.30.2
  nmcli connection modify enp0s8 ipv4.method manual
  nmcli connection up enp0s8
  # ip route
  default via 10.10.30.2

```

## 호스트 설정

```shell
sudo hostnamectl set-hostname be-arm
```
-------

## 내부망에서는 방화벽 비활성화

```bash
sudo systemctl disable --now firewalld
```


## 목표
- vm 들을 모니터링하는 툴 설치

### 역할

| 구성요소 | 역할 | 기본 포트 |
|---|---|---:|
| Node Exporter | VM의 시스템 Metric 제공 | 9100 |
| Prometheus | Metric 수집 및 저장 | 9091 |
| Grafana | Metric 시각화 | 8080 |
| Alertmanager | Alert 수신 및 알림 전달 | 9093 |

---

# 2. Prometheus 설치

Prometheus 서버에 설치합니다.

## 2.1 사용자 생성

```bash
sudo useradd --no-create-home --shell /sbin/nologin prometheus
```

## 2.2 디렉터리 생성

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

## 2.3 Prometheus 다운로드

서버가 ARM64인 경우 ARM64 바이너리를 사용합니다.

```bash
cd /tmp

wget https://github.com/prometheus/prometheus/releases/download/v3.13.1/prometheus-3.13.1.linux-arm64.tar.gz

tar -xzf prometheus-3.13.1.linux-arm64.tar.gz

cd prometheus-3.13.1.linux-arm64
```

설치 전에 서버 아키텍처를 확인합니다.

```bash
uname -m
```

ARM64 서버라면:

```text
aarch64
```

## 2.4 바이너리 설치

```bash
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

sudo chmod +x /usr/local/bin/prometheus
sudo chmod +x /usr/local/bin/promtool
```

## 2.5 설정 파일 설치

```bash
sudo cp prometheus.yml /etc/prometheus/
```

권한 설정:

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

---

# 3. Prometheus 설정

설정 파일:

```text
/etc/prometheus/prometheus.yml
```

초기 설정:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9091"
```

Prometheus 자체의 Metric을 수집하는 설정입니다.

---

# 4. Prometheus systemd 등록

서비스 파일:

```bash
sudo vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9091

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
ss -lntp | grep 9091
```

Prometheus Web UI:

```text
http://<PROMETHEUS_IP>:9091
```

---

# 5. Node Exporter 설치

Node Exporter는 모니터링 대상 VM마다 설치합니다.

예:

```text
Prometheus
10.10.20.100:9091
     │
     │ TCP 9100
     ▼
Backend VM
10.10.30.2:9100
```

## 5.1 Node Exporter 다운로드

대상 VM에서 실행합니다.

```bash
cd /tmp

wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-arm64.tar.gz

tar -xzf node_exporter-1.12.1.linux-arm64.tar.gz
```

바이너리 설치:

```bash
sudo cp node_exporter-1.12.1.linux-arm64/node_exporter /usr/local/bin/

sudo chmod +x /usr/local/bin/node_exporter
```

## 5.2 사용자 생성

```bash
sudo useradd --system \
  --no-create-home \
  --shell /sbin/nologin \
  node_exporter
```

---

# 6. Node Exporter systemd 등록

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple

ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

실행:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

상태 확인:

```bash
sudo systemctl status node_exporter
```

Metric 확인:

```bash
curl http://localhost:9100/metrics
```

정상적으로 Metric이 출력되면 Node Exporter 설치가 완료된 것입니다.

---

# 7. 방화벽 설정

다른 망에 있는 VM을 모니터링하는 경우 Prometheus 서버에서 Node Exporter까지 TCP 9100 통신이 가능해야 합니다.

예:

```text
Prometheus
10.10.20.100
     │
     │ TCP 9100
     ▼
Backend VM
10.10.30.2
```

Backend VM의 방화벽에서 Prometheus 서버의 IP만 허용하는 것을 권장합니다.

예:

```bash
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="10.10.20.100" port port="9100" protocol="tcp" accept'

sudo firewall-cmd --reload
```

확인:

```bash
sudo firewall-cmd --list-all
```

Prometheus 서버에서:

```bash
nc -vz 10.10.30.2 9100
```

정상:

```text
Connected to 10.10.30.2:9100
```

그리고:

```bash
curl http://10.10.30.2:9100/metrics
```

---

# 8. Prometheus에 Node Exporter 등록

Prometheus 서버에서:

```bash
sudo vi /etc/prometheus/prometheus.yml
```

예:

```yaml
global:
  scrape_interval: 15s

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9091"

  - job_name: "node"
    static_configs:
      - targets:
          - "10.10.30.2:9100"
        labels:
          server: "be-arm"
          env: "prod"
```

여러 VM이면:

```yaml
  - job_name: "node"
    static_configs:
      - targets:
          - "10.10.30.2:9100"
        labels:
          server: "be-arm"
          env: "prod"

      - targets:
          - "10.10.30.3:9100"
        labels:
          server: "dmz-lb"
          env: "prod"
```

설정 검사:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

정상:

```text
SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

재시작:

```bash
sudo systemctl restart prometheus
```

---

# 9. Node Exporter 정상 수집 확인

Prometheus Web UI:

```text
http://<PROMETHEUS_IP>:9091
```

상단 메뉴:

```text
Status
  → Targets
```

여기서:

```text
node
  10.10.30.2:9100
  State: UP
```

이면 정상입니다.

또는 PromQL에서:

```promql
up
```

결과가:

```text
prometheus  1
be-arm      1
```

처럼 나오면 정상입니다.

> Node Exporter가 정상적으로 실행 중이어도 Prometheus에 Target으로 등록하지 않으면 Prometheus가 수집하지 않습니다.

---

# 10. Grafana 설치

Grafana는 Prometheus에 저장된 Metric을 Dashboard 형태로 보여주는 역할을 합니다.

```text
Prometheus
     │
     │ PromQL
     ▼
Grafana
```

## 10.1 Repository 등록

```bash
sudo tee /etc/yum.repos.d/grafana.repo > /dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```

설치:

```bash
sudo dnf makecache
sudo dnf install grafana
```

---

# 11. Grafana 설정

```bash
sudo vi /etc/grafana/grafana.ini
```

현재 `/arm/` 경로를 사용하는 경우:

```ini
[server]

http_addr = 127.0.0.1
http_port = 8080

domain = 127.0.0.1
root_url = http://127.0.0.1:8080/arm/
serve_from_sub_path = true
```

Nginx Reverse Proxy를 사용하는 경우 `root_url`은 실제 외부 접속 URL에 맞춰야 합니다.

예:

```text
https://example.com/arm/
```

이라면:

```ini
[server]
http_addr = 127.0.0.1
http_port = 8080
root_url = https://example.com/arm/
serve_from_sub_path = true
```

Grafana 실행:

```bash
sudo systemctl enable --now grafana-server
```

상태 확인:

```bash
sudo systemctl status grafana-server
```

포트 확인:

```bash
ss -lntp | grep 8080
```

---

# 12. Grafana 접속

현재 설정 기준:

```text
http://127.0.0.1:8080/arm/
```

Grafana가 `127.0.0.1`에만 Listen하면 다른 PC에서 직접 접속할 수 없습니다.

외부에서 접속한다면 일반적으로 다음과 같이 구성합니다.

```text
사용자 PC
    │
    ▼
Nginx :443
    │
    ▼
Grafana :8080
```

---

# 13. Grafana와 Prometheus 연결

Grafana Web UI:

```text
Connections
  → Data sources
  → Add data source
  → Prometheus
```

Prometheus URL:

```text
http://localhost:9091
```

> 현재 Prometheus는 9091 포트를 사용하므로 9090이 아니라 9091을 입력합니다.

`Save & test` 실행:

```text
Data source is working
```

가 나오면 연결 완료입니다.

---

# 14. Grafana Node Exporter Dashboard

Grafana:

```text
Dashboards
  → New
  → Import
```

Grafana Dashboard ID 또는 URL을 입력합니다.

Dashboard를 Import할 때 Data Source는 방금 등록한:

```text
Prometheus
```

를 선택합니다.

다음과 같은 VM 정보를 Dashboard에서 확인할 수 있습니다.

```text
CPU
Memory
Disk
Network
Load Average
Filesystem
Uptime
```

---

# 15. Prometheus Alertmanager
- [ALERT_MANAGER_README.md](ALERT_MANAGER_README.md) 참고

---

# 16. 최종 확인 순서

## 16.1 Node Exporter

대상 VM:

```bash
systemctl status node_exporter
```

```bash
curl http://localhost:9100/metrics
```

## 16.2 Prometheus → Node Exporter 통신

Prometheus 서버:

```bash
nc -vz 10.10.30.2 9100
```

```bash
curl http://10.10.30.2:9100/metrics
```

## 16.3 Prometheus Target

```text
Prometheus
 → Status
 → Targets
```

다음과 같이 확인합니다.

```text
be-arm   UP
```

## 16.4 Prometheus Rule

```bash
promtool check rules /etc/prometheus/rules/node.yml
```

## 16.5 Alertmanager

```bash
systemctl status alertmanager
```

```bash
ss -lntp | grep 9093
```

## 16.6 Prometheus → Alertmanager

Prometheus 설정:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"
```

## 16.7 Grafana

```text
Grafana
 → Connections
 → Data sources
 → Prometheus
```

URL:

```text
http://localhost:9091
```

`Save & test`

## 23.8 Dashboard

```text
Grafana
 → Dashboards
 → Import
```

Node Exporter Dashboard Import.

---

# 24. 전체 구성 요약

```text
[각 VM]
   │
   │ :9100
   ▼
Node Exporter
   │
   │ scrape
   ▼
Prometheus :9091
   │
   ├──────────────► Grafana :8080
   │                   │
   │                   └─ Dashboard
   │
   └──────────────► Alertmanager :9093
                         │
                         └─ Email / Slack 등
```

## 포트 정리

| 서비스 | 포트 | 통신 방향 |
|---|---:|---|
| Node Exporter | 9100 | Prometheus → VM |
| Prometheus | 9091 | 사용자/Grafana → Prometheus |
| Grafana | 8080 | 사용자/Nginx → Grafana |
| Alertmanager | 9093 | Prometheus → Alertmanager |

## 핵심

1. **Node Exporter**
  - 모니터링 대상 VM마다 설치
  - `9100`에서 Metric 제공

2. **Prometheus**
  - Node Exporter의 Metric을 주기적으로 수집
  - `9091`에 저장 및 조회
  - Alert Rule 평가

3. **Grafana**
  - Prometheus의 데이터를 가져와 Dashboard로 시각화
  - Prometheus URL은 현재 구성상 `http://localhost:9091`

4. **Alertmanager**
  - Prometheus에서 발생한 Alert를 수신
  - Email, Slack 등의 알림으로 전달
  - 기본 포트 `9093`

5. **방화벽**
  - 다른 망의 VM을 모니터링하는 경우
  - Prometheus 서버 → 대상 VM `TCP 9100` 통신 허용 필요

6. **Target 상태**
  - Prometheus `Status → Targets`에서 `UP`인지 확인
  - `UP`이면 Prometheus가 해당 Node Exporter에서 Metric을 정상적으로 수집하고 있는 상태


