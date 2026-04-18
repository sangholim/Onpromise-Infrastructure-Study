x`# Onpromise-Infrastructure-Study
온프로미스 인프라 구축 학습

# 구성도
![DMZ 서버](./images/Onpromise-Infrastructure.png "DMZ Server VM")

# 구현 모듈
- 학습 노트북이 Apple Silicon CPU 를 쓰므로 많이 제한적임
- VM 구현 application: 
  - VM Fusion: PORT FORWARDING, host only 설정 이슈 존재
  - UTM: NAT , host only
  - Virtual BOX: ?
- 방화벽 application: IPFire
- OS : rhel9 CE
- DMZ/SZ Web-Server: Nginx
## VM 네트워크 정보
- NAT DHCP
- VIP 10.10.10.250/24
- EXT-FW 10.10.10.0/24
- DMZ 10.10.20.0/24
