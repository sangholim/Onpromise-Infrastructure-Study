# oracle vm , multiple subnet, ipvs 를 이용한 망연계

# 구성도
![DMZ 서버](./images/Onpromise-Infrastructure.png "DMZ Server VM")

# 프로젝트 계기
- 방화벽 장비의 역할 및 실제 동작 방식 이해
- L7 기반 단순 reverse proxy 가 아닌 L3/L4 기반 네트워크 흐름 검증
  - [L3, L7 영역 비교](./contents/L3_L7_DIFF.md)
- VIP 및 L4 Load Balancer 구조 직접 구현
- DMZ / Server Zone 분리를 통한 내부망 보호 구조 검증
- Stateful Firewall 환경에서의 실제 세션 흐름 이해
- asymmetric routing 및 NAT 환경 문제 분석 경험 확보

# 구현 모듈
- 학습 노트북이 Apple Silicon CPU 를 쓰므로 많이 제한적임
- VM 구현 application: 
  - VM Fusion: PORT FORWARDING, host only 설정 이슈 존재
  - UTM: NAT , host only
  - Virtual BOX: oracle vm
- 방화벽 application: IPFire
- OS : rhel10 CE
- DMZ-LB: IPVS + keepavlied
- DMZ/SZ Web-Server: Nginx

# 네트워크 구간 구성
### WAN
외부 사용자 접근 영역.
### DMZ
외부 공개 서비스 위치.
VIP 및 Load Balancer 배치.
### Backend / Server Zone
실제 Application 및 Web Server 위치.
외부 직접 접근 차단.
### Host-only / Management
운영 및 관리 네트워크 영역.

# Load Balancing / High Availability 구성
### IPVS 기반 L4 Load Balancer 구성 (NAT MODE)
적용 이유
- Backend 설정 단순화
- NAT 기반 트래픽 흐름 제어 용이
- 실습 환경에서 Routing 구조 단순화
검증 항목
- VIP 기반 서비스 엔드포인트
- Backend 트래픽 분산
- NAT 기반 Return Path 동작
  ```text
  # 예제1
  Client:     1.1.1.1
  VIP:        10.10.10.221
  DMZ LB(IPVS):    10.10.20.200
  Backend:    10.10.20.2
  # 요청 흐름
  1.1.1.1 -> 10.10.10.221 (ip 변환: 10.10.20.200) -> 10.10.20.2
  # Backend 수신 패킷 정보
  Backend 수신 packet:
  SRC=10.10.20.200
  DST=10.10.20.2 
  # Backend 수신 packet 이 응답시 
  10.10.20.2 -> 10.10.20.200 -> 1.1.1.1
  ```
- Session 흐름 분석

### 참고
- [라우트 컨셉](./contents/ROUTE_CONCEPT.md)
- [dmz-lb 구축](./vms/ext-dmz-lb/README.md)

### Keepalived 기반 VIP 이중화
구성 방식
- Active / Backup 구조
- VRRP 기반 VIP Failover
- Health Check 기반 자동 절체
검증 항목
- VIP failover 동작
- Gratuitous ARP 기반 VIP 이동
- 장애 발생 시 세션 영향 확인

# 방화벽 및 네트워크 보안 구성
### 보안 정책 설계
- 외부 → DMZ 접근 허용
- DMZ → Backend 최소 포트 허용
- Backend 직접 외부 노출 차단
- NAT 기반 내부망 보호
- Stateful Firewall 기반 세션 관리 적용
### 참고
- [dmz-fw 구축](./vms/ext-fw1/README.md)

# 네트워크 문제 분석 및 트러블슈팅
### asymmetric routing 문제 분석 및 해결
- [dmz-lb 연결 이슈 대응 구축](./vms/ext-dmz-lb/TROUBLE_SHOOTING.md)