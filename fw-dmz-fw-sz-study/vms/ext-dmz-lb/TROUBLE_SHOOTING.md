# DMZ-LB 장애 대응
## 비대칭 라우팅
### 발생 원인
DMZ-LB (VIP) 에서 BACKEND 로 요청시 정상 응답을 받지만 \
다른 DMZ VM 또는 FW 구간에서 응답을 받지 못함.
### 확인 결과
- ```text
  DMZ-LB 는 VIP 기반으로 BACKEND 로 요청시 NAT 형태로 라우팅 되어 있음.
  즉 소스 IP 도 변환된 상태 (SNAT) 인대, 응답시 나가는 IP 도 들어온 IP 와 같아야 하는데, 
  깉은존에 있는 목적지 읺은 경우, 소스 IP 를 경유하지 않거나, 변환된 소스 IP 로 경유하지 않는 현상 발견  
  ```
 ### 분석 명령어
```shell
# Client -> LB -> Backend -> LB -> Client 가야하는지 체크
tcpdump -nn
# NAT/state 세션이 정상 유지되는가
conntrack -L
# 커널 경로 확인
ip route
# L2 에서 직접 통신하고 있는지 확인
arp -a
```

- 분석결과
```text
VIP - DMZ 구간이 동일한 네트워크 인터페이스 환경이 문제인걸로 확인
(응답시 VIP 를 거치지 않고 직접 연결이 가능하기 때문에, 비대칭 라우팅 가능성이 있음)
```
### 분석후 대응
- DMZ-LB VIP 를 FW 인터페이스로 지정
- **BACKEND GW 를 DMZ-LB 네트워크 인터페이스로 설정**