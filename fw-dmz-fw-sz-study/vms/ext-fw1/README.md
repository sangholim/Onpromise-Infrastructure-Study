# EXT-FW1
## 네트워크
- NAT (10.10.1.2)\
  RED
- FW HOST-ONLY (10.10.10.2)\
  GREEN
- DMZ HOST-ONLY (10.10.20.1)\
  dmz 외부로 나가는용\
  ORANGE
## 포트
- 관리자웹
  - 8444 -> fw1-host:444
- fw ssh 
  - 2222 -> fw1-host:22
- dmz bastion ssh
  - 3333 -> dmz-bastion-host:22