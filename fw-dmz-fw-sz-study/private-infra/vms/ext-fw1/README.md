# EXT-FW1
## 네트워크
- RED \
  NAT (10.10.1.3)
- GREEN \
  FW 10.10.10.2
- ORANGE \
  DMZ HOST-ONLY (10.10.20.2)\

## IP FIRE 정책
| Zone   | 역할      |
| ------ | ------- |
| RED    | 외부 네트워크 |
| ORANGE | DMZ     |
| GREEN  | 내부망     |

# 포트포워딩, IP TABLES 설정
- vm 네트워크는 RED + GREEN 조합
- RED = NAT
- GREEN = HOST ONLY
## 관리자 웹 접근
- 관리자 GUI 포트는 444 이며 \
  GREEN 에서만 접근할수 있도록 방화벽이 OPEN 되어 있음
- Virtual box 에서 포트포워딩 설정 필요 \
  ![포트포워딩](img.png)
- ``` {shell} \
  # 처음에는 cli 로 방화벽 설정후, GUI 에서 등록
  # 프로그램 포트 확인
  netstat -tnlp | grep 444
  # iptables 방화벽 추가
  sudo iptables -I INPUT -i red0 -p tcp --dport 444 -j ACCEPT
  # 출력값 확인
  iptables -L -v -n | grep 444
  # 웹 콘솔 붙이고 나서 나중에 변경 필요
  ```
  ![방화벽](img_1.png)
  
