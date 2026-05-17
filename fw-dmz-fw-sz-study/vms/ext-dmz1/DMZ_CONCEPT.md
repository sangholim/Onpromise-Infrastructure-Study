# DMZ / LB 구조 설계 문서

## 1. 아키텍처 개요
```text
[ Client ]  
↓  
[ L7 Load Balancer (NGINX) ]  
↓  
[ DMZ - Proxy VM (NGINX) ]  
↓  
[ Internal Services ]

```

### 구성 역할

| 구성 요소 | 역할 |
|----------|------|
| L7 LB | 외부 트래픽 분산, SSL 종료, 라우팅 |
| DMZ Proxy | 내부망 보호, 추가 보안 필터링 |
| Backend | 실제 서비스 처리 |

---

## 2. LB 구간 설명 (External → DMZ)

### 주요 역할
- SSL Termination
- 도메인 기반 라우팅
- 트래픽 분산 (Load Balancing)
- 기본 보안 필터링 (WAF 연동 가능)

### 특징
- 외부 트래픽 최초 진입 지점
- 고가용성(HA) 필수 구성
- Stateless 구조 권장

### 한계
- 내부 네트워크 보호 기능 없음
- Outbound 트래픽 제어 불가

---

## 3. DMZ 구간 설명 (Proxy VM)

### 핵심 역할
- 내부망 격리 (Network Isolation)
- 2차 보안 필터링
- 내부 서비스 접근 제어

### 트래픽 흐름
Inbound  : LB → DMZ → Internal (LB 기능 일부 수행 가능)  
Outbound : Internal → DMZ → External (단순 Proxy / NAT 수준)

### 핵심 포인트
- DMZ Proxy는 Inbound에서 LB 역할 수행 가능
- Outbound 방향에서는 LB 역할 불가

---

## 4. DMZ에서 LB 역할 가능 여부
| 구분 | 가능 여부 | 설명 |
|------|----------|------|
| Inbound LB | 가능 | 내부 서버로 트래픽 분산 |
| Outbound LB | 불가능 | 클라이언트 성격이라 분산 의미 없음 |

---

## 5. DMZ Proxy 설계 옵션
### 5.1 Reverse Proxy (NGINX)
#### 장점
- 설정 단순
- 성능 우수
- HTTP/HTTPS 최적화

#### 단점
- TCP/UDP 제한
- 복잡한 정책 구현 어려움

---

### 5.2 Forward Proxy
#### 장점
- 외부 접근 제어 가능
- 로그 및 감사 용이
- ACL 정책 적용 가능
#### 단점
- 클라이언트 설정 필요
- 사용자 환경 의존성 존재

---
### 5.3 Transparent Proxy
#### 장점
- 클라이언트 설정 불필요
- 네트워크 레벨 강제 적용 가능
#### 단점
- 디버깅 어려움
- HTTPS 처리 복잡 (MITM 필요)

---

### 5.4 WAF 연동
#### 장점
- SQL Injection, XSS 등 공격 차단
- 보안 강화
#### 단점
- 성능 저하 가능
- 오탐 튜닝 필요

---

### 5.5 Dual LB 구조
Client → LB → DMZ Proxy → Backend

#### 장점
- 보안 계층 분리 (Defense in Depth)
- 장애 격리 가능
- 정책 분리

#### 단점
- 구조 복잡
- 지연 증가
- 비용 증가

---

## 6. 핵심 정리

- LB: 외부 트래픽 처리 담당
- DMZ: 보안 경계 및 내부망 보호
- Proxy VM: Inbound에서 LB 역할 일부 수행 가능
- Outbound LB 기능은 없음

---

## 7. 결론

DMZ는 단순 Proxy가 아니라 **보안 경계 계층**이다.

LB와 역할이 일부 겹치지만 목적은 다르다.

- LB = 트래픽 분산
- DMZ = 보안 및 네트워크 격리

👉 "DMZ가 LB 역할을 한다"는 표현은  
**Inbound 상황에서만 부분적으로 성립한다.**