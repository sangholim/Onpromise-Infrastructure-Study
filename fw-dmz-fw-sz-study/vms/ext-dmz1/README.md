# EXT-DMZ
## 네트워크
- HOST-ONLY (10.10.20.231) \
  gateway 10.10.20.221 (dmz -> dmz-lb)
``` {shell} \
  # ipv4 할당 및 gateway 설정
  nmcli connection modify enp0s8 ipv4.addresses 10.10.20.231/24
  nmcli connection modify enp0s8 ipv4.gateway 10.10.20.221
  nmcli connection modify enp0s8 ipv4.method manual
  nmcli connection up enp0s8
  # ip route
  default via 10.10.20.221

```

## 호스트 설정

```shell
sudo hostnamectl set-hostname ext-dmz1
```
-------

## 목표
- dmz-lb 에서 트래픽을 전달받는 역할을한다.
- nginx 로 프록시를 수행한다.


## 설치
```shell
# nginx 설치
dnf install -y epel-release
dnf install -y nginx

# 서비스 시작 + 부팅 자동 실행
systemctl enable nginx --now
systemctl status nginx

# 방화벽 허용 (필요 시)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# 동작확인
curl http://localhost

```
## http 포트 개방

```bash
getenforce
# output 이 enforcing 이면 아래 수행
getsebool httpd_can_network_connect
# httpd_can_network_connect --> off 이면 아래 수행
sudo setsebool -P httpd_can_network_connect 1
# 네트워크 연결 다시 확인
getsebool httpd_can_network_connect
# httpd_can_network_connect --> on 이면 nginx 재기동
```

## 포트 방화벽 허용

```bash
sudo firewall-cmd --add-port=9100/tcp
# 방화벽 개방 (node_exporter)
```