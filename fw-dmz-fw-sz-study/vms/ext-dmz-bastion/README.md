# EXT-DMZ-BASTION
- 역할: 내부 DMZ 밑 다른 서버존 접근을 경유해주는 서버
## 네트워크
- HOST-ONLY (10.10.20.251) \
  gateway 10.10.20.2
``` {shell} \
  # ipv4 할당 및 gateway 설정
  nmcli connection modify enp0s8 ipv4.addresses 10.10.20.251/24
  nmcli connection modify enp0s8 ipv4.gateway 10.10.20.2
  nmcli connection modify enp0s8 ipv4.method manual
  nmcli connection up enp0s8
  # ip route
  default via 10.10.20.2 dev enp0s8
```
## 호스트 설정
```shell
sudo hostnamectl set-hostname ext-dmz-bastion
```

## ssh 설정 참고
- [ssh 설정가이드](./ssh/README.md)


## 포트 방화벽 허용

```bash
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
# 방화벽 개방 (node_exporter)
```

## Node exporter 설치
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


