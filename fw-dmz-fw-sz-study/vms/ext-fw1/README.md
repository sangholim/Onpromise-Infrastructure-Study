# EXT-FW1
## 네트워크
- RED \
  NAT (10.10.1.3)
  
- GREEN \
  FW 10.10.10.2
 
- ORANGE \
  DMZ HOST-ONLY (10.10.20.2)\
  dmz 외부로 나가는용\
## 포트
- 관리자웹
  - 8444 -> fw1-host:444
- fw ssh 
  - 2222 -> fw1-host:22
- dmz bastion ssh
  - 3333 -> dmz-bastion-host:22
## IP FIRE 정책
| Zone   | 역할      |
| ------ | ------- |
| RED    | 외부 네트워크 |
| ORANGE | DMZ     |
| GREEN  | 내부망     |
