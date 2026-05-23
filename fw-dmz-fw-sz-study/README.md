# oracle vm , multiple subnet, ipvs 를 이용한 망연계

# 구성도
![DMZ 서버](./images/Onpromise-Infrastructure.png "DMZ Server VM")

# 프로젝트 계기
- 방화벽 장비의 역할 및 실제 동작 방식 이해
- L7 기반 단순 reverse proxy 가 아닌 L3/L4 기반 네트워크 흐름 검증
- VIP 및 L4 Load Balancer 구조 직접 구현
- DMZ / Server Zone 분리를 통한 내부망 보호 구조 검증
- Stateful Firewall 환경에서의 실제 세션 흐름 이해
- asymmetric routing 및 NAT 환경 문제 분석 경험 확보

# 네트워크 영역 분리
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

# 설계 목적
- 외부 공개 구간과 내부 서비스 구간 분리
- 내부 서버 직접 노출 방지
- Firewall 기반 접근 제어 강화
- 네트워크 역할별 트래픽 분리
- 운영 및 장애 분석 용이성 확보

# L3, L7 라우팅 성능 차이
| 항목         | L3 라우팅                | L7 라우팅                                                 |
| ---------- |-----------------------|--------------------------------------------------------|
| 처리 계층      | IP 계층                 | 애플리케이션 계층                                              |
| 확인 정보      | Source/Destination IP | URL, Header, Cookie, Host 등                            |
| 처리 방식      | 패킷 포워딩(IP 헤더만 앍음) | 프록시 + 패킷 해석 (패킷 본문 읽음)                                 |
| 속도         | 매우 빠름                 | 상대적으로 느림                                               |
| 지연         | μs(마이크로초) 수준          | ms(밀리초) 수준                                             |
| CPU 사용량    | 낮음                    | 높음                                                     |
| TLS 종료     | 보통 안함                 | 대부분 수행 (외부 TLS 종료후, nginx - 내부 server 간 프록시 연결 가능성 존재) |
| Throughput | 매우 높음                 | CPU 의존                                                 |
| 대표 장비      | Router, L3 Switch     | Nginx, Envoy, HAProxy                                  |


# Load Balancing / High Availability 구성
## IPVS 기반 L4 Load Balancer 구성
### Linux IPVS 기반 NAT Mode L4 Load Balancer 구성.
적용 이유
- Backend 설정 단순화
- NAT 기반 트래픽 흐름 제어 용이
- 실습 환경에서 Routing 구조 단순화
검증 항목
- VIP 기반 서비스 엔드포인트
- Backend 트래픽 분산
- NAT 기반 Return Path 동작
  - 응답시 NAT 장비를 다시 지나야한다.
    ```text
    # 예제1
    Client:     1.1.1.1
    VIP:        10.10.20.221
    Backend:    10.10.20.2
    IPVS LB:    10.10.20.251
    # 요청 흐름
    DST = 10.10.20.221
    # IPVS 가 NAT 수행 (DNAT)
    DST 변경:
    10.10.20.221 → 10.10.20.2     
    # Backend 응답 (SNAT)
    VIP 통신하고 있기 때문에, 출발지 10.10.20.221 이 되어야한다.
    SRC = 10.10.20.2 -> 10.10.20.221 로 바꿈
    # Client 는 backend 존재를 모름, VIP 만 봄 
    ```
- Session 흐름 분석

## Keepalived 기반 VIP 이중화
구성 방식
- Active / Backup 구조
- VRRP 기반 VIP Failover
- Health Check 기반 자동 절체
검증 항목
- VIP failover 동작
- Gratuitous ARP 기반 VIP 이동
- 장애 발생 시 세션 영향 확인
# 방화벽 및 네트워크 보안 구성
## IPFire 기반 Zone 분리
| Zone   | 역할      |
| ------ | ------- |
| RED    | 외부 네트워크 |
| ORANGE | DMZ     |
| GREEN  | 내부망     |

## 보안 정책 설계
- 외부 → DMZ 접근 허용
- DMZ → Backend 최소 포트 허용
- Backend 직접 외부 노출 차단
- NAT 기반 내부망 보호
- Stateful Firewall 기반 세션 관리 적용

# 네트워크 문제 분석 및 트러블슈팅
## Stateful Connection Tracking 분석
conntrack 기반 TCP 세션 상태 분석 수행.

분석 항목
- NEW
- ESTABLISHED
- INVALID
- TIME_WAIT
확인 내용
- NAT 환경 세션 유지 방식
- Firewall state tracking 동작
- 세션 비정상 종료 상황

## asymmetric routing 문제 분석 및 해결
### 발생 원인
요청 패킷과 응답 패킷이 서로 다른 경로로 전달되면서 Stateful Firewall state mismatch 발생.

증상
- SYN/ACK 유실
- TCP 연결 불안정
- INVALID state 발생
분석 방법
```shell
tcpdump -nn
conntrack -L
ip route
arp -a
```

해결 방법
- Routing 정책 수정
- NAT 구조 정리
- Return Path 일관성 확보