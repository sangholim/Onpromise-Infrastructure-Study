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

## Prometheus 설치
Prometheus 서버에 Prometheus를 설치합니다.
- [참고](../software/prometheus/INSTALL.md)
- Metrics 수집 및 저장 port: 9091
- 



### 1 사용자 생성

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

### 2 디렉터리 생성

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```


### 2.3 Prometheus 설치

Prometheus 바이너리를 다운로드하여 설치합니다.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.13.1/prometheus-3.13.1.linux-arm64.tar.gz
tar xvf prometheus-3.13.1.linux-arm64.tar.gz
cd prometheus-3.13.1.linux-arm64
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
          - "localhost:9091"
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
## 5. Grafana 설치

Prometheus Metrics를 Dashboard로 시각화하기 위해 Grafana를 설치합니다.

```text
Prometheus
     │
     │ PromQL
     ▼
Grafana
```

Grafana 설치

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

```bash
sudo dnf makecache
sudo dnf install grafana
```

그라파나 설정 편집 (1)
```bash
sudo vi /etc/grafana/grafana.ini
```

그라파나 설정 편집 (2)
```bash
[server]

domain = 127.0.0.1
root_url = https://127.0.0.1/arm/
serve_from_sub_path = true
```


Grafana 설치 후 서비스를 실행합니다.

```bash
sudo systemctl enable --now grafana-server
```

상태 확인:

```bash
sudo systemctl status grafana-server
```

접속:

```text
http://127.0.0.1:8080/arm

id: admin
pw: admin
```

Node exporter 설치 (다른망에 있는 vm 연결이 필요한 경우, 방화벽에서 연결설정 필요함)

```bash
cd /tmp

wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-arm64.tar.gz
tar -xzf node_exporter-1.12.1.linux-arm64.tar.gz
sudo cp node_exporter-1.12.1.linux-arm64/node_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/node_exporter
```

계정 생성

```bash
sudo useradd --system \
--no-create-home \
--shell /sbin/nologin \
node_exporter
```

서비스

```bash
sudo vi /etc/systemd/system/node_exporter.service

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

실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

systemctl status node_exporter

curl http://localhost:9100/metrics
```


Prometheus 에 등록

```bash
sudo vi /etc/prometheus/prometheus.yml
#  아래 형태로 등록
# - job_name: 'node'
#   static_configs:
#     - targets:
#          - '10.10.30.2:9100'
#
          
scrape_configs:
  - job_name: 'node'
    # vm 용도별로 변수 수정 필요
    static_configs:
      - targets:
          - '10.10.30.2:9100'
        labels:
          server: 'be-arm' 
          env: 'prod'

# prometheus 재기동
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

Grafana 웹 화면에서 프로메테우스 등록
1. Grafana → Connections → Data sources
2. 프로메테우스 선택
3. http://localhost:9090
4. save & test

Grafana 웹 화면에서 대시보드에 node_exporter metric 등록
1. Grafana → Dashboards 선택
2. 우측 new 버튼 -> import dashboard 선택
3. 원하는 정보 템플릿의 url 또는 grafana dashboard id 입력후 load
4. Grafana → Dashboards 선택후, 등록한 목록 선택
