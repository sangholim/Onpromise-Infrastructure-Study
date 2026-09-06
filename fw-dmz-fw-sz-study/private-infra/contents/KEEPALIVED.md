# Keepalived - VRRP 기반 Active/Backup VIP Failover 구조

---

## 1. Keepalived 개념

Keepalived는 **VRRP(Virtual Router Redundancy Protocol)** 기반으로 동작하며,  
서로 다른 서버 중 하나를 MASTER로 선출하여 **VIP(Virtual IP)의 소유권을 관리하는 HA(High Availability) 솔루션**이다.

---

## 2. 하는 일

| 역할 | 설명 |
|------|------|
| MASTER | VIP 소유 (Active 역할) |
| BACKUP | MASTER 상태 감시 |
| VRRP | MASTER 생존 여부 확인 (heartbeat) |
| VIP | 클라이언트가 접근하는 단일 진입 IP |
| Failover | MASTER 장애 시 VIP를 다른 노드로 이동 |

---

## 3. Keepalived vs IPVS

| 기능 | Keepalived | IPVS |
|------|------------|------|
| VIP 관리 | O | X |
| MASTER/BACKUP 선출 | O | X |
| 장애 조치 (Failover) | O | X |
| 로드밸런싱 | X | O |
| Backend 분배 | X | O |
| 역할 | HA (고가용성) | L4 Load Balancer |

---

## 4. VRRP 핵심 개념

Keepalived는 VRRP 프로토콜을 통해 MASTER를 선출한다.

### 핵심 요소
- Priority 기반 선출 (값이 높을수록 MASTER)
- Advertisement (heartbeat)
- Interface 기반 L2 통신
- Multicast 또는 Unicast VRRP

---

## 5. 기본 구성

### 초기 상태

| 구분 | dmz-lb | dmz-lb1 |
|------|--------|---------|
| 상태 | MASTER | BACKUP |
| Priority | 100 | 90 |
| VIP (10.10.10.221) | 보유 | 없음 |
| Client 요청 수신 | O | X |
| VRRP 광고 송신 | O | X |
| VRRP 광고 수신 | X | O |

---

## 6. MASTER 장애 발생 시 동작

### dmz-lb1 입장 동작

| 시간 | 동작 |
|------|------|
| 0초 | VRRP advertisement 수신 중 |
| 1초 | 광고 미수신 |
| 2초 | 광고 미수신 |
| 3초 | advertisement timeout 발생 |
| 이후 | MASTER 승격 |

> 일반적으로 advertisement interval × 3 이상 미수신 시 failover 발생

---

## 7. Failover 이후 상태

| 구분 | dmz-lb | dmz-lb1 |
|------|--------|---------|
| 상태 | DOWN | MASTER |
| VIP (10.10.10.221) | 없음 | 보유 |
| Client 요청 수신 | X | O |
| 서비스 상태 | 불가 | 정상 |

---

## 8. VIP 이동 과정

| 단계 | 설명 |
|------|------|
| 1 | MASTER 장애 발생 |
| 2 | BACKUP에서 VRRP timeout 감지 |
| 3 | BACKUP → MASTER 승격 |
| 4 | VIP 인터페이스에 추가 |
| 5 | Gratuitous ARP 전송 (MAC 갱신) |
| 6 | 클라이언트 트래픽 정상 수신 |

---

## 9. Preempt vs Nopreempt

### Preempt (기본 동작)

| 상황 | MASTER |
|------|--------|
| 장애 전 | dmz-lb |
| 장애 발생 | dmz-lb1 |
| dmz-lb 복구 | dmz-lb (복귀) |

- priority 높은 노드가 항상 MASTER 복귀

---

### Nopreempt

| 상황 | MASTER |
|------|--------|
| 장애 전 | dmz-lb |
| 장애 발생 | dmz-lb1 |
| dmz-lb 복구 | dmz-lb1 유지 |

- 한번 MASTER가 변경되면 원복되지 않음

---

## 10. Split Brain (중요 개념)

### 정의
VRRP 통신 단절로 인해 **두 노드가 동시에 MASTER가 되는 상태**

### 원인
- VRRP multicast/unicast 차단
- firewall blocking (protocol 112)
- interface mismatch
- network segmentation

### 결과
- VIP 충돌
- ARP flapping
- TCP 세션 불안정

---

## 11. Split Brain 방지 방법

- VRRP unicast 설정 (VirtualBox 환경 권장)
- firewall VRRP (protocol 112) 허용
- interface 일치
- priority 차이 설정
- authentication 설정

---

## 12. IPVS와의 관계

Keepalived는 IPVS를 직접 제어할 수도 있고,  
VRRP 기반 VIP failover만 수행할 수도 있다.

---

## 13. 전체 구조

```text
Client
  ↓
FW (IPFire)
  ↓
VIP (Keepalived)
  ↓
DMZ-LB (IPVS)
  ↓
DMZ