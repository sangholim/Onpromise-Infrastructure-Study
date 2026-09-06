# Prometheus Alertmanager

Alertmanager는 Prometheus가 발생시킨 Alert를 받아서 관리하고 알림을 전달합니다.

구조:

```text
Node Exporter
      │
      ▼
 Prometheus
      │
      │ Alert
      ▼
Alertmanager
      │
      ├── Email
      ├── Slack
      └── 기타 알림
```

---

# Alertmanager 설치

```bash
cd /tmp

wget https://github.com/prometheus/alertmanager/releases/download/v0.34.0/alertmanager-0.34.0.linux-arm64.tar.gz

tar -xzf alertmanager-0.34.0.linux-arm64.tar.gz

cd alertmanager-0.34.0.linux-arm64
```

파일 확인:

```bash
ls
```

```text
alertmanager
amtool
alertmanager.yml
LICENSE
NOTICE
```

바이너리 설치:

```bash
sudo cp alertmanager /usr/local/bin/
sudo cp amtool /usr/local/bin/

sudo chmod +x /usr/local/bin/alertmanager
sudo chmod +x /usr/local/bin/amtool
```

확인:

```bash
alertmanager --version
amtool --version
```

---

# Alertmanager 사용자 및 디렉터리

사용자 생성:

```bash
sudo useradd --no-create-home \
  --shell /sbin/nologin \
  alertmanager
```

디렉터리:

```bash
sudo mkdir -p /etc/alertmanager
sudo mkdir -p /var/lib/alertmanager
```

권한:

```bash
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown -R alertmanager:alertmanager /var/lib/alertmanager
```

---

# Alertmanager 설정

```bash
sudo vi /etc/alertmanager/alertmanager.yml
```

기본 구조:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: "default"

receivers:
  - name: "default"
```

실제 Email, Slack 등의 알림을 사용할 경우 `receivers`에 해당 설정을 추가합니다.

설정 검사:

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
```

---

# Alertmanager systemd 등록

```bash
sudo vi /etc/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple

ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

등록 및 실행:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
```

상태:

```bash
sudo systemctl status alertmanager
```

포트 확인:

```bash
ss -lntp | grep 9093
```

Alertmanager Web UI:

```text
http://<ALERTMANAGER_IP>:9093
```

---

# Prometheus와 Alertmanager 연결

Prometheus 설정:

```bash
sudo vi /etc/prometheus/prometheus.yml
```

추가:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"
```

Rule 파일을 사용할 경우 다음 설정도 추가합니다.

```yaml
rule_files:
  - "/etc/prometheus/rules/*.yml"
```

전체적인 구조:

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"

rule_files:
  - "/etc/prometheus/rules/*.yml"

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

---

# Alert Rule 작성

디렉터리 생성:

```bash
sudo mkdir -p /etc/prometheus/rules
```

Rule 파일:

```bash
sudo vi /etc/prometheus/rules/node.yml
```

Node Exporter가 Down 되었을 때 Alert를 발생시키는 예제:

```yaml
groups:
  - name: node
    rules:

      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node is down"
          description: "{{ $labels.server }} is down."
```

설정 검사:

```bash
promtool check rules /etc/prometheus/rules/node.yml
```

Prometheus 설정 검사:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

재시작:

```bash
sudo systemctl restart prometheus
```

---

# Alert 동작 구조

예를 들어 `be-arm` VM이 다운되면:

```text
BE-ARM
10.10.30.2
    │
    │ Node Exporter 응답 없음
    ▼
Prometheus
    │
    │ up{job="node"} == 0
    │ 1분 지속
    ▼
NodeDown Alert
    │
    ▼
Alertmanager
    │
    ├── Email
    ├── Slack
    └── 기타 알림
```

Prometheus Web UI에서:

```text
Alerts
```

를 통해 Alert 상태를 확인할 수 있습니다.

Alertmanager Web UI:

```text
http://<ALERTMANAGER_IP>:9093
```

에서 Prometheus가 전달한 Alert를 확인할 수 있습니다.

# Slack 으로 Alert 알림 받기 (1) - slack webhook url 생성
- slack 채널 생성 (이름: monitoring)
- 앱생성: blank app
- name, workspace 설정
- webhook 활성화
- url 생성
- 참고파일: [slack](config/slack)

# Slack 으로 Alert 알림 받기 (2) - prometheus alert manager 연동
- [alertmanager.yml](config/alertmanager.yml) 에 slack 채널명, webhook 정보 포함해서 설정

