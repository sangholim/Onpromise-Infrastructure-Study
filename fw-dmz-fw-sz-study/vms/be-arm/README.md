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

## 목표
- dmz-lb 에서 트래픽을 전달받는 역할을한다.
- nginx 로 프록시를 수행한다.


## Prometheus 설치
- [참고](../software/prometheus/INSTALL.md)

Prometheus 서버에 Prometheus를 설치합니다.

### 1 사용자 생성

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

### 2 디렉터리 생성

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

