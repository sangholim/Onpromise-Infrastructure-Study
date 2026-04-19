# EXT-DMZ-LB
## 네트워크
- HOST-ONLY (10.10.20.200) \
  gateway 10.10.20.1
``` {shell} \
  # ipv4 할당 및 gateway 설정
  nmcli connection modify enp0s8 ipv4.addresses 10.10.20.200/24
  nmcli connection modify enp0s8 ipv4.gateway 10.10.20.1
  nmcli connection modify enp0s8 ipv4.method manual
  nmcli connection up enp0s8
  # ip route
  default via 10.10.20.1 dev enp0s8

```

## 호스트 설정

```shell
sudo hostnamectl set-hostname ext-dmz-lb
```
-------

## ssh 설정 참고
- [ssh 설정가이드](./ssh/README.md)

## 목표
- fw 에서 proxy 로 오는 ip 를 받는 역할, proxy vm 들의 lb 를 담당
- IPVS . 구현

