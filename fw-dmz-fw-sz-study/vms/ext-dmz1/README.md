# EXT-DMZ
## 네트워크
- HOST-ONLY (10.10.20.2) \
  gateway 10.10.20.1
``` {shell} \
  # ipv4 할당 및 gateway 설정
  nmcli connection modify enp0s8 ipv4.addresses 10.10.20.2/24
  nmcli connection modify enp0s8 ipv4.gateway 10.10.20.1
  nmcli connection modify enp0s8 ipv4.method manual
  nmcli connection up enp0s8
  # ip route
  default via 10.10.20.1 dev enp0s8

```

## 호스트 설정

```shell
sudo hostnamectl set-hostname ext-dmz1
```
-------

## 목표
- dmz-lb 에서 트래픽을 전달받는 역할을한다.

  

