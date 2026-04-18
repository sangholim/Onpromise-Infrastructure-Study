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
- 모듈: HAPROXY \
```
장점: 트래픽 분산이 정교하다.
     장애 감지/FailOver 가 뛰어나다.
     커넥션 처리 효율이 뛰어나다,.
     정적리소스 캐싱 처리는 nginx 가 뛰어나다. (sz web server 는 nginx)
     다만, 정적리소스 캐싱인 경우 nginx 도 권한다.     
```
  - 


  

